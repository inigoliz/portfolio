---
date: 2024-12-26T03:51:29+01:00
draft: false
title: Teenygrad Study Notes
slug: teenygrad-learning-notes
---
![Title image](/portfolio/images/teenygrad-learning-notes/cover.png "700px")
<!-- ![Title image](/portfolio/images/teenygrad-learning-notes/big_graph.svg "700px") -->

What happens when you reduce a ML framework all the way to its bare bones? - You get [**teenygrad**](https://github.com/tinygrad/teenygrad).

In this post, I want to dive into the source code of teenygrad to illustrate the software architecture behind some important concepts of ML-engines like backpropagation and computational graphs.

It's true that you can go read the code directly (it's not that difficult) but... why not take a guided tour first? If you enjoyed Karpathy's [micrograd](https://www.youtube.com/watch?v=VMj-3S1tku0), you'll like teenygrad.

## What is Teenygrad?
It is a minimalistic ML framework. It does not aim to be production-grade but to illustrate the core ideas behind ML frameworks: **tensor arithmetics**, **computational graphs** and **automatic differentiation**.

**teenygrad** has a big brother: [**tinygrad**](). They both build on top of the same philosophy, but tinygrad aims to be production-grade: it includes computation graph optimizations, JIT, lazy computation and device-specific kernel generation.

**teenygrad** also has a smaller brother: Karpathy's [micrograd](https://github.com/karpathy/micrograd), the differences being that micrograd is value-based while teenygrad is array-based, and that teenygrad implements a more pulished software architecture, which is what I want to discuss in this post.

## Tensors Arithmetics
The core data structure in teenygrad is the `Tensor` object. A tensor is like a Numpy `ndarray`. Like in other frameworks, tensors have `.shape`, `.dtype` and `.device` properties.

```python
from teenygrad.tensor import Tensor

x = Tensor.ones((1, 3, 16, 16))
print(x.shape) # (1, 3, 16, 16)
print(x.dtype) # dtypes.float
print(x.device) # 'CPU'
```

Tensors can be operated through regular arithmetics:

```python
a = Tensor([[1, 2], [3, 4]])
b = Tensor([[5, 6], [7, 8]])

(a + b).numpy()
# array([[ 6., 8.], [10., 12.]], dtype=float32)

(a * b).numpy()
# array([[ 5., 12.], [21., 32.]], dtype=float32)

(a @ b).numpy()
# array([[19., 22.], [43., 50.]], dtype=float32)
```

More complex operations are also available:

```python
x = Tensor.ones(2,)
w = Tensor.ones(2, 3)
b = Tensor.ones(3)

y = x.linear(weight=w, bias=b)
# array([3., 3., 3.], dtype=float32)
```

Tensors may also store gradients in their `.grad` property after a backward pass is run:
```python
a = Tensor.rand((2, 3), requires_grad=True)
y = a.sum()
y.backward()

a.grad.numpy()
# array([[1., 1., 1.],
#        [1., 1., 1.]], dtype=float32)
```

## The Computational Graph
Operations on tensors record a **Computational Graph**. The computational graph holds the information used by the autograd engine to run the backward pass and compute the gradients of each tensor involved.

In particular, each tensor holds a **context** property `._ctx` that refers to the math operation that created the tensor:

```python
a = Tensor.ones((2), requires_grad=True)
b = Tensor.ones((2), requires_grad=True)
c = a + b

print(c._ctx)
# <teenygrad.mlops.Add object at 0x10a88d2a0>
```

There are a couple of things to mention about the previous code:
- Tensors without `requires_grad=True` are not added to the computational graph.
- The `_ctx` holds an instance of the `mlops.Add` object.

At this point I was wondering... what's the need of an instance of `mlops.Add` in `._ctx`? The answer is that, in teenygrad, operations themselves are stateful. This makes sense, since they need to store certain information in order to run the backward pass (we'll come back to this point later).

