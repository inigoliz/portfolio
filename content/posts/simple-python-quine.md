---
date: '2025-06-17T09:26:01+01:00'
draft: false
title: 'Simple Python quine, step-by-step'
slug: simple-python-quine
cover:
  image: "/images/simple-python-quine/escher.png"
  alt: "Cover image"
  relative: false
---

![(Escher self reference)](/images/simple-python-quine/escher.png "700px")

## Self-replicating programs

Let's talk about self-replicating programs: A self-printing program, or *quine*, is a program that, when executed, prints it's own source code.

Why is this relevant? Think about DNA, humans or the [RepRap](https://www.wikiwand.com/en/articles/RepRap) movement.

One can find many quines after a quick Google. However, most of them seem rather obfuscated and difficult to understand as if the author had tried to obscure them to brag about their skills. They appeal the programmer's ego but do not provide much insight about how a sane human mind might have found them.

In this post, I'll motivate and build step-by-step a simple Python quine.

## My Python quine

It's rather short:

```python
a = 'a = %r\nprint(%r %% (a, a))'
print('a = %r\nprint(%r %% (a, a))' % (a, a))
```

This is what comes to the terminal when I run that code:

```bash
$ python quine.py
a = 'a = %r\nprint(%r %% (a, a))'
print('a = %r\nprint(%r %% (a, a))' % (a, a))
```

I can run it forever, in an endless loop of execution.

## Exploiting the syntax

Many quines grow around a certain syntax feature of a programming languge that they exploit.

In my case, the feature that I'll use is *string formatting* in Python:

```python
a = 'hola'
print('%r' % a)
```

which prints:

```bash
$ 'hola'
```

## Printing the print

Now we can start building the actual quine. The first thing I did was to print the first line of the program:

```python
a = 'hola'
print('a = %r' % a)
```

```bash
$ a = 'hola'
```

Then, I can also add the second line, with the `print()`:

```python
a = 'hola'
print('a = %r\nprint()' % a)
```

```bash
$ a = 'hola'
  print()
```

Last, how can we print the contents inside the `print()`? Easy: we can use `a` to hold the string inside the `print()`.

The following snippet is a bit *mind-bending*, I know. But it's not difficult to understand if you **stare at it** for a little while:

```python
a = 'a = %r\nprint(%r)'
print('a = %r\nprint(%r)' % (a, a))
```

```bash
a = 'a = %r\nprint(%r)'
print('a = %r\nprint(%r)')
```

We're not done, but at least we're able to print *some* of the contents inside the `print()` statement.

## Breaking cycles

We're quite close, but we're missing the last part of the string: the `% (a,a)`.
My first instinct would be to just add it to the contents of `a`.

However, that doesn't work:

```python
a = 'a = %r\nprint(%r) % (a, a)'
print('a = %r\nprint(%r)' % (a, a))
```

```bash
$ a = 'a = %r\nprint(%r) % (a, a)'
  print('a = %r\nprint(%r) % (a, a)')
  #                                ^ misplaced quote
```

We're facing a common challenge when building quines: how to print quotes (`'`). If you do `print(" 'hola' ")`, then how do you print the `"` quotes now?

Building quines is about breaking recursion cycles:

```
print('hola') -> print("print('hola')") -> ?
```

Some people address this challenge with a nifty trick: printing the quotes using their ASCII character:

```python
print(chr(39) + chr(104) + chr(111) + chr(108) + chr(97) + chr(39))
```

```bash
'hola'
```

However, I'd not like to depend on ASCII for my quine to work.

Instead, I'll use a different technique. Actually, I've been using it since the beginning of this post: remember when I said I would use *string formatting* as my core *syntactic trick*? Well, this is why: it allows me to print quotes without having to write the quotes explicitly in the print.

How can we print the following *string*?

```bash
$ print('hola')
```

I can use the following *source code*

```python
a = 'hola'
print('print(%r)' % (a))
```

```bash
$ print('hola')
```

See where this is going? Using *string formatting* allows to print quotes *wihtout* including the quotes *to be printed* in the print statement.

## Last details

Now, back to the quine. We had:


```python
a = 'a = %r\nprint(%r) % (a, a)'
print('a = %r\nprint(%r)' % (a, a))
```

Which printed, erroneously:

```bash
$ a = 'a = %r\nprint(%r) % (a, a)'
  print('a = %r\nprint(%r) % (a, a)')
  #                                ^ misplaced quote
```

What if we include the `% (a, a)` in the `print()`? We must escape the `%` with `%%`:

```python
a = 'a = %r\nprint(%r)'
print('a = %r\nprint(%r %% (a, a))' % (a, a))
```

```bash
$ a = 'a = %r\nprint(%r)'
  print('a = %r\nprint(%r)' % (a, a))
#                     ^ I must place %% (a, a) after here
```

Almost there. One last fix...

```python
a = 'a = %r\nprint(%r) %% (a, a)'
print('a = %r\nprint(%r %% (a, a))' % (a, a))
```

And there it is. The **quine**:

```bash
$ a = 'a = %r\nprint(%r %% (a, a))'
  print('a = %r\nprint(%r) %% (a, a)' % (a, a))
```

## Quines and DNA

Let's think about DNA whit an information processing mindset:
DNA can be thought as a program such as, upon being run, preforms certain actions and self-replicates.

It's not trivial how a program might do that. It's not even trivial whether that might even be possible. However, I hope that by this point of the blog post it's more or less makes sense that such a program can exist.

In fact, with a rather small modifyication of the Python quine, I can allow the progam to perform actions besides self replicating.

The following Python quine can be thought as an analogy of what DNA does:

```python
a = 'a = %r\ndo_other_stuff()\nprint(%r %% (a, a))'
do_other_stuff()
print('a = %r\ndo_other_stuff()\nprint(%r %% (a, a))' % (a, a))
```

Upon being run, the code executes `do_other_stuff()` which may have side-effects, and finally it prints itself (or copies itself to a new cell).

(Sorry for my loose precision when writing about biology concepts 😅)

## Wrapping up

I know, it takes a while to wrap one's head around the syntax acrobatics.

One methapore I read somewhere and helped me understand quines better is the following:

Imagine you had an instruction called `DUP` that duplicates the contents passed to it and prints them as an string.
Example:

```
DUP HOLA   --execute-->   HOLA HOLA
```

What happens if we give `DUP` as an argument?

```
DUP DUP  --execute-->  DUP DUP  --execute-->  DUP DUP --> ...
```

See? We have a loop. Or a cycle. Or a *fixed point* of the *execution*.

Self-replicating stuff is great!