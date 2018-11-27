---
layout: post
title:  "tensorflow模型导出记录"
categories: python
tags:  tensorflow 模型部署 非标准模型
---

* content
{:toc}

## 摘要
本文记录部署一个非标准模型（未定义name、未定义placeholder、未定义batchnorm中的train参数）的过程

## 引言
之前训练的一个比较好的模型需要部署到实际应用场景中，但从之前训练时到现在，tensorflow版本已经更新了7、8个，一些借口已经改变。给部署带来一定的难度

## 主题
本文在旧的tensorflow版本上先进行模式导出和试验，成功后再部署到新的tensorflow版本上，先采用最基础的meta方式进行导入导出

## meta方式
参考[Tensorflow加载预训练模型和保存模型](https://blog.csdn.net/huachao1001/article/details/78501928)

查看模型所有name代码 `print('graph names:{}'.format(graph._names_in_use))`

## 查找最终输出的op
原模型最终代码：
```
# 全连接层 + Softmax
with tf.variable_scope('logit'):
    logits = self._fully_connected(x, self.hps.num_classes)

```
`self._fully_connected`最后一步调用了`tf.nn.xw_plus_b(x, w, b)`
写一个例子程序，用来查看默认的输出op名称
```
graph = tf.get_default_graph()
w = tf.Variable(np.random.randn(5, 5).astype('float32'),name="w")
x = tf.Variable(np.random.randn(5, 5).astype('float32'), name="w2")
b = tf.Variable(np.random.randn(5).astype('float32'), name="b")
tf.nn.xw_plus_b(w, x, b)
print('graph names:{}'.format(graph._names_in_use))
```
查看输出为：
```
graph names:{'w/read': 1, 'b/assign': 1, 'b': 1, 'w2/read': 1, 'xw_plus_b/matmul': 1, 'w2': 1, 'w2/assign': 1, 'b/initial_value': 1, 'w/assign': 1, 'w/initial_value': 1, 'w': 1, 'b/read': 1, 'xw_plus_b': 1, 'w2/initial_value': 1}
```
可以看到`xw_plus_b`为操作的原始OP，那么原模型最终输出op为`"logit/xw_plus_b:1"`，模型部署时的输出代码为
```
op_logit = graph.get_tensor_by_name("logit/xw_plus_b:0")
logits=sess.run(op_logit, feed_dict)
predictions = np.argmax(logits, axis=1)
```

## 查找输入tensor
写一个例子程序，用来查看默认的输入op名称
```
graph = tf.get_default_graph()
image = tf.Variable(np.random.randn(5, 60000).astype('float32'),name="image")
label = tf.Variable(np.random.randn(5).astype('int'), name="label")
data_num = tf.Variable(np.random.randn(5).astype('int'), name="data_num")
data_num, images, sparse_labels = tf.train.shuffle_batch(
        [data_num, image, label], batch_size=5, num_threads=2,
        capacity=20,        min_after_dequeue=10)
print('graph names:{}'.format(graph._names_in_use))
```

查看输出为：
```
graph names:{'shuffle_batch/tofloat': 1, 'image/read': 1, 'label': 1, 'image': 1, 'shuffle_batch/sub': 1, 'shuffle_batch/const': 1, 'label/assign': 1, 'label/read': 1, 'shuffle_batch/random_shuffle_queue_close': 2, 'shuffle_batch/maximum': 1, 'shuffle_batch/random_shuffle_queue_enqueue': 1, 'data_num': 1, 'shuffle_batch/mul/y': 1, 'image/initial_value': 1, 'shuffle_batch/random_shuffle_queue': 1, 'shuffle_batch/sub/y': 1, 'data_num/assign': 1, 'shuffle_batch/random_shuffle_queue_close_1': 1, 'label/initial_value': 1, 'data_num/read': 1, 'image/assign': 1, 'shuffle_batch/random_shuffle_queue_size': 1, 'shuffle_batch/n': 1, 'shuffle_batch/fraction_over_10_of_10_full/tags': 1, 'shuffle_batch': 1, 'shuffle_batch/maximum/x': 1, 'data_num/initial_value': 1, 'shuffle_batch/mul': 1, 'shuffle_batch/fraction_over_10_of_10_full': 1}
```
可以看到`shuffle_batch`为操作的原始OP，那么原模型输入tensor为`"input/shuffle_batch:1"`，模型部署时的输出代码为
```
op_logit = graph.get_tensor_by_name("logit/xw_plus_b:0")
logits=sess.run(op_logit, feed_dict)
predictions = np.argmax(logits, axis=1)
```

## 验证能否强行使用feed_dict改变变量的值
写一个例子程序，用来查看强行填充feed_dict的效果
```
graph = tf.get_default_graph()

w = tf.placeholder("float", name="w")
w1 = tf.Variable(5.0, name="w2")
x = tf.Variable(2.0, name="w2")
feed_dict={w:5.0,w1:4.0}
y = tf.multiply(w,x)
y1 = tf.multiply(w1,x)

sess = tf.Session()
sess.run(tf.global_variables_initializer())
ty,ty1=sess.run([y,y1],feed_dict=feed_dict)
print(ty)
print(ty1)
```
查看输出为：
```
10.0
8.0
```
可以看到`y1`的值为强制填充后计算结果`4*2=8`，feeddict有效

在原模型中加入强行填充feed_dict代码
```
images_placeholder = np.random.randn(100, 60000).astype('float32')
feed_dict={test_data: images_placeholder}
```

#### batchnorm问题
[tensorflow中batch normalization的用法](https://www.cnblogs.com/hrlnw/p/7227447.html)
batchnorm参数需要在训练时加入代码和重新训练，暂不考虑加入，通过每次预测输入足量样本来解决

## 附录


## 参考
---
- [Tensorflow加载预训练模型和保存模型](https://blog.csdn.net/huachao1001/article/details/78501928)
- [tensorflow中batch normalization的用法](https://www.cnblogs.com/hrlnw/p/7227447.html)