![Screenshot of the state of an Add instance](/portfolio/images/teenygrad-learning-notes/add_state.png#center "700px")

Operations store references to to the tensors that spawned the operation itself. These are called the `parents` of the operation:

```python
a = Tensor.ones((2), requires_grad=True)  # id(a): 4403513344
b = Tensor.ones((2), requires_grad=True)  # id(b): 4403663552
c = a + b

[id(p) for p in c._ctx.parents]  # [4403666432, 4403672192]
```

Recap so far:
- Tensors have a `._ctx` property that points to an instance of an operation.
- Operations have a `.parents` property that points to the tensors that created it (the inputs of the operation).

These links between tensors and operations, repeated, create a graph that describes the full computation. This is **The Computational Graph**.

### Visualizing Computational Graphs

Let's visualize some computational graphs:

I have put together the following code to visualize computational graphs. It's an adaptation of Karpathy's code in [micrograd/trace_graph.ipynb](https://github.com/karpathy/micrograd/blob/master/trace_graph.ipynb).

```python
from graphviz import Digraph

def trace(root):
    nodes, edges = set(), set()
    def build(v):
        if v not in nodes:
            nodes.add(v)
            if v._ctx:
                for child in v._ctx.parents:
                    edges.add((child, v))
                    build(child)
    build(root)
    return nodes, edges

def draw_graph(root, show_grads=False, format='svg', rankdir='LR'):
    """
    format: png | svg | ...
    rankdir: TB (top to bottom graph) | LR (left to right)
    """
    assert rankdir in ['LR', 'TB']
    nodes, edges = trace(root)
    dot = Digraph(format=format, graph_attr={'rankdir': rankdir}) #, node_attr={'rankdir': 'TB'})
    
    for n in nodes:
        grad_label = f"{n.grad.numpy().item():.2f}" if n.grad else "None"
        # dot.node(name=str(id(n)), label = "{ data %.4f | grad %.4f }" % (n.numpy(), n.grad() if n.grad() is not None else 0.0), shape='record')
        if show_grads:
            dot.node(name=str(id(n)), label = f"data: {n.numpy().item():.2f} | grad: {grad_label}", shape='record')
        else:
            dot.node(name=str(id(n)), label = f"data: {n.numpy().item():.2f}", shape='record')

        if n._ctx:
            dot.node(name=str(id(n)) + str(id(n._ctx)), label=n._ctx.__class__.__name__)
            dot.edge(str(id(n)) + str(id(n._ctx)), str(id(n)))
    
    for n1, n2 in edges:
        dot.edge(str(id(n1)), str(id(n2)) + str(id(n2._ctx)))
    
    return dot
```

The first visualization is the addition of two tensors. I'll use tensors with a scalar value for clarity in the visualization:

```python
a = Tensor([0.5], requires_grad=True)
b = Tensor([0.8], requires_grad=True)

c = a + b
draw_graph(c)
```

![Graph visualization of addition](/portfolio/images/teenygrad-learning-notes/graph_add.svg#center  "400px")

Next, a linear transformation:

```python
x = Tensor([1.0], requires_grad=True)
w = Tensor([0.5], requires_grad=True)
b = Tensor([0.8], requires_grad=True)

c = x * w + b
draw_graph(c)
```

![Graph visualization of linear](/portfolio/images/teenygrad-learning-notes/graph_linear.svg#center  "600px")

> **Note:** Python parses a composed operation following the conventional order of operations. `c = x * w + b` is broken into `hidden_tensor = x * w` and `c = hidden_tensor + b`. You can see the instance of `hidden_tensor` in the graph, even though we have not explicitely defined a variable for it.

Graphs can get as complex as one wants: tensors can be involved in several operations, there can be residual connections, etc. Note that the resulting computational graph is a DAG (Directed Acyclic graph).

```python
x1 = Tensor([1.0], requires_grad=True)
x2 = Tensor([2.0], requires_grad=True)
w = Tensor([0.5], requires_grad=True)
b = Tensor([0.8], requires_grad=True)

c = x1 * w + x2 * w + b + x2
draw_graph(c)
```

![Graph visualization of residual](/portfolio/images/teenygrad-learning-notes/graph_res.svg#center "800px")

## Diving Deeper: Meet the `LazyBuffer`
Since this is a post about the inner workings of teenygrad, we need to dive into the internals of the `Tensor` object. Right under `Tensor`, there is the `LazyBuffer` class, the object that ultimately holds *data* and operates on it. In fact, `Tensor` is just a convenient wrapper around `LazyBuffer`.

Actually, it's not the first time that we encounter a `LazyBuffer`. Before, in the image showcasing the attributes of `mlops.Add`, there was this one:

```python
_ctx.parents = (<Tensor <LB () dtypes.float> on CPU with grad None>, <Tensor <LB () dtypes.float> on CPU with grad None>)
```

See that `LB`? That's the `LazyBuffer` within the `Tensor`. We also see them if we print a `Tensor` instance. Rather than getting the data, we get the `LazyBuffer` underneath:

```python
a = Tensor([1.0])
print(a)

# <Tensor <LB (1,) dtypes.float> on CPU with grad None>
```

Let me paste the definition of the `LazyBuffer` class to make clear that it's there where the *actual data* is stored (in this case, as a Numpy `ndarray`):

```python
class LazyBuffer:
	device = "CPU"

	def __init__(self, buf: np.ndarray):
	  self._np = buf
	  
	@property
	def base(self): return self
	
	@property
	def dtype(self): return dtypes.from_np(self._np.dtype)
	
	@property
	def realized(self): return RawCPUBuffer(self._np)
	
	@property
	def shape(self): return self._np.shape
	...
```

Now, looking at the defintion of the `Tensor` class it's crystal clear that it is just a wrapper around a `LazyBuffer` object, stored in its `lazydata` property:

```python
class Tensor:
	...
	@property
	def device(self) -> str: return self.lazydata.device
	
	@property
	def shape(self) -> Tuple[sint, ...]: return self.lazydata.shape
	
	@property
	def dtype(self) -> DType: return self.lazydata.dtype
	...
```

At this point you may be wondering why we need tensors in the first place? Well, even if the `LazyBuffer` holds the data and operates on it, it does not know about computational graphs and backward passes. Those things happen one level of abstraction up, in the `Tensor` class.

Now that I've introduced the `LazyBuffer`, let's trace down what happens when two tensors are added:

```python
a = Tensor([1.], requires_grad=True)
b = Tensor([2.], requires_grad=True)

c = a+b
```

The first thing to note is that `a + b == a.__add__(b)`. In other words, our conventional math notation is just syntactic sugar for and `__add__()` method call on the `Tensor` class. We can take a look at what the `Tensor.__add__()` method looks like:

```python
def __add__(self, x) -> Tensor: return self.add(x)
```

One step deeper...

```python
def add(self, x:Union[Tensor, float], reverse=False) -> Tensor:
	x = self._to_float(x)
    return mlops.Add.apply(*self._broadcasted(x, reverse)) if x.__class__ is Tensor or x else self
```

The first time I looked at the `mlops.Add` definition I got quite confused. I was expecting an `.apply(self, x: Tensor) -> Tensor` method returning a new `Tensor`. However, that's not what it looks like:

```python
class Add(Function):
  def forward(self, x:LazyBuffer, y:LazyBuffer) -> LazyBuffer:
    return x.e(BinaryOps.ADD, y)

  def backward(self, grad_output:LazyBuffer) -> Tuple[Optional[LazyBuffer], Optional[LazyBuffer]]:
    return grad_output if self.needs_input_grad[0] else None, \
           grad_output if self.needs_input_grad[1] else None
```

What's going on then? Let's start by the `forward()` method, which seems to be like I was expecting: it takes a couple of `LazyBuffer` (the actual data) and returns a new one. The `x.e()` stands for *evaluate*:

```python
class LazyBuffer:
	...

	def e(self, op, *srcs:LazyBuffer):
		if DEBUG >= 1: print(op, self, srcs)
		if op == UnaryOps.NEG: ret = -self._np
		elif op == UnaryOps.EXP2: ret = np.exp2(self._np)
		elif op == UnaryOps.LOG2: ret = np.log2(self._np)
		elif op == UnaryOps.SIN: ret = np.sin(self._np)
		elif op == UnaryOps.SQRT: ret = np.sqrt(self._np)
		elif op == BinaryOps.ADD: ret = self._np + srcs[0]._np
		elif op == BinaryOps.SUB: ret = self._np - srcs[0]._np
		elif op == BinaryOps.MUL: ret = self._np * srcs[0]._np
		elif op == BinaryOps.DIV: ret = self._np / srcs[0]._np
		elif op == BinaryOps.MAX: ret = np.maximum(self._np, srcs[0]._np)
		elif op == BinaryOps.CMPLT: ret = self._np < srcs[0]._np
		
	...
```

Recap: `forward()` and `backward()` wrap calls to the `LazyBuffer.e()` method, which evaluates operations on the actual data (using Numpy as the backend).

The question still is... where is the `apply()` method?

The answer is in the function definition: `class Add(Function)`. All the operations in `mlops` (like `Add`) inherit from the `Function` parent class. I suspected that the `Function` class has an `apply()` method that executes the call to `.forward()`, and that is correct:

```python
class Function:
    def __init__(self, device:str, *tensors:Tensor):
        self.device = device
        self.needs_input_grad = [t.requires_grad for t in tensors]
        self.requires_grad = True if any(self.needs_input_grad) else None if None in self.needs_input_grad else False
        if self.requires_grad:
            self.parents = tensors

    def forward(self, *args, **kwargs): raise NotImplementedError(f"forward not implemented for {type(self)}")
    def backward(self, *args, **kwargs): raise RuntimeError(f"backward not implemented for {type(self)}")

    @classmethod
    def apply(fxn:Type[Function], *x:Tensor, **kwargs) -> Tensor:
        ctx = fxn(x[0].device, *x)
        ret = Tensor(ctx.forward(*[t.lazydata for t in x], **kwargs), device=ctx.device, requires_grad=ctx.requires_grad)
        if ctx.requires_grad and not Tensor.no_grad: ret._ctx = ctx    # used by autograd engine
        return ret
```

Quite a bit of code, so let's take it in steps.

See the `@classmethod`? That is what is called by `return mlops.Add.apply(...)`. One point that confused me for a while was the presence of `fxn` in `apply(fxn, ...)`, instead of the usual `self`, since the first argument is always a reference to the class itself. For a while I thought that it was some special syntax for class methods, but no. The truth is that, while its custom to name the first argument `self`, we can name it as we want. For even more clarity, I will rewrite the `apply()` as follows:

```python
 def apply(self:Type[Function], *x:Tensor, **kwargs) -> Tensor:
    ctx = self(x[0].device, *x)
    lb = ctx.forward(*[t.lazydata for t in x], **kwargs)
    ret = Tensor(lb, device=ctx.device, requires_grad=ctx.requires_grad)
    if ctx.requires_grad and not Tensor.no_grad:
        ret._ctx = ctx    # used by autograd engine
    return ret
```

It's not difficult to understand whats going on. First, `ctx = ...` calls the `__init__()` method of `Add`, which creates an instance of it and stores it in the *context* attribute. Then, the `.forward()` method of the `Add` class is called, which returns a `LazyBuffer`, which is then wrapped into a Tensor and assigned to `ret`. The `ret` tensor is then given a `._ctx`, which is the operation that created it with its information to perform backpropagation.

For example, looking at the definition of the `mlops.Mul` operation, we can see that it stores the input `LazyBuffer` in its context, since they are needed for the backward pass:

```python
class Mul(Function):
  def forward(self, x:LazyBuffer, y:LazyBuffer) -> LazyBuffer:
    self.x, self.y = x, y
    return x.e(BinaryOps.MUL, y)

  def backward(self, grad_output:LazyBuffer) -> Tuple[Optional[LazyBuffer], Optional[LazyBuffer]]:
    return self.y.e(BinaryOps.MUL, grad_output) if self.needs_input_grad[0] else None, \
           self.x.e(BinaryOps.MUL, grad_output) if self.needs_input_grad[1] else None
```

And that's it.

There is one last mystery to uncover, which is where is the call to `mlops.Add.backward()`. The answer is in the next section about **The Backward Pass**.

## The Backward Pass

If you come from PyTorch, you'll be used to calling `loss.backward()`, which runs the backward pass and stores in `.grad` the gradient of the loss with respect to each tensor. teenygrad works in the same way. 

The question is: How does the backward pass work?

Let's go back to a simple computation: the multiplication of two tensors:

```python
x = Tensor([1.0], requires_grad=True)
y = Tensor([2.0], requires_grad=True)

z = x * y
loss = z.sum()

loss.backward()
```

This is what the computational graph looks like before and after running `loss.backward()`:

![Before grad](/portfolio/images/teenygrad-learning-notes/beforegrad.svg "700px")

![After grad](/portfolio/images/teenygrad-learning-notes/aftergrad.svg "700px")

>**Note:** In order to plot the previous graph, I had to disable a line in teenygrad's source code (`tensor.py:259`) that deletes the `._ctx` once a tensor's grad is set:
>```python
>   def backward(self) -> Tensor:
>    ...
>      # del t0._ctx
>    return self
>```

The *backward* pass is called as such because it starts running from the last node in the computational graph and propagates the gradients backwards.

Looking at the graph from the **right to the left**:

- The gradient of the last tensor is 1.0 by construction (`dloss/dloss=1`).
- The `Reshape` operation just shuffles data around. Hence, the upstream gradient is just the downstream gradient but shuffled back.
- The `Sum` operation copies the downstream gradient to its inputs:
```bash
out = x + y -> dloss/dx = dloss/dout * dout/dx = dloss/dout * 1
```
- The `Mul` operation swaps its inputs and multiplies them by the downstream gradient:
```bash
out = x * y -> dloss/dx = dloss/dout * dout/dx = dloss/dout * y
```

### Running the Backward Pass Manually

Let's run the backward pass manually, step by step, starting from the last node and propagating the gradients back (from the right to the left). We start with a graph with empty gradients:

![Before grad](/portfolio/images/teenygrad-learning-notes/beforegrad.svg "700px")

Steps:
1. I'll set `loss.grad=Tensor(1)` manually.
2. To traverse back the `Reshape` operation I can use `loss._ctx.backward()` (remember that `loss._ctx` points to the instance of the `Reshape` operation which created the `loss` tensor). Then I can grab `loss._ctx.parents` and set their gradients manually:
```python
reshape_op = loss._ctx
reshape_parents = loss._ctx.parents  # tuple of a single element

grad_lazydata = reshape_op.backward(loss.grad.lazydata)  # takes lazydata and gives lazydata
reshape_parents[0].grad = Tensor(grad_lazydata)
```
![Before grad](/portfolio/images/teenygrad-learning-notes/graph_interm_1.svg "700px")

3. Traversing back the `Sum` operation is slightly more cumbersome, since we cannot direclty grab it, so we need to use `loss._ctx.parents[0]._ctx`. Again, `Sum` has a single parent. 
```python
sum_op = loss._ctx.parents[0]._ctx
sum_parents = loss._ctx.parents[0]._ctx.parents  # tuple of a single element

grad_lazydata = sum_op.backward(loss.grad.lazydata) 
sum_parents[0].grad = Tensor(grad_lazydata)
```

![Before grad](/portfolio/images/teenygrad-learning-notes/graph_interm_2.svg "700px")

4. Traversing back the `Mul` operation is easier. It has two parents, so `mul_op.backward()` will return two gradients.
```python
mul_op = z._ctx
mul_parents = z._ctx.parents  # tuple of a single element

grad_lazydata = mul_op.backward(z.grad.lazydata)  # tuple of 2 elements
mul_parents[0].grad = Tensor(grad_lazydata[0])
mul_parents[1].grad = Tensor(grad_lazydata[1])
```

![Before grad](/portfolio/images/teenygrad-learning-notes/graph_interm_3.svg#center "700px")

That was easy! It was always the same pattern:
- From a given `node`, grab `node._ctx` and call `node._ctx.backward(node.grad.lazydata)`.
- Grab `node._ctx.parents` and set their gradients.

Hopefully now the following extract from the teenygrad source code will not be daunting anymore:

```python
def backward(self) -> Tensor:
    assert self.shape == tuple(), f"backward can only be called for scalar tensors, but it has shape {self.shape})"

    # fill in the first grad with one. don't use Tensor.ones because we don't need contiguous
    # this is "implicit gradient creation"
    self.grad = Tensor(1, device=self.device, requires_grad=False)

    for t0 in reversed(self.deepwalk()):
      assert (t0.grad is not None)
      grads = t0._ctx.backward(t0.grad.lazydata)
      grads = [Tensor(g, device=self.device, requires_grad=False) if g is not None else None
        for g in ([grads] if len(t0._ctx.parents) == 1 else grads)]
      for t, g in zip(t0._ctx.parents, grads):
        if g is not None and t.requires_grad:
          assert g.shape == t.shape, f"grad shape must match tensor shape, {g.shape!r} != {t.shape!r}"
          t.grad = g if t.grad is None else (t.grad + g)
      del t0._ctx
    return self
```

There are a couple of details to note:
- Gradients are **accumulated** (notice `t.grad + g`): when a node participates in several operations, it's gradient comes from all of them.
- After a node is used, it's detached from the graph with `del t0._ctx`. I don't fully understand why.

The last question is... what is `self.deepwalk()`?

### Topological Sort

There is anotehr important ingredient in an autograd engine: an algorithm to organize the order in which each node should propagate its gradient back. This is important because a node must have its gradient set before being able to propagate to the previous ones. This algorithm is **Topological Sort**

```bash
Given a DAG, Topological Sort organizes the nodes into a list such that every node appears after all its dependencies.
```

Topological Sort is implemented in `def deepwalk()`. It's an standard algorithm so I'll not discuss it much:

```python
  def deepwalk(self):
    def _deepwalk(node, visited, nodes):
      visited.add(node)
      if getattr(node, "_ctx", None):
        for i in node._ctx.parents:
          if i not in visited: _deepwalk(i, visited, nodes)
        nodes.append(node)
      return nodes
    return _deepwalk(self, set(), [])
```

Instead, let's see it in action. For the following graph (where I'll use each node's data as its identifier, since they are all different):

![Before grad](/portfolio/images/teenygrad-learning-notes/toposort.svg#center "700px")

This is the output of `deepwalk()`, called on the last node:

```python
[2, 3, 4, 5, 6, 11, 13]
```

We just need to reverse the order ov the list, and that is the order in which we call the `.backward()` on each node.

```python
def backward(self) -> Tensor:
    ...
    for t0 in reversed(self.deepwalk()):
        ...
```

## Conclusion
These Study Notes do not aim to be an exhaustive discussion of teenygrad but to illustrate a few of its concepts. However, after reading these notes I hope approaching the source code of teenygrad to dig more becomes way easier. Also, having a clear picture of the code architecture of teenygrad is important to approach it's bigger (and more complex) brother [tinygrad](https://github.com/tinygrad/tinygrad).




