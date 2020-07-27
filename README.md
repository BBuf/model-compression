# model-compression

"目前在深度学习领域分类两个派别，一派为学院派，研究强大、复杂的模型网络和实验方法，为了追求更高的性能；另一派为工程派，旨在将算法更稳定、高效的落地在硬件平台上，效率是其追求的目标。复杂的模型固然具有更好的性能，但是高额的存储空间、计算资源消耗是使其难以有效的应用在各硬件平台上的重要原因。所以，卷积神经网络日益增长的深度和尺寸为深度学习在移动端的部署带来了巨大的挑战，深度学习模型压缩与加速成为了学术界和工业界都重点关注的研究领域之一"


## 项目简介 

基于pytorch实现模型压缩（1、量化：8/4/2 bits(dorefa)、三值/二值(twn/bnn/xnor-net)；2、剪枝：正常、规整、针对分组卷积结构的通道剪枝；3、分组卷积结构；4、针对特征A二值的BN融合）


## 目前提供

- 1、普通卷积和分组卷积结构
- 2、权重W和特征A的训练中量化, W(32/8/4/2bits, 三/二值) 和 A(32/8/4/2bits, 三/二值)任意组合
- 3、针对三/二值的一些tricks：W二值/三值缩放因子，W/grad（ste、saturate_ste、soft_ste）截断，W三值_gap(防止参数更新抖动)，W/A二值时BN_momentum(<0.9)，A二值时采用B-A-C-P可比C-B-A-P获得更高acc
- 4、多种剪枝方式：正常剪枝、规整剪枝（比如model可剪枝为每层剩余filter个数为N(8,16等)的倍数）、针对分组卷积结构的剪枝（剪枝后仍保证分组卷积结构）
- 5、batch normalization的融合及融合前后model对比测试：普通融合（BN层参数 —> conv的权重w和偏置b）、针对特征A二值的融合（BN层参数 —> conv的偏置b)


## 代码结构

