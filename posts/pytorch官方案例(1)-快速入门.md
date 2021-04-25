---
title: "Pytorch官方案例(1) 快速入门"
date: 2021-04-25T22:08:51+08:00
draft: False
---
[toc]

## 快速入门

这个一小节, 包含了机器学习中的基本流程

* 数据准备
* 创建模型
* 定义优化器
* 保存模型参数
* 加载模型

### 1. 数据准备

pytorch有两个处理数据的工具:torch.utils.data. DataLoader和torch.utils.data. Dataset
Dataset存储样本及其相应的标签，而DataLoader在数据集上包装一个可迭代对象。

``` python
import torch
from torch import nn
from torch.utils.data import DataLoader
from torchvision import datasets
from torchvision.transforms import ToTensor, Lambda, Compose
import matplotlib.pyplot as plt
```

``` python
#从公开数据集下载训练数据
training_data = datasets.FashionMNIST(
    root="../data",
    train=True,
    download=False,#如果需要下载改为True
    transform=ToTensor(),
)

#从公开数据集下载测试数据
test_data = datasets.FashionMNIST(
    root="../data",
    train=False,
    download=False,
    transform=ToTensor(),
)
```

我们把Dataset当作一个参数传给DataLoader. 这个迭代器, 支持自动批处理, 采样
随机打乱数据, 多进程加载数据.
这里我们定义batch大小为64, Dataloader迭代器每次将会返回一个batch, 包含feature和labels

``` python
batch_size = 64

#创建dataloaders
train_dataloader = DataLoader(training_data, batch_size=batch_size)
test_dataloader = DataLoader(test_data, batch_size=batch_size)

for X, y in test_dataloader:
    print("X的维度[N,C,H,W]:",X.shape)     #[N,C,H,W] [在索引中的编号,通道,高,宽]
    print("y的维度: ", y.shape, y.dtype)
    break
```

