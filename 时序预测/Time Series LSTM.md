# LSTM & Time Series

> 参考链接: [How to Develop LSTM Models for Time Series Forecasting](https://machinelearningmastery.com/how-to-develop-lstm-models-for-time-series-forecasting/)

## 单变量 时间序列

### 数据准备

我们知道, 在 机器学习中, 都需要将数据转换为 给定输入值, 预测输出值的这种形式来作为训练集, 进行学习.

而 对于简单的时间序列, 我们并不能够将时间 作为输入值, 将 结果作为输出值这种模式来进行学习.

我们的目标是:

根据已有的数据, 来预测将来的数据.

那么对应的LSTM数据模型更像是:

假定数据集 [10, 20, 30, 40, 50, 60, 70, 80, 90] 需要将数据转换成

X,				y
10, 20, 30		40
20, 30, 40		50
30, 40, 50		60
...

这种形式, 才会更符合 LSTM预测的数据模型. 即 假定 对于 t时刻, 其输入参数为 t - 1, t - 2, t - 3, 其输出是  t.

至于具体的实现, 在文档开头的参考连接中已经给出.

转换后的数据格式如下:

    [10 20 30] 40
    [20 30 40] 50
    [30 40 50] 60
    [40 50 60] 70
    [50 60 70] 80
    [60 70 80] 90

### Vanilla LSTM

单层LSTM

> A Vanilla LSTM is an LSTM model that has a single hidden layer of LSTM units, and an output layer used to make a prediction.

### Stacked LSTM

多层LSTM

> Multiple hidden LSTM layers can be stacked one on top of another in what is referred to as a Stacked LSTM model.

### Bidirectional LSTM

双向LSTM

> On some sequence prediction problems, it can be beneficial to allow the LSTM model to learn the input sequence both forward and backwards and concatenate both interpretations.

### CNN LSTM

前面三者在RNN LSTM GRU中已经做过相关了解. 而 CNN 是指卷积神经网络.

CNN的特点是在于 可以非常有效地从一维序列数据（例如单变量时间序列数据）中自动提取和学习特征。

CNN模型可以在具有LSTM后端的混合模型中使用，其中CNN用于解释输入的子序列，这些子序列一起作为序列提供给LSTM模型进行解释.

对于单变量 时序数据而言, 这种方式的 预测结果 优于前面几种结果. 对于 具有季节性的数据而言, 是否拥有更优越的特性?

### ConvLSTM