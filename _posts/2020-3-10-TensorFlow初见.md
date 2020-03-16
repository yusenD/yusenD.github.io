---
layout:     post
title:      "TensorFlow初见"
subtitle:   "TensorFlow模块架构、APIs以及一些基础知识"
date:       2020-03-10 12:00:00
author:     "Luyuan"
header-img: "img/post-bg-unix-linux.jpg"
catalog: true
tags:
    - 机器学习
    - TensorFlow
---


## 1. TensorFlow模块与APIs



![image-20200310123654422](/Users/yuhao/Library/Application Support/typora-user-images/image-20200310123654422.png)


> TODO LIST
>
> * 了解Keras
> * 了解Estimators



### 对于 LowLevel 和 Kernel



![image-20200310124027843](/Users/yuhao/Library/Application Support/typora-user-images/image-20200310124027843.png)



可以看到Python Java Go等语言实际上是基于C语言API的一层封装，C++则是完整实现了一套API。



> TODO LIST
>
> * 了解XLA gRPC RDMA
> * 公共运行时





## 2. TensorFlow数据流图



TF数据流图是一种声明式编程范式

![image-20200310124441248](/Users/yuhao/Library/Application Support/typora-user-images/image-20200310124441248.png)

数据流图主要概念有 有向边 和 节点



#### 有向边

* 张量：Tensor是一种用多维数组表示高维数据的一种抽象
* 稀疏张量：SparseTensor由于高维空间中大概率是很空旷的，使用稀疏张量可以节省大部分空间



#### 节点

* 计算节点：Operation 做一些规约、逻辑等运算
* 存储节点：Variable 整个训练过程中W（模型权重）和B（偏置）会不断迭代更新，存储节点会对这些数据进行存储
* 数据节点：Placeholder 描述图输入数据



## 3. 张量

在数学概念里面，张量是一种几何实体，广义上表示任意形式的数据。张量可以理解为0阶标量 1阶向量 2阶矩阵在高维空间上的推广。张量的阶描述它表示数据的最大维度。

在TensorFlow中，张量表示**相同数据类型的多维数组**。

 因此张量有两个重要属性：

* 数据类型（整型、浮点型、字符串）
* 数组形状（维度大小）



>Q：张量的作用
>
>A：张量是用来表示多维数据的。
>
>​		张量是执行操作时的输入或输出数据。
>
>​		用户通过执行操作来创建或计算张量。
>
>​		张量的形状不一定在编译时确定，可以在运行时通过形状推断计算得出。



#### 特殊张量

* tf.constant 常量
* tf.placeholder 占位符
* tf.variable 变量



## 变量

变量主要是维护特定节点的状态，如深度学习或机器学习的模型参数（W和B）。

`tf.Variable` 方法是操作，返回值是变量（特殊张量）   



通过 `tf.Variable`  方法创建的变量，和张量一样，可以作为操作的输入和输出。不同之处在于：

* 张量的生命周期通常随依赖的计算完成而街红薯，内存也随机释放
* 变量责常驻内存，在每一步训练时不断更新值，实现模型参数的更新。



#### 创建变量

```
tf.Variable(<initial-value>, name=<optional-name>)
```

 变量使用流程如下



![image-20200310205435996](/Users/yuhao/Library/Application Support/typora-user-images/image-20200310205435996.png)

## 5. 操作

 

TensorFlow用护具流图表示算法模型。数据流图由节点和有向边组成，每个节点均对应一个具体的操作。因此， 操作 是模型功能的**实际载体**。

数据流图中的节点按照功能不同可以分为三种：

* 存储节点：有状态的变量操作，通常用来存储模型参数
* 计算节点：无状态戴尔计算或控制操作，主要负责算法逻辑表达或流程控制
* 数据节点：数据的 占位符操作，用于描述图外输入数据的属性

操作（Operation）的输入和输出是张量或操作（函数式编程）



#### 典型计算和控制操作

![image-20200310215805356](/Users/yuhao/Library/Application Support/typora-user-images/image-20200310215805356.png)



#### 占位符操作

TF使用占位符操作表示图外输入的数据，如训练数据和测试数据。

TF数据流图描述了算法模型的计算拓扑，其中的各种操作（节点）都是抽象的函数映射或者数学表达式。

换句话说，数据流图本身是一个具有计算拓扑和内部结构的壳子，在用户向数据流图填充数据之前，图中并没有真正执行任何计算。 



## 6. 会话

会话提供了估算张量和执行操作的运行环境，它是发放计算任务的客户端，所有计算任务都由它连接的执行引擎完成，一个会话典型使用以下3步：

```python
1、 创建会话
# target 是会话连接的指定引擎，默认是本地机器
# graph 指定的数据流图
# confit 启动配置项，是否需要打开调试信息等等
sess = tf.Session(target=..., graph=..., config=...)
2、 估算张量或执行操作
sess.run(..)
3、 关闭会话
sess.close()

```



#### 会话执行



![image-20200310223150304](/Users/yuhao/Library/Application Support/typora-user-images/image-20200310223150304.png)

  

原理：

当我们调用sess.run(train_op) 语句执行训练操作时候：