![img1](https://github.com/666DZY666/model-compression/blob/master/readme_imgs/code_structure.jpg)

## 环境要求

- python >= 3.5
- torch >= 1.1.0
- torchvison >= 0.3.0
- numpy

## BBuf自测环境
此环境通过所有测试，建议和我保持一致。
- python 3.6.2
- torch == 1.1.0
- cuda 10.0
- torchvison == 0.3.0
- numpy

## 使用

### 量化

#### W（FP32/三/二值）、A（FP32/三/二值）

--W --A, 权重W和特征A量化取值

```
cd WbWtAb
```

- WbAb

```
python main.py --W 2 --A 2
```

- WbA32

```
python main.py --W 2 --A 32
```

- WtAb

```
python main.py --W 3 --A 2
```

- WtA32

```
python main.py --W 3 --A 32
```

#### W（FP32/8/4/2 bits）、A（FP32/8/4/2 bits）

--Wbits --Abits, 权重W和特征A量化位数

```
cd WqAq
```

- W8A8

```
python main.py --Wbits 8 --Abits 8
```

- W4A8

```
python main.py --Wbits 4 --Abits 8
```

- W4A4

```
python main.py --Wbits 4 --Abits 4
```

- 其他bits情况类比

### 剪枝

稀疏训练 ——> 剪枝 ——> 微调

```
cd prune
```

#### 正常训练

```
python main.py
```

#### 稀疏训练

-sr 稀疏标志, --s 稀疏率(需根据dataset、model情况具体调整)

- nin(正常卷积结构)

```
python main.py -sr --s 0.0001
```

- nin_gc(含分组卷积结构)

```
python main.py -sr --s 0.001
```

#### 剪枝

--percent 剪枝率, --normal_regular 正常、规整剪枝标志及规整剪枝基数(如设置为N,则剪枝后模型每层filter个数即为N的倍数), --model 稀疏训练后的model路径, --save 剪枝后保存的model路径（路径默认已给出, 可据实际情况更改）

- 正常剪枝

```
python normal_regular_prune.py --percent 0.5 --model models_save/nin_preprune.pth --save models_save/nin_prune.pth
```

- 规整剪枝

```
python normal_regular_prune.py --percent 0.5 --normal_regular 8 --model models_save/nin_preprune.pth --save models_save/nin_prune.pth
```

或

```
python normal_regular_prune.py --percent 0.5 --normal_regular 16 --model models_save/nin_preprune.pth --save models_save/nin_prune.pth
```

- 分组卷积结构剪枝

```
python gc_prune.py --percent 0.4 --model models_save/nin_gc_preprune.pth
```

#### 微调

--refine 剪枝后的model路径（在其基础上做微调）

```
python main.py --refine models_save/nin_prune.pth
```

### 剪枝 —> 量化（注意剪枝率和量化率平衡）

剪枝完成后,加载保存的模型参数在其基础上再做量化

#### 剪枝 —> 量化（8/4/2 bits）（剪枝率偏大、量化率偏小）

```
cd WqAq
```

- W8A8
- nin(正常卷积结构)

```
python main.py --Wbits 8 --Abits 8 --refine ../prune/models_save/nin_refine.pth
```

- nin_gc(含分组卷积结构)

```
python main.py --Wbits 8 --Abits 8 --refine ../prune/models_save/nin_gc_refine.pth
```

- 其他bits情况类比

#### 剪枝 —> 量化（三/二值）（剪枝率偏小、量化率偏大）

```
cd WbWtAb
```

- WbAb
- nin(正常卷积结构)

```
python main.py --W 2 --A 2 --refine ../prune/models_save/nin_refine.pth
```

- nin_gc(含分组卷积结构)

```
python main.py --W 2 --A 2 --refine ../prune/models_save/nin_gc_refine.pth
```

- 其他取值情况类比

### BN融合

```
cd WbWtAb/bn_merge
```

--W 权重W量化取值(据训练时W量化(FP/三值/二值)情况而定)

#### 融合并保存融合前后model

```
python bn_merge.py --W 2
```

或

```
python bn_merge.py --W 3
```

#### 融合前后model对比测试

```
python bn_merge_test_model.py
```

## 模型压缩数据对比（示例）

注：可在更冗余模型、更大数据集上尝试其他组合压缩方式

|            类型             |  Acc   | GFLOPs | Para(M) | Size(MB) | 压缩率 | 损失  |                        备注                        |
| :-------------------------: | :----: | :----: | :-----: | :------: | :----: | :---: | :------------------------------------------------: |
|        原模型（nin）        | 91.01% |  0.15  |  0.67   |   2.68   |  ***   |  ***  |                       全精度                       |
|  采用分组卷积结构(和原始网络不一致)   | 90.88% |  0.15  |  0.58   |   2.32   | 13.43% | 0.13% |                       全精度                       |
|            剪枝             | 90.26% |  0.09  |  0.32   |   1.28   | 44.83% | 0.62% |                       全精度                       |
|        量化(W/A二值)        | 90.02% |  ***   |   ***   |   0.18   | 92.21% | 0.86% |                  W二值含缩放因子                   |
|      量化(W三值/A二值)      | 87.68% |  ***   |   ***   |   0.26   | 88.79% | 3.20% | W三值不含缩放因子,适配一些无法做缩放因子运算的硬件 |
|   剪枝+量化(W三值/A二值)    | 86.13% |  ***   |   ***   |   0.19   | 91.81% | 4.75% | W三值不含缩放因子,适配一些无法做缩放因子运算的硬件 |
| 分组+剪枝+量化(W三值/A二值) | 86.13% |  ***   |   ***   |   0.19   | 92.91% | 4.88% | W三值不含缩放因子,适配一些无法做缩放因子运算的硬件 |

## 模型压缩数据对比(BBuf自测版)
|            类型             | Epoch |Acc   | GFLOPs | Para(M) | Size(MB) | 压缩率 | 损失  |                        备注                        |
| :-------------------------: |:---:|:----: | :----: | :-----: | :------: | :----: | :---: | :---------: |
|        原模型（nin）        | 50|88.03% |  0.21  |  0.92   |   3.9   |  ***   |  ***  |                       全精度                       |
|  采用分组卷积结构(和原始网络一致)   | 50|84.88% |  0.05  |0.08     | 0.36     | 91% |  3.15%|                       全精度                       |
|  采用分组卷积结构(和原始网络不一致)   | 50|  88.13%  |  0.15   |  0.56    |2.4  | 39% |0.0% |                      全精度                       |
|            原模型常规剪枝(比例为0.5)             | 50| 25.01%|   0.05 | 0.26    | 1.1     | 72%| 63.02% |                       全精度                       |
|            原模型常规剪枝(比例为0.5)+finutune             | 10|87.76% |  0.05  | 0.26    | 1.1     |72% | 0.27% |                       全精度                       |
|            原始模型常规剪枝(比例为0.5)+规整剪枝(通道数为8)             | 50|25.63% | 0.06   | 0.26    |  1.1    | 72%| 62.4% |                       全精度                       |
|            原始模型常规剪枝(比例为0.5)+规整剪枝(通道数为8)+finetune             | 10|87.52% |  0.06  | 0.26    |1.1    |72% | 0.51% |                       全精度                       |
|        量化(W/A二值)        | 50| |     |      |      |  |  |                  W二值含缩放因子                   |
|      量化(W三值/A二值)      | 50| |     |      |      | | | W三值不含缩放因子,适配一些无法做缩放因子运算的硬件 |
|   剪枝+量化(W三值/A二值)    | 50| |     |      |      |  | | W三值不含缩放因子,适配一些无法做缩放因子运算的硬件 |
| 分组+剪枝+量化(W三值/A二值) | 50| |     |      |     |  | | W三值不含缩放因子,适配一些无法做缩放因子运算的硬件 |

## 注意
- 第二，三个分组卷积网络使用了ShuffleNet的通道Shuffle策略，即有规律的通道打乱。
- 全精度的意思是Float32。
- 训练和测试过程的`batch_size`均设置为50。
- 测试的时候剪枝剪枝率都设为0.5。
- 规整剪枝的意思是将所有的通道数都剪枝到预设值(这里为8)的倍数。
- 剪枝的时候微调10个epoch
- 测试环境：GTX2070

## 网络结构对比
|类型|卷积层通道数变化|网络结构各层形状详细参数|
|:---:|:---:|:---:|
|原模型（nin）|cfg = [192, 160, 96, 192, 192, 192, 192, 192]|[Here](Network.md)|
|采用分组卷积结构(和原始网络一致)|cfg = [192, 160, 96, 192, 192, 192, 192, 192]|[Here](Network.md)|
|采用分组卷积结构(和原始网络不一致)|cfg = [256, 256, 256, 512, 512, 512, 1024, 1024]|[Here](Network.md)|
|原模型常规剪枝(比例为0.5)|cfg=[47, 93, 47, 122, 116, 90, 105, 83]|[Here](Network.md)|
|原始模型常规剪枝(比例为0.5)+规整剪枝(通道数为8)|[48, 96, 48, 120, 120, 88, 104, 80]|[Here](Network.md)|


## BBuf自测

在CIFAR10做Quantization Aware Training实验，网络结构为：



```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from .util_wqaq import Conv2d_Q, BNFold_Conv2d_Q

class QuanConv2d(nn.Module):
    def __init__(self, input_channels, output_channels,
            kernel_size=-1, stride=-1, padding=-1, groups=1, last_relu=0, abits=8, wbits=8, bn_fold=0, q_type=1, first_layer=0):
        super(QuanConv2d, self).__init__()
        self.last_relu = last_relu
        self.bn_fold = bn_fold
        self.first_layer = first_layer

        if self.bn_fold == 1:
            self.bn_q_conv = BNFold_Conv2d_Q(input_channels, output_channels,
                    kernel_size=kernel_size, stride=stride, padding=padding, groups=groups, a_bits=abits, w_bits=wbits, q_type=q_type, first_layer=first_layer)
        else:
            self.q_conv = Conv2d_Q(input_channels, output_channels,
                    kernel_size=kernel_size, stride=stride, padding=padding, groups=groups, a_bits=abits, w_bits=wbits, q_type=q_type, first_layer=first_layer)
            self.bn = nn.BatchNorm2d(output_channels, momentum=0.01) # 考虑量化带来的抖动影响,对momentum进行调整(0.1 ——> 0.01),削弱batch统计参数占比，一定程度抑制抖动。经实验量化训练效果更好,acc提升1%左右
        self.relu = nn.ReLU(inplace=True)

    def forward(self, x):
        if not self.first_layer:
            x = self.relu(x)
        if self.bn_fold == 1:
            x = self.bn_q_conv(x)
        else:
            x = self.q_conv(x)
            x = self.bn(x)
        if self.last_relu:
            x = self.relu(x)
        return x

class Net(nn.Module):
    def __init__(self, cfg = None, abits=8, wbits=8, bn_fold=0, q_type=1):
        super(Net, self).__init__()
        if cfg is None:
            cfg = [192, 160, 96, 192, 192, 192, 192, 192]
        # model - A/W全量化(除输入、输出外)
        self.quan_model = nn.Sequential(
                QuanConv2d(3, cfg[0], kernel_size=5, stride=1, padding=2, abits=abits, wbits=wbits, bn_fold=bn_fold, q_type=q_type, first_layer=1),
                QuanConv2d(cfg[0], cfg[1], kernel_size=1, stride=1, padding=0, abits=abits, wbits=wbits, bn_fold=bn_fold, q_type=q_type),
                QuanConv2d(cfg[1], cfg[2], kernel_size=1, stride=1, padding=0, abits=abits, wbits=wbits, bn_fold=bn_fold, q_type=q_type),
                nn.MaxPool2d(kernel_size=3, stride=2, padding=1),
                
                QuanConv2d(cfg[2], cfg[3], kernel_size=5, stride=1, padding=2, abits=abits, wbits=wbits, bn_fold=bn_fold, q_type=q_type),
                QuanConv2d(cfg[3], cfg[4], kernel_size=1, stride=1, padding=0, abits=abits, wbits=wbits, bn_fold=bn_fold, q_type=q_type),
                QuanConv2d(cfg[4], cfg[5], kernel_size=1, stride=1, padding=0, abits=abits, wbits=wbits, bn_fold=bn_fold, q_type=q_type),
                nn.MaxPool2d(kernel_size=3, stride=2, padding=1),
                
                QuanConv2d(cfg[5], cfg[6], kernel_size=3, stride=1, padding=1, abits=abits, wbits=wbits, bn_fold=bn_fold, q_type=q_type),
                QuanConv2d(cfg[6], cfg[7], kernel_size=1, stride=1, padding=0, abits=abits, wbits=wbits, bn_fold=bn_fold, q_type=q_type),
                QuanConv2d(cfg[7], 10, kernel_size=1, stride=1, padding=0, last_relu=1, abits=abits, wbits=wbits, bn_fold=bn_fold, q_type=q_type),
                nn.AvgPool2d(kernel_size=8, stride=1, padding=0),
                )

    def forward(self, x):
        x = self.quan_model(x)
        x = x.view(x.size(0), -1)
        return x
```



训练Epoch数为30，学习率调整策略为：



```python
def adjust_learning_rate(optimizer, epoch):
    if args.bn_fold == 1:
        if args.model_type == 0:
            update_list = [12, 15, 25]
        else:
            update_list = [8, 12, 20, 25]
    else:
        update_list = [15, 17, 20]
    if epoch in update_list:
        for param_group in optimizer.param_groups:
            param_group['lr'] = param_group['lr'] * 0.1
    return
```





| 类型                 | Acc    | 备注   |
| -------------------- | ------ | ------ |
| 原模型(nin)          | 91.01% | 全精度 |
| 对称量化, bn不融合   | 88.88% | INT8   |
| 对称量化，bn融合     | 86.66% | INT8   |
| 非对称量化，bn不融合 | 88.89% | INT8   |
| 非对称量化，bn融合   | 87.30% | INT8   |

## 后续补充

- 1、使用示例部分细节说明
- 2、参考论文及工程
- 3、imagenet测试（目前cifar10）

## 后续扩充

- 1、Nvidia、Google的INT8量化方案
- 2、对常用检测模型做压缩
- 3、部署（1、针对4bits/三值/二值等的量化卷积；2、终端DL框架（如MNN，NCNN，TensorRT等））


## 一些额外小实验
### 验证深度可分离卷积1*1卷积通道倍增是否可以保持准确率
|            类型             | Epoch |Acc   | GFLOPs | Para(M) | Size(MB) | 压缩率 | 损失  |
| :-------------------------: |:---:|:----: | :----: | :-----: | :------: | :----: | :---: |
|        原始的卷积        | 300 | 88.41%|  0.34  |   2.56 |   10.8   |  ***   |  ***  |
|        原始的深度可分离卷积        | 300 |87.35% |  0.15  |  1.48   |   6.3   |  42%   |  1.06% |
|  深度可分离卷积+对应的1*1卷积通道倍增  | 300 | 87.72%| 0.30   | 2.97   |  6.3    | 42% | 0.69% |

- 原始卷积网络模型通各个卷积层道数设置为cfg = [32, 64, 128, 256, 256, 256, 512, 1024]
- 3个网络的详细结构见[Here](Network_depthwise_conv.md)