输出：
![](https://gitee.com/wubmu/image/raw/master/img/20210425214805.png)

### 2. 模型创建

pytorch定义神经网络, 需要创建一个类继承nn. Module.
`__init__` : 定义网络层数
`forword` : 定义数据在网络中的流向

``` python
# 如果有GPU采用gpu加速训练
device = "cuda" if torch.cuda.is_available() else "cpu"

# 定义模型
class DNN(nn.Module):
    def __init__(self):
        super(DNN, self).__init__()
        self.flatten = nn.Flatten()     #原始数据是[1,28,28],把它拉直  变成[28*28]一维
        self.linear_relu_stack = nn.Sequential(
            nn.Linear(28*28, 512),
            nn.ReLU(),
            nn.Linear(512, 512),
            nn.ReLU(),
            nn.Linear(512, 10),         #有10个类别
            nn.ReLU()
        )

    def forward(self, x):
        x = self.flatten(x)
        x = self.linear_relu_stack(x)
        return x

model = DNN().to(device)
print(model)
```

输出
![](https://gitee.com/wubmu/image/raw/master/img/20210425214943.png)

### 3. 定义优化器和损失函数

``` python
loss_fn = nn.CrossEntropyLoss()
optimizer = torch.optim.Adam(model.parameters(), lr=0.001)
```

在训练过程中，在单次循环中, 模型需要对训练数据做预测, 并且反向传播更新模型参数

``` python
def train(dataloader, model, loss_fn, optimizer):
    size = len(dataloader.dataset)
    for batch, (X, y) in enumerate(dataloader):
        X, y = X.to(device), y.to(device)

        # 前向传播和计算误差
        pred = model(X)
        loss = loss_fn(pred, y) #交叉熵会自动对y进行one-hot

        #反向传播
        optimizer.zero_grad()   #梯度清零
        loss.backward()         #方向传播
        optimizer.step()        #更新模型

        if batch % 100 == 0:   #每100个batch打印一下误差
            loss, current = loss.item(), batch*len(X)
            print(f'loss: {loss:>7f}    [{current:>5d}/{size:>5d}]')
```

我们还将对照测试数据集检查模型的性能，以确保模型是可学习的。

``` python
def test(dataloader, model):
    # size = len(dataloader.dataset)
    size = len(dataloader.dataset)
    model.eval()
    test_loss, correct = 0, 0
    with torch.no_grad():   #测试集不用更新参数,不记录梯度
        for X, y in dataloader:
            X , y = X.to(device), y.to(device)
            pred = model(X)
            test_loss += loss_fn(pred, y).item()
            correct += (pred.argmax(1) == y).type(torch.float).sum().item()
            #pred.argmax(1) 找到概率最大的索引位置, 即预测的label
            #(pred.argmax(1) == y) 是否与 y的label相等
            #(pred.argmax(1) == y).type(torch.float).sum():统计true的个数 ,true转换成float为1

    test_loss /= size
    correct /= size
    print(f"Test Error: \n Accuracy: {(100*correct):>0.1f}%, Avg loss: {test_loss:>8f} \n")
```

训练的过程需要迭代多次(epoch)

``` python
epochs = 5
for t in range(epochs):
    print(f"Epoch {t+1}\n------------------------------")
    train(train_dataloader, model, loss_fn, optimizer)
    test(test_dataloader, model)
print("Done!")
```

``` python
out:
Epoch 1
------------------------------
loss: 2.298897    [    0/60000]
loss: 1.661566    [ 6400/60000]
loss: 1.727862    [12800/60000]
loss: 1.804164    [19200/60000]
loss: 1.650022    [25600/60000]
loss: 1.702805    [32000/60000]
loss: 1.369759    [38400/60000]
loss: 1.585616    [44800/60000]
loss: 1.414686    [51200/60000]
loss: 1.726880    [57600/60000]
Test Error: 
 Accuracy: 35.1%, Avg loss: 0.026484 

Epoch 2
------------------------------
loss: 1.485116    [    0/60000]
loss: 1.520031    [ 6400/60000]
loss: 1.653199    [12800/60000]
loss: 1.769573    [19200/60000]
loss: 1.478098    [25600/60000]
loss: 1.689139    [32000/60000]
loss: 1.349994    [38400/60000]
loss: 1.552191    [44800/60000]
loss: 1.387923    [51200/60000]
loss: 1.632932    [57600/60000]
Test Error: 
 Accuracy: 36.9%, Avg loss: 0.025579 

Epoch 3
------------------------------
loss: 1.375883    [    0/60000]
loss: 1.485450    [ 6400/60000]
loss: 1.643194    [12800/60000]
loss: 1.745134    [19200/60000]
loss: 1.489459    [25600/60000]
loss: 1.683173    [32000/60000]
loss: 1.332691    [38400/60000]
loss: 1.527047    [44800/60000]
loss: 1.088369    [51200/60000]
loss: 1.412117    [57600/60000]
Test Error: 
 Accuracy: 56.1%, Avg loss: 0.019232 

Epoch 4
------------------------------
loss: 0.949627    [    0/60000]
loss: 1.063320    [ 6400/60000]
loss: 1.107010    [12800/60000]
loss: 1.242038    [19200/60000]
loss: 1.226886    [25600/60000]
loss: 1.273832    [32000/60000]
loss: 1.029097    [38400/60000]
loss: 1.310232    [44800/60000]
loss: 0.952710    [51200/60000]
loss: 1.395312    [57600/60000]
Test Error: 
 Accuracy: 56.5%, Avg loss: 0.018658 

Epoch 5
------------------------------
loss: 0.894558    [    0/60000]
loss: 1.040670    [ 6400/60000]
loss: 1.136559    [12800/60000]
loss: 1.185049    [19200/60000]
loss: 1.184287    [25600/60000]
loss: 1.242846    [32000/60000]
loss: 0.998598    [38400/60000]
loss: 1.194539    [44800/60000]
loss: 0.940521    [51200/60000]
loss: 1.336018    [57600/60000]
Test Error: 
 Accuracy: 56.6%, Avg loss: 0.018490 

Done!
```

### 4. 模型存储

``` python
torch.save(model.state_dict(),"DNN.pth")
print("把模型参数保存在DNN.pth")
```

### 5. 加载模型

加载模型的过程包括重新创建模型结构并将参数加载到其中。

``` python
model2 = DNN()
model.load_state_dict(torch.load("DNN.pth"))

# 做一次预测
classes = [
    "T-shirt/top",
    "Trouser",
    "Pullover",
    "Dress",
    "Coat",
    "Sandal",
    "Shirt",
    "Sneaker",
    "Bag",
    "Ankle boot",
]

model2.eval()
x, y = test_data[3][0], test_data[3][1]
with torch.no_grad():
    pred = model2(x)
    predicted, actual = classes[pred[0].argmax(0)], classes[y]
    print(f'预测值: "{predicted}", 实际值: "{actual}"')
```