* 首先，程序内部提取操作依赖的所有前置操作。这些操作的节点共同组成一幅子图。
* 然后，程序会将子图中的计算节点、存储节点和数据接待刹那按照各自执行设备分类，相同设备上的节点组成了一幅局部图。
* 最后，每个设备上的局部图在实际执行时，根据节点间的依赖关系将各个节点有序的加载到设备上执行。

## 7. 优化器



#### 前置知识：损失函数

损失函数是评估特定模型参数和特定输入时，表达模型输出的推理值与真实值之间不一致程度的函数，损失函数L的形式化定义如下：

![image-20200310224200976](/Users/yuhao/Library/Application Support/typora-user-images/image-20200310224200976.png)

常见的损失函数有平方损失函数、交叉熵损失函数和指数损失函数：

![image-20200310224307934](/Users/yuhao/Library/Application Support/typora-user-images/image-20200310224307934.png)



使用损失函数对所有训练样本求损失值，再累加求平均可得到模型的经验风险，即 f(x) 关于训练集的平均损失就是**经验风险**，其形式化定义如下：

![image-20200310225158418](/Users/yuhao/Library/Application Support/typora-user-images/image-20200310225158418.png)

如果过度追求训练数据上的低损失值，就会遇到**过拟合问题**，训练集并不能代表真实场景的数据分布，如果过分依赖训练集上的数据，遇到新数据就会无所适从，模型泛化能力就会变差。

模型训练的目标是不断最小化经验风险，随着训练步数的增加，经验风险将逐渐降低，模型复杂度也将逐渐上升、为了降低过度训练造成的过拟合风险，可以引入专门用来度量模型复杂度的正则化项（regularizer）或惩罚项(penalty term) -J(f)。常用的正则化项有L0/L1和L2范数、因此我们将最优化的目标替换为鲁棒性更好的**结构风险最小化**（structural risk minimization, SRM）。如下所示，它由经验风险项和正则项两部分构成：

![image-20200312103002545](/Users/yuhao/Library/Application Support/typora-user-images/image-20200312103002545.png)

 模型训练过程中，结构风险不断降低。当小于我们设置的损失阈值时，则认为此时的模型已经满足需求。因此，训练模型的本质就是在最小化结构风险的同事取得最优的模型参数。定义如下：

![image-20200312103317246](/Users/yuhao/Library/Application Support/typora-user-images/image-20200312103317246.png)

#### 前置知识：优化算法

典型的机器学习和深度学习通常都要转换为**最优化问题**进行求解。

求解最优化问题的算法成为**优化算法**，通常采用**迭代方式**实现：首先设置一个初始的可行解，然后基于特定的函数反复重新计算可行解，直到找到一个最优解或达到预设的收敛条件。不同的优化算法采用的迭代策略各有不同：

* 使用目标函数的一阶导数，如梯度下降法
* 使用目标函数的二阶导数，如牛顿法
* 使用前几轮迭代的信息，如Adam

基于梯度下降法的迭代策略最简单，它直接沿着梯度负方向，即**目标函数减小最快的方向**进行直线搜索。表达式如下：

![image-20200312103946157](/Users/yuhao/Library/Application Support/typora-user-images/image-20200312103946157.png)

#### TF训练机制

典型的机器学习和深度学习问题，包含以下3个部分：

1. 模型：`y=f(x)=wx+b`，其中x是输入模型，y是模型输出的推理值，w和b是模型参数，即用户的训练对象
2. 损失函数：`loss=L(y,y_-)`，其中`y_`是x对应的真实值（标签），loss是损失函数输出的真实值
3. 优化算法：`w←w+α*grad(w)`，`b←b+α*grad(b)`，其中grad(w)和grad(b)分别为当损失值为loss时，模型参数w和b各自的梯度值，α为学习率。 

![image-20200312110330655](/Users/yuhao/Library/Application Support/typora-user-images/image-20200312110330655.png)

#### TF优化器

优化器是实现优化算法的载体

一次典型的迭代优化应该分为下面3个步骤：

1. 计算梯度：调用compute_gradients方法
2. 处理梯度：用户按照自己需求处理梯度值，如梯度裁剪和梯度加权等
3. 应用梯度：调用apply_gradients方法，将处理后的梯度值应用到模型参数



![image-20200312110759576](/Users/yuhao/Library/Application Support/typora-user-images/image-20200312110759576.png)



代码实例：

![image-20200312110853731](/Users/yuhao/Library/Application Support/typora-user-images/image-20200312110853731.png)



![image-20200312111621518](/Users/yuhao/Library/Application Support/typora-user-images/image-20200312111621518.png)



这里是一些内置的优化器：

![image-20200312111716192](/Users/yuhao/Library/Application Support/typora-user-images/image-20200312111716192.png)

 

```python
import pandas as pd
import numpy as np

def normalize_feature(df):
    return df.apply(lambda column: (column - column.mean()) / column.std())


df = normalize_feature(pd.read_csv('data1.csv',
                                   names=['square', 'bedrooms', 'price']))

ones = pd.DataFrame({'ones': np.ones(len(df))})# ones是n行1列的数据框，表示x0恒为1
df = pd.concat([ones, df], axis=1)  # 根据列合并数据

X_data = np.array(df[df.columns[0:3]])
y_data = np.array(df[df.columns[-1]]).reshape(len(df), 1)

print(X_data.shape, type(X_data))
print(y_data.shape, type(y_data))
```

