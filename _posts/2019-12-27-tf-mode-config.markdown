---
layout: post
title:  "tf mode 指定方式"
date:   2019-12-27 22:28:17 +0800
categories: tf
---

是在看tf2.0的文档的时候，tutorial里没用training，导致我去翻例子，干脆把1.x也理一遍。

有空再补代码例子，因为是短时间稍微写了一下，不是很详细，有例子应该会清楚很多。

### tl;dr

总共就这么短就没有tldr了。。

简单说就是：

> 可以在构建图的时候静态指定training或者run session的时候传入placeholder

只不过tf调用方式本身的多样性导致了问题的复杂，有空单独写一个tf调用方式的总结。

### 正文

用keras时，不用关心，在fit和predict函数里自动设置。

tf2.0 里keras的model.fit好像也支持dict类型的输入了，一般用fit就够了

### tf 1.0 的两种接口

- tf.nn.xx：

    算子接口，不带变量，典型场景是使用下面第一种方式调用；

- tf.layers.xx:

    算子内生成变量，tf.layers.Xx只是keras.layers封装了一个名字；tf.layers.xx比较尴尬，调用的时候生成对应的算子对象然后apply，别用这类接口了，1.x高版本里大部分都被干掉了。目前全面移到tf.keras.layers.

    看到[有人][tf1-example]结合variable scope来用tf.layers.xx，不知道能不能复用。

    看tf.keras的[源码(tf1.15)][tf1.15-make-variable]，好像是不行，可能更早版本的tf.layers.xx不是用keras吧？

    看了下[tf1.4][tf1.4-source]还没集成keras的时候，确实是用的get_variable，有空再写个实验看看吧

- tf.keras.layers.xx:

    自带变量，training是个tensor，动态判断。手动训练的时候，在session里传入K.learning_phase()生成的placeholder


### tf不同使用方式的mode指定方式

tf2.0的官方建议的[几种调用方式][tf2.0-guide]，都是keras的，这里列一下更一般的方法。

1. variable scope

    tf1的经典调用方式，虽然官方没有建议，但是一些老项目还是用的很多的，纯种tf调用方式。

    - 变量作用域都是用“tf语言”定义和管理的。

    - 用get_variable(..., reuse=tf.AUTO_REUSE)的方式共享变量

    - 为train和infer分别构建图，用共享变量来共享参数，在构造图时传training参数（这是一个python变量）静态指定

    **算子和变量是分离的**

    和上面tf.layers.xx方式不同，算子和变量是分离的，可以用不同的参数构造任意个算子，并且共享变量。

2. object oriented

    用python的对象来管理scope，把算子或者tf变量定义成类属性。

    这种方式用现成的layers比较多，虽然也可以用定义tf变量的方式，但是搭tf.keras.layers使用，可以少写很多变量定义的代码。

    **tf.layers.xx和tf.keras.layers里算子和变量是绑定的**

    “用相同的一批算子，定义train和infer图”

    有两种方式指定mode：

    - train/infer混用图，在session.run的时候通过给training这个tensor（tf.keras.layers.xx根据K.learning_phase()定义行为）传入不同的值来动态指定“同一个算子”的不同行为

    - 用相同的一批算子，分别定义train和infer图（和第一种vs类似），定义图的时候，算子的apply()函数可以静态指定training（python变量）。这种方式个人感觉只是比vs方式少写一点变量定义的代码。

3. tf 2.0 eager模式

    用默认的eager模式几乎和pytorch一样了，并且可以用tf.function加速，非常方便，除了库代码不好读。
    和pytorch的接口稍微有点区别，

    - pytorch:
    ```python
    model.train()
    ...
    ```
    - tf2:
    ```python
    model(x, training=True)
    ```

    答应我一定要手动指定，不要只看官方手册，官方的example repo有[完整例子][tf2-example]

[tf1-example]: https://github.com/aymericdamien/TensorFlow-Examples/blob/master/examples/3_NeuralNetworks/convolutional_network.py
[tf2-example]: https://github.com/tensorflow/examples/blob/master/tensorflow_examples/models/dcgan/dcgan.py
[tf1.4-source]: https://github.com/tensorflow/tensorflow/blob/r1.4/tensorflow/python/layers/base.py#L453
[tf1.15-make-variable]: https://github.com/tensorflow/tensorflow/blob/r1.15/tensorflow/python/keras/engine/base_layer_utils.py#L63
[tf2.0-guide]: https://medium.com/tensorflow/standardizing-on-keras-guidance-on-high-level-apis-in-tensorflow-2-0-bad2b04c819a
