## 基础操作

生成正态分布:
```python
tf.random_normal([128, 32, 32, 3], dtype=tf.float32, stddev=1e-1)
```
生成截断正态分布
```python
tf.truncated_normal([11, 11, 3, 64], dtype=tf.float32, stddev=1e-1)
```

合并多个多维数组:
```
x = tf.constant([1, 4])
y = tf.constant([2, 5])
z = tf.constant([3, 6])
a = tf.stack([x, y, z])
x, y, z, a
```
输出:
>(<tf.Tensor 'Const_3:0' shape=(2,) dtype=int32>,
 <tf.Tensor 'Const_4:0' shape=(2,) dtype=int32>,
 <tf.Tensor 'Const_5:0' shape=(2,) dtype=int32>,
 <tf.Tensor 'stack_2:0' shape=(3, 2) dtype=int32>)

 #### 参数解析
 ```

 ```

 将常规会上转换为Tensor:
 `ops.convert_to_tensor`. C.f. https://github.com/pemami4911/TF-Queues-Full-MNIST-Example/blob/master/cnn.py

#### 计算图相关
- `tf.group`

从pb文件中创建计算图:
C.f. `models\tutorials\image\imagenet\classify_image.py`
```python
def create_graph():
  """Creates a graph from saved GraphDef file and returns a saver."""
  # Creates graph from saved graph_def.pb.
  with tf.gfile.FastGFile(os.path.join(
      FLAGS.model_dir, 'classify_image_graph_def.pb'), 'rb') as f:
    graph_def = tf.GraphDef()
    graph_def.ParseFromString(f.read())
    _ = tf.import_graph_def(graph_def, name='')
```

根据名字获取tensor: `sess.graph.get_tensor_by_name`
#### 文件IO
- `tf.gfile.Exists`
- `tf.gfile.Glob`: 不是有了`glob.glob`吗?

```python
if tf.gfile.Exists(FLAGS.log_dir):
  tf.gfile.DeleteRecursively(FLAGS.log_dir)
tf.gfile.MakeDirs(FLAGS.log_dir)
```

- 读取文件: https://www.tensorflow.org/api_guides/python/reading_data

## CNN相关
执行Local Response Normalization:
```
tf.nn.local_response_normalization(conv1,
                                              alpha=1e-4,
                                              beta=0.75,
                                              depth_radius=2,
                                              bias=2.0)
```

#### 损失函数
```
objective = tf.nn.l2_loss(pool5)
```


## TODO
- `tf.local_variables_initializer()`

### 模型分析
参考models/research/resnet/resnet_main.py, 其中也包含了各种Hook方法, 比如保存模型、日志、学习率的监控等。
