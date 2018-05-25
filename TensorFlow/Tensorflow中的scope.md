```python
with tf.variable_scope("foo"):
  with tf.variable_scope("bar"):
      v = tf.get_variable("v", [1])
      assert v.name == "foo/bar/v:0"
```

Basic example of sharing a variable AUTO_REUSE:

```python
def foo():
with tf.variable_scope("foo", reuse=tf.AUTO_REUSE):
  v = tf.get_variable("v", [1])
return v

v1 = foo()  # Creates v.
v2 = foo()  # Gets the same, existing v.
assert v1 == v2
```

Basic example of sharing a variable with reuse=True:

```python
with tf.variable_scope("foo"):
  v = tf.get_variable("v", [1])
with tf.variable_scope("foo", reuse=True):
  v1 = tf.get_variable("v", [1])
assert v1 == v
```

Sharing a variable by capturing a scope and setting reuse:

```python
with tf.variable_scope("foo") as scope:
  v = tf.get_variable("v", [1])
  scope.reuse_variables()
  v1 = tf.get_variable("v", [1])
assert v1 == v
```

To prevent accidental sharing of variables, we raise an exception when getting
an existing variable in a non-reusing scope.

```python
with tf.variable_scope("foo"):
  v = tf.get_variable("v", [1])
  v1 = tf.get_variable("v", [1])
  #  Raises ValueError("... v already exists ...").
```

Similarly, we raise an exception when trying to get a variable that does not
exist in reuse mode.

```python
with tf.variable_scope("foo", reuse=True):
  v = tf.get_variable("v", [1])
  #  Raises ValueError("... v does not exists ...").
```

Note that the `reuse` flag is inherited: if we open a reusing scope, then all
its sub-scopes become reusing as well.

与name_scope的对比:
```
with tf.variable_scope("varscope", reuse=tf.AUTO_REUSE) as scope:
    v1 = tf.get_variable("v", [7.0])
    x1 = tf.Variable(5., name = "x")
with tf.name_scope("namescope") as scope:
    v2 = tf.get_variable("v", [7.0])
    x2 = tf.Variable(5., name = "x")
```
v1.name, x1.name: ('varscope/v:0', 'varscope/x:0')
v2.name, x2.name: ('v:0', 'namescope/x:0')