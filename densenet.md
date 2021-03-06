```{.python .input  n=1}
# 如果训练下载的Kaggle的完整数据集，把下面改False
demo = False
if demo:
    import zipfile
    for fin in ['train_tiny.zip', 'test_tiny.zip', 'trainLabels.csv.zip']:
        with zipfile.ZipFile('../data/kaggle_cifar10/' + fin, 'r') as zin:
            zin.extractall('../data/kaggle_cifar10/')
```

```{.python .input  n=2}
import os
import shutil

def reorg_cifar10_data(data_dir, label_file, train_dir, test_dir, input_dir, valid_ratio):
    # 读取训练数据标签。
    with open(os.path.join(data_dir, label_file), 'r') as f:
        # 跳过文件头行（栏名称）。
        lines = f.readlines()[1:]
        tokens = [l.rstrip().split(',') for l in lines]
        idx_label = dict(((int(idx), label) for idx, label in tokens))
    labels = set(idx_label.values())

    def mkdir_if_not_exist(path):
        if not os.path.exists(os.path.join(*path)):
            os.makedirs(os.path.join(*path))

    # 整理训练和验证集。
    for train_file in os.listdir(os.path.join(data_dir, train_dir)):
        idx = int(train_file.split('.')[0])
        label = idx_label[idx]
        mkdir_if_not_exist([data_dir, input_dir, 'train_valid', label])
        shutil.copy(os.path.join(data_dir, train_dir, train_file),
                    os.path.join(data_dir, input_dir, 'train_valid', label))

    # 整理测试集。
    mkdir_if_not_exist([data_dir, input_dir, 'test', 'unknown'])
    for test_file in os.listdir(os.path.join(data_dir, test_dir)):
        shutil.copy(os.path.join(data_dir, test_dir, test_file),
                    os.path.join(data_dir, input_dir, 'test', 'unknown'))
```

```{.python .input  n=3}
if demo:
    # 注意：此处使用小训练集为便于网页编译。Kaggle的完整数据集应包括5万训练样本。
    train_dir = 'train_tiny'
    # 注意：此处使用小测试集为便于网页编译。Kaggle的完整数据集应包括30万测试样本。
    test_dir = 'test_tiny'
    # 注意：此处相应使用小批量。对Kaggle的完整数据集可设较大的整数，例如128。
    batch_size = 1
else:
    train_dir = 'train'
    test_dir = 'test'
    batch_size = 128

data_dir = '../data/kaggle_cifar10'
label_file = 'trainLabels.csv'
input_dir = 'train_valid_test'
valid_ratio = 0.1
reorg_cifar10_data(data_dir, label_file, train_dir, test_dir, input_dir, valid_ratio)
```

```{.python .input  n=4}
from mxnet import autograd
from mxnet import gluon
from mxnet import image
from mxnet import init
from mxnet import nd
from mxnet.gluon.data import vision
import numpy as np

def transform_train(data, label):
    im = data.asnumpy()
    im = np.pad(im, ((4, 4), (4, 4), (0, 0)), mode='constant', constant_values=0)
    im = nd.array(im, dtype='float32') / 255
    auglist = image.CreateAugmenter(data_shape=(3, 32, 32), resize=0, 
                        rand_crop=True, rand_mirror=True,
                        mean=np.array([0.4914, 0.4822, 0.4465]), 
                        std=np.array([0.2023, 0.1994, 0.2010]))
    for aug in auglist:
        im = aug(im)
    # 将数据格式从"高*宽*通道"改为"通道*高*宽"。
    im = nd.transpose(im, (2,0,1))
    return (im, nd.array([label]).asscalar().astype('float32'))

# 测试时，无需对图像做标准化以外的增强数据处理。
def transform_test(data, label):
    im = data.astype('float32') / 255
    auglist = image.CreateAugmenter(data_shape=(3, 32, 32), 
                        mean=np.array([0.4914, 0.4822, 0.4465]), 
                        std=np.array([0.2023, 0.1994, 0.2010]))
    for aug in auglist:
        im = aug(im)
    im = nd.transpose(im, (2,0,1))
    return (im, nd.array([label]).asscalar().astype('float32'))
```

接下来，我们可以使用`Gluon`中的`ImageFolderDataset`类来读取整理后的数据集。

```{.python .input  n=5}
input_str = data_dir + '/' + input_dir + '/'

# 读取原始图像文件。flag=1说明输入图像有三个通道（彩色）。
train_valid_ds = vision.ImageFolderDataset(input_str + 'train_valid', 
                                           flag=1, transform=transform_train)
test_ds = vision.ImageFolderDataset(input_str + 'test', flag=1, 
                                     transform=transform_test)

loader = gluon.data.DataLoader
train_valid_data = loader(train_valid_ds, batch_size, shuffle=True, last_batch='keep')
test_data = loader(test_ds, batch_size, shuffle=False, last_batch='keep')

# 交叉熵损失函数。
softmax_cross_entropy = gluon.loss.SoftmaxCrossEntropyLoss()
```

## 设计模型


```{.python .input  n=6}
import mxnet as mx
import math
from mxnet.gluon import nn

class Bottleneck(nn.HybridBlock):
    def __init__(self, growthRate):
        super(Bottleneck, self).__init__()
        interChannels = 4 * growthRate
        with self.name_scope():
            self.bn1 = nn.BatchNorm()
            self.conv1 = nn.Conv2D(
                interChannels,
                kernel_size=1,
                use_bias=False)
            self.bn2 = nn.BatchNorm()
            self.conv2 = nn.Conv2D(
                growthRate,
                kernel_size=3,
                padding=1,
                use_bias=False)

    def hybrid_forward(self, F, x):
        out = self.conv1(F.relu(self.bn1(x)))
        out = self.conv2(F.relu(self.bn2(out)))
        out = F.concat(* [x, out], dim=1)
        return out


class SingleLayer(nn.HybridBlock):
    def __init__(self, growthRate):
        super(SingleLayer, self).__init__()
        with self.name_scope():
            self.bn1 = nn.BatchNorm()
            self.conv1 = nn.Conv2D(
                growthRate,
                kernel_size=3,
                padding=1,
                use_bias=False)

    def hybrid_forward(self, F, x):
        out = self.conv1(F.relu(self.bn1(x)))
        out = nd.concat(x, out, dim = 1)
#         out = F.concat(* [x, out], 1)
        return out


class Transition(nn.HybridBlock):
    def __init__(self, nOutChannels):
        super(Transition, self).__init__()
        with self.name_scope():
            self.bn1 = nn.BatchNorm()
            self.conv1 = nn.Conv2D(
                nOutChannels,
                kernel_size=1,
                use_bias=False)

    def hybrid_forward(self, F, x):
        out = self.conv1(F.relu(self.bn1(x)))
        out = F.Pooling(out, kernel=(2, 2), stride=(2, 2), pool_type='avg')
        return out


class DenseNet(nn.HybridBlock):
    def __init__(self, growthRate, depth, reduction, nClasses, bottleneck):
        super(DenseNet, self).__init__()

        nDenseBlocks = (depth - 4) // 3
        if bottleneck:
            nDenseBlocks //= 2

        nChannels = 2 * growthRate
        with self.name_scope():
            self.conv1 = nn.Conv2D(
                nChannels,
                kernel_size=3,
                padding=1,
                use_bias=False)
            self.dense1 = self._make_dense(growthRate, nDenseBlocks,
                                           bottleneck)

        nChannels += nDenseBlocks * growthRate
        nOutChannels = int(math.floor(nChannels * reduction))
        with self.name_scope():
            self.trans1 = Transition(nOutChannels)

        nChannels = nOutChannels
        with self.name_scope():
            self.dense2 = self._make_dense(growthRate, nDenseBlocks,
                                           bottleneck)
        nChannels += nDenseBlocks * growthRate
        nOutChannels = int(math.floor(nChannels * reduction))
        with self.name_scope():
            self.trans2 = Transition(nOutChannels)

        nChannels = nOutChannels
        with self.name_scope():
            self.dense3 = self._make_dense(growthRate, nDenseBlocks,
                                           bottleneck)
        nChannels += nDenseBlocks * growthRate

        with self.name_scope():
            self.bn1 = nn.BatchNorm()
            self.fc = nn.Dense(nClasses)

    def _make_dense(self, growthRate, nDenseBlocks, bottleneck):
        layers = nn.HybridSequential()
        for i in range(int(nDenseBlocks)):
            if bottleneck:
                layers.add(Bottleneck(growthRate))
            else:
                layers.add(SingleLayer(growthRate))
        return layers

    def hybrid_forward(self, F, x):
        out = self.conv1(x)
        out = self.trans1(self.dense1(out))
        out = self.trans2(self.dense2(out))
        out = self.dense3(out)
        out = F.Pooling(
            F.relu(self.bn1(out)),
            global_pool=1,
            pool_type='avg',
            kernel=(8, 8))
        out = self.fc(out)
        return out

def get_net(ctx):
    num_outputs = 10
    net = DenseNet(growthRate=40, depth=190, reduction=0.5, bottleneck=True, nClasses=10)
    net.initialize(ctx=ctx, init=init.MSRAPrelu())
    return net
```

