
原文参见: [Understanding LSTM in Tensorflow(MNIST dataset)](https://jasdeep06.github.io/posts/Understanding-LSTM-in-Tensorflow-MNIST/?spm=5176.100239.0.0.F2BDyz)

本笔记旨在了解TensorFlow中LSTM的基础输入格式。

cell和unit:
![](https://github.com/jasdeep06/jasdeep06.github.io/blob/master/posts/Understanding-LSTM-in-Tensorflow-MNIST/images/num_units.png?raw=True)

输入格式:
![](https://github.com/jasdeep06/jasdeep06.github.io/blob/master/posts/Understanding-LSTM-in-Tensorflow-MNIST/images/inputs.png?raw=True)


```python
import sys
import tensorflow as tf
from tensorflow.contrib import rnn
from tensorflow.examples.tutorials.mnist import input_data
```


```python
time_steps = 28
num_units = 128
n_input = 28
learning_rate = 0.001
n_classes = 10
batch_size = 128

mnist = input_data.read_data_sets("./MNIST/", one_hot = True)

out_weights = tf.Variable(tf.random_normal([num_units, n_classes]))
out_bias = tf.Variable(tf.random_normal([n_classes]))
x = tf.placeholder(dtype = tf.float32, shape = [None, time_steps, n_input])
y = tf.placeholder(dtype = tf.float32, shape = [None, n_classes])

input = tf.unstack(x, time_steps, 1)
lstm_layer = rnn.BasicLSTMCell(num_units, forget_bias = 1)
outputs, _ = rnn.static_rnn(lstm_layer, input, dtype = tf.float32)
prediction = tf.matmul(outputs[-1], out_weights) + out_bias
loss = tf.reduce_mean(tf.nn.softmax_cross_entropy_with_logits(logits=prediction, labels=y))
opt = tf.train.AdamOptimizer(learning_rate).minimize(loss)

accuracy = tf.reduce_mean(tf.cast(tf.equal(tf.argmax(prediction, 1), tf.argmax(y, 1)), tf.float32))
```

    Extracting ./MNIST/train-images-idx3-ubyte.gz
    Extracting ./MNIST/train-labels-idx1-ubyte.gz
    Extracting ./MNIST/t10k-images-idx3-ubyte.gz
    Extracting ./MNIST/t10k-labels-idx1-ubyte.gz
    


```python
with tf.Session() as sess:
    tf.global_variables_initializer().run()
    for i in range(2000):
        batch_x, batch_y = mnist.train.next_batch(batch_size = batch_size)
        batch_x = batch_x.reshape((batch_size, time_steps, n_input))
        sess.run(opt, feed_dict = {x: batch_x, y: batch_y})
        if (i + 1) % 100 == 0:
            loss_, acc_ = sess.run([loss, accuracy], feed_dict = {x: batch_x, y: batch_y})
            print("iter %d: loss = %.4f, accuracy = %.4f." % (i + 1, loss_, acc_))
    test_x, test_y = mnist.test.images.reshape((-1, time_steps, n_input)), mnist.test.labels
    loss_, acc_ = sess.run([loss, accuracy], feed_dict = {x: test_x, y: test_y})
    print("final test loss: %.4f, accuracy: %.4f" % (loss_, acc_))
```

    iter 100: loss = 0.5025, accuracy = 0.8594.
    iter 200: loss = 0.2134, accuracy = 0.9219.
    iter 300: loss = 0.2078, accuracy = 0.9297.
    iter 400: loss = 0.1500, accuracy = 0.9531.
    iter 500: loss = 0.0706, accuracy = 0.9688.
    iter 600: loss = 0.0688, accuracy = 0.9766.
    iter 700: loss = 0.2038, accuracy = 0.9531.
    iter 800: loss = 0.0271, accuracy = 0.9922.
    iter 900: loss = 0.0706, accuracy = 0.9922.
    iter 1000: loss = 0.1382, accuracy = 0.9531.
    iter 1100: loss = 0.0720, accuracy = 0.9844.
    iter 1200: loss = 0.0620, accuracy = 0.9688.
    iter 1300: loss = 0.0274, accuracy = 0.9844.
    iter 1400: loss = 0.0492, accuracy = 0.9922.
    iter 1500: loss = 0.0501, accuracy = 0.9766.
    iter 1600: loss = 0.0213, accuracy = 0.9922.
    iter 1700: loss = 0.0346, accuracy = 0.9922.
    iter 1800: loss = 0.0509, accuracy = 0.9766.
    iter 1900: loss = 0.0274, accuracy = 0.9922.
    iter 2000: loss = 0.0123, accuracy = 1.0000.
    final test loss: 0.0657, accuracy: 0.9786
    
