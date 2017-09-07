Python numpy在slide的过程中也会有对象传递，也就是地址传递，并不是直接copy过去，这个问题困扰我一天了，详情见下例，c数组中的第一个元素随着第二个元素的增加而改变了．

---
date: 2017-05-05
title:  Python numpy中的对象传递问题
categories: 学习笔记
tags: [python]
---

解决方案是用`c.append(np.array(a[1]))`，将`a[1]`用`np.array()`方法重新申明为numpy数组，因为`np.array()`默认copy矩阵中的元素再创建一个新的numpy.ndarray对象，但是与之很相近的`np.asarray()`则不copy，这两个方法在使用的时候要注意了，关于`np.asarray()`和`np.array()`的区别，详情见[asarray vs array](http://stackoverflow.com/questions/14415741/numpy-array-vs-asarray)

```　python
# 声明a为3*3的矩阵 
>>> import numpy as np
>>> a=np.zeros((3,3))
>>> a
array([[ 0.,  0.,  0.],
       [ 0.,  0.,  0.],
       [ 0.,  0.,  0.]])

# 声明ｂ为3*1的矩阵,并赋值给a的第一行 
>>> b=np.ones((1,3))
>>> a[1]=b
>>> a
array([[ 0.,  0.,  0.],
       [ 1.,  1.,  1.],
       [ 0.,  0.,  0.]])

# 声明ｃ为空矩阵,把a的第一行append到c 
>>> c=[]
>>> c.append(a[1])
>>> c
[array([ 1.,  1.,  1.])]

# 改变b,同时赋值给a的第一行  
>>> b=np.array([1,2,3])
>>> a[1]=b
>>> a
array([[ 0.,  0.,  0.],
       [ 1.,  2.,  3.],
       [ 0.,  0.,  0.]])

# 再次把a的第一行append到c中，发现c的第一个元素也跟着改变了
>>> c.append(a[1])
>>> c
[array([ 1.,  2.,  3.]), array([ 1.,  2.,  3.])]
```