## 训练模型并调参

在[过拟合](../chapter_supervised-learning/underfit-overfit.md)中我们讲过，过度依赖训练数据集的误差来推断测试数据集的误差容易导致过拟合。由于图像分类训练时间可能较长，为了方便，我们这里不再使用K折交叉验证，而是依赖验证集的结果来调参。

我们定义模型训练函数。这里我们记录每个epoch的训练时间。这有助于我们比较不同模型设计的时间成本。

```{.python .input  n=7}
import datetime
import sys
sys.path.append('..')
import utils

def train(net, train_data, valid_data, num_epochs, lr, wd, ctx, lr_period, lr_decay):
    trainer = gluon.Trainer(
        net.collect_params(), 'nag', {'learning_rate': lr, 'momentum': 0.9, 'wd': wd})

    prev_time = datetime.datetime.now()
    for epoch in range(num_epochs):
        train_loss = 0.0
        train_acc = 0.0
        if epoch > 0 and epoch == lr_period:
            trainer.set_learning_rate(trainer.learning_rate * lr_decay)
            net.save_params('densenet190_40_%depoch.params' % epoch)
        elif epoch > 0 and (epoch - 75) == lr_period:
            trainer.set_learning_rate(trainer.learning_rate * lr_decay)
            net.save_params('densenet190_40_%depoch.params' % epoch)

            
        for data, label in train_data:
            label = label.as_in_context(ctx)
            with autograd.record():
                output = net(data.as_in_context(ctx))
                loss = softmax_cross_entropy(output, label)
            loss.backward()
            trainer.step(batch_size)
            train_loss += nd.mean(loss).asscalar()
            train_acc += utils.accuracy(output, label)
        cur_time = datetime.datetime.now()
        h, remainder = divmod((cur_time - prev_time).seconds, 3600)
        m, s = divmod(remainder, 60)
        time_str = "Time %02d:%02d:%02d" % (h, m, s)
        if valid_data is not None:
            valid_acc = utils.evaluate_accuracy(valid_data, net, [ctx])
            epoch_str = ("Epoch %d. Loss: %f, Train acc %f, Valid acc %f, "
                         % (epoch, train_loss / len(train_data),
                            train_acc / len(train_data), valid_acc))
        else:
            epoch_str = ("Epoch %d. Loss: %f, Train acc %f, "
                         % (epoch, train_loss / len(train_data),
                            train_acc / len(train_data)))
        prev_time = cur_time
        print(epoch_str + time_str + ', lr ' + str(trainer.learning_rate))
```

以下定义训练参数并训练模型。这些参数均可调。为了使网页编译快一点，我们这里将epoch数量有意设为1。事实上，epoch一般可以调大些，例如100。

我们将依据验证集的结果不断优化模型设计和调整参数。依据下面的参数设置，优化算法的学习率将在每80个epoch自乘0.1。

```{.python .input  n=36}
ctx = utils.try_gpu()
num_epochs = 300
learning_rate = 0.1
weight_decay = 1e-4
lr_period = 150
lr_decay = 0.1

net = get_net(ctx)
net.hybridize()
train(net, train_valid_data, None, num_epochs, learning_rate, 
      weight_decay, ctx, lr_period, lr_decay)
```

## 对测试集分类

当得到一组满意的模型设计和参数后，我们使用全部训练数据集（含验证集）重新训练模型，并对测试集分类。

```{.python .input  n=11}
import numpy as np
import pandas as pd

preds = []
for data, label in test_data:
    output = net(data.as_in_context(ctx))
    preds.extend(output.argmax(axis=1).astype(int).asnumpy())

sorted_ids = list(range(1, len(test_ds) + 1))
sorted_ids.sort(key = lambda x:str(x))

df = pd.DataFrame({'id': sorted_ids, 'label': preds})
df['label'] = df['label'].apply(lambda x: train_valid_ds.synsets[x])
df.to_csv('submission.csv', index=False)
```
