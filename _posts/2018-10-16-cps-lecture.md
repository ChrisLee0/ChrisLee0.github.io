---
title:  CPS讲义
tags: PL
---


### CPS讲义

没有任何出版的书把这个主题讲明白了(包括Dan的书)

表达式``(f (g (h i) j) k)``哪个部分先被求值？``(h i)``，因为它必须在``(g (h i) j)``应用之前被求值。

那么``(f (g (h i) (j k)))``呢？Scheme没有指定参数求值顺序，所以``(h i)``和``(j k)``都可以先求值。

所以，我们自己决断。``(h i (lambda (hi) ...))`` 假设hi是``(h i)``应用的结果，那么，我们替换掉在``...``中任何能被应用的东西：

``(f (g (h i) (j l)))``变成``(h i (lambda (hi) (f (g hi (j l)))))``

``(lambda (hi) (f (g hi (j l))))``是一个延续，``hi``只在延续体中出现一次，因为``hi``将只替换``(h i)``。

让我们用CPS风格写个rember。首先，这是原始风格：

```scheme
(define rember8
  (lambda (ls)
    (cond
      [(null? ls) '()][(= (car ls) 8) (cdr ls)]
      [else (cons (car ls) (rember8 (cdr ls)))])))
```

第一条规则：任何时候，我们看到代码中的lambda，都必须增加一个参数，然后处理函数体：

``(lambda (x ...) ...) => (lambda (x ... k) ...^)``

我们先在外层lambda增加一个参数``k``：

```scheme
(define rember8
  (lambda (ls k)
    (cond
      [(null? ls) '()]
      [(= (car ls) 8) (cdr ls)]
      [else (cons (car ls) (rember8 (cdr ls)))])))
```

现在，来处理剩下的代码，我们得先介绍一条新的规则。

第二条规则：不要在意这些细节。

小细节就是我们知道会马上终结的部分。

如果我们知道它会被求值，就不要处理它。

如果它**可能**会被求值，不要处理它，而应该传递给``k``。

在第一行cond代码，``(null? ls)``是一个很好的例子。我们知道它会被求值，并且我们知道它是小细节，所以我们不必为它操心。

那么作为答案返回的那个``'()``怎么办？第二条规则其余部分是，如果一个它可能会被求值，那么只需要传递给``k``。

应用第二条规则在第一行cond后，我们得到：

```scheme
(define rember8
  (lambda (ls k)
    (cond
      [(null? ls) (k '())]
      [(= (car ls) 8) (cdr ls)]
      [else (cons (car ls) (rember8 (cdr ls)))])))
```

在第二行cond代码的判断条件和返回值中，都有小细节，所以我们可以像第一行代码一样对待它。

```scheme
(define rember8
  (lambda (ls k)
    (cond
      [(null? ls) (k '())]
      [(= (car ls) 8) (k (cdr ls))]
      [else (cons (car ls) (rember8 (cdr ls)))])))
```

在else分支，小细节**不要**作为返回值，所以我们必须创建一个新的延续：

```scheme
(define rember8
  (lambda (ls k)
    (cond
      [(null? ls) (k '())]
      [(= (car ls) 8) (k (cdr ls))]
      [else (rember8 (cdr ls) (lambda (x) (cons (car ls) x)))])))
```

我们还没有全部完成，因为我们还有一些小细节在延续体内。所以，我们只需要传递给``k``：
```scheme
(define rember8
  (lambda (ls k)
    (cond
      [(null? ls) (k '())]
      [(= (car ls) 8) (k (cdr ls))]
      [else (rember8 (cdr ls) (lambda (x) (k (cons (car ls) x))))])))
```

这是完全的CPS风格，但是，我们如何调用它？别忘了，我们需要把``k``传递进去。
因为``(rember8 '() k)``应该返回``'()``，``k``应该是个恒等函数``(lambda (x) x)``：
```scheme
> (rember8 '(1 2 8 3 4 6 7 8 5) (lambda (x) x))
```

我们能看出到这个程序的什么性质？

首先，所有不是小细节的部分都是尾递归。程序里所有的尾递归都被星号标记：
```scheme
(define rember8
  (lambda (ls k)
    (cond
      [(null? ls) (*k* '())]
      [(= (car ls) 8) (*k* (cdr ls))]
      [else (*rember8* (cdr ls) (lambda (x) (*k* (cons (car ls) x))))])))
```

为什么``null?``,`` =``,`` car``, ``cdr``和 ``cons`` 不是？因为他们是小细节，即使我们把小细节组合，仍然是微不足道的。


其次，所有的参数都是小细节。没错，即使是在else分支的lambda，因为lambda**永远**都是小细节。

注意，本质上这是一个C程序。所有我们需要做的都是把延续转换成数据结构(记住我们是如何用闭包完成同样的事)。


我们来追踪一下``(rember8  (lambda (x) x))``

```scheme
ls | k
'(1 2 8 3 4 6 7 8 5) | (lambda (x) x) = id
'(2 8 3 4 6 7 8 5)   | (lambda (x) (id (cons 1 x))) = k2
'(8 3 4 6 7 8 5)     | (lambda (x) (k2 (cons 2 x))) = k3
```

一旦我们命中8，我们应用``(k (cdr ls))``，此时``k``是``k3``，而``ls``是``'(8 3 4 6 7 8 5)``
```scheme
(k3 '(3 4 6 7 8 5)) = (k2 (cons 2 '(3 4 6 7 8 5)))
(k2 '(2 3 4 6 7 8 5)) = (id (cons 1 '(2 3 4 6 7 8 5)))
(id '(1 2 3 4 6 7 8 5)) = '(1 2 3 4 6 7 8 5)
```
现在，我们完成了。

我们来尝试一个更复杂的程序，``multirember8``。不仅仅是移除第一个8，而是移除所有的8.
```scheme
(define multirember8
  (lambda (ls)
    (cond
      [(null? ls) '()]
      [(= (car ls) 8) (multirember8 (cdr ls))]
      [else (cons (car ls) (multirember8 (cdr ls)))])))
```

现在，我们回到CPS风格的``rember8``进行CPS转换：
```scheme
(define multirember8
  (lambda (ls k)
    (cond
      [(null? ls) (k '())]
      [(= (car ls) 8) (multirember8 (cdr ls))] ;; uh-oh!
      [else (multirember8 (cdr ls) (lambda (x) (k (cons (car ls) x))))])))
```

我们需要在第二行做些什么？因为``multirember8``需要2个参数，我们现在需要传递一个延续。
```scheme
(define multirember8
  (lambda (ls k)
    (cond
      [(null? ls) (k '())]
      [(= (car ls) 8) (multirember8 (cdr ls) (lambda (x) (k x)))]
      [else (multirember8 (cdr ls) (lambda (x) (k (cons (car ls) x))))])))
```

但是，``(lambda (x) (k x))``做了什么？它接受任何传递给它的参数，然后传递给``k``。因此，整个表达式，等同于``k``。


η约简：如果``x``在``M``中不是自由变量，并且保证``M``会终结，那么``(lambda (x) (M x)) = M``。
``M``是满足这些规则的任意表达式。它不必是像``k``一样的单变量。

所以，任何时候看到尾递归，你都不需要想到η约简，只需要把它传递给k。

```scheme
(define multirember8
  (lambda (ls k)
    (cond
      [(null? ls) (k '())][(= (car ls) 8) (multirember8 (cdr ls) k)]
      [else (multirember8 (cdr ls) (lambda (x) (k (cons (car ls) x))))])))
```

### 中英对照

CPS Lecture
CPS讲义

No published books do this subject justice (including Dan's!)
没有任何出版的书把这个主题讲明白了(包括Dan的书)

Which part of ``(f (g (h i) j) k)`` can be done first? ``(h i)``, since it must be evaluated before ``(g (h i) j)`` can be applied.
表达式``(f (g (h i) j) k)``哪个部分先被求值？``(h i)``，因为它必须在``(g (h i) j)``应用之前被求值。

What about ``(f (g (h i) (j k)))``? Scheme doesn't specify the order in which arguments are evaluated so it could be either ``(h i)`` or ``(j k)``.
那么``(f (g (h i) (j k)))``呢？Scheme没有指定参数求值顺序，所以``(h i)``和``(j k)``都可以先求值。

So, let's take control. ``(h i (lambda (hi) ...))`` We assume that hi is the result of applying ``(h i)``. Then, we drop in everything else that has to be done to replace the ``...``:
所以，我们自己决断。``(h i (lambda (hi) ...))`` 假设hi是``(h i)``应用的结果，那么，我们替换掉在``...``中任何能被应用的东西：

``(f (g (h i) (j l)))``becomes ``(h i (lambda (hi) (f (g hi (j l)))))``
``(f (g (h i) (j l)))``变成``(h i (lambda (hi) (f (g hi (j l)))))``

``(lambda (hi) (f (g hi (j l))))`` is a continuation. ``hi`` only appears once in the body of the continuation, because ``hi`` is intended to replace ``(h i)`` and only ``(h i)``.
``(lambda (hi) (f (g hi (j l))))``是一个延续，``hi``只在延续体中出现一次，因为``hi``将只替换``(h i)``。

Let's write rember in CPS. First, the direct style:
让我们用CPS风格写个rember。首先，这是原始风格：

```scheme
(define rember8
  (lambda (ls)
    (cond
      [(null? ls) '()][(= (car ls) 8) (cdr ls)]
      [else (cons (car ls) (rember8 (cdr ls)))])))
```

First rule: whenever we see a lambda in the code we want to CPS, we have to add an argument, and then process the body:
第一条规则：任何时候，我们看到代码中的lambda，都必须增加一个参数，然后处理函数体：

``(lambda (x ...) ...) => (lambda (x ... k) ...^)``

Let's start by adding a ``k`` to the outer lambda:
我们先在外层lambda增加一个参数``k``：

```scheme
(define rember8
  (lambda (ls k)
    (cond
      [(null? ls) '()]
      [(= (car ls) 8) (cdr ls)]
      [else (cons (car ls) (rember8 (cdr ls)))])))
```

Now, to handle the rest of the program, we have to introduce a new rule.
现在，来处理剩下的代码，我们得先介绍一条新的规则。

Second rule: "Don't sweat the small stuff!"
第二条规则：不要在意这些细节！

Small stuff is stuff we know will terminate right away.
小细节就是我们知道会马上终结的部分。

Don't sweat the small stuff if we know it will be evaluated.
如果我们知道它会被求值，就不要处理它。

Don't sweat the small stuff if it *might* be evaluated, but instead pass it to ``k``.
如果它可能会被求值，不要处理它，而应该传递给``k``。

A good example of the first is ``(null? ls)`` in the first cond line. We know it will be evaluated, and we know it's small stuff, so we don't have to worry about it.
在第一行cond代码，``(null? ls)``是一个很好的例子。我们知道它会被求值，并且我们知道它是小细节，所以我们不必为它操心。

What about the ``'()`` that's returned as an answer? The other part of the second rule is that if some small stuff *might* be evaluated, we just pass it to ``k``.
那么作为答案返回的那个``'()``怎么办？第二条规则其余部分是，如果一个它可能会被求值，那么只需要传递给``k``。

After applying the second rule to the first cond line, we get:
应用第二条规则在第一行cond后，我们得到：

```scheme
(define rember8
  (lambda (ls k)
    (cond
      [(null? ls) (k '())]
      [(= (car ls) 8) (cdr ls)]
      [else (cons (car ls) (rember8 (cdr ls)))])))
```

The second cond line also has small stuff in both the test and the return value, so we can treat it just like the first line.
在第二行cond代码的判断条件和返回值中，都有小细节，所以我们可以像第一行代码一样对待它。

```scheme
(define rember8
  (lambda (ls k)
    (cond
      [(null? ls) (k '())]
      [(= (car ls) 8) (k (cdr ls))]
      [else (cons (car ls) (rember8 (cdr ls)))])))
```
The else case, however, does *not* have small stuff as a return value, so we have to build a new continuation:
在else分支，小细节**不要**作为返回值，所以我们必须创建一个新的延续：

```scheme
(define rember8
  (lambda (ls k)
    (cond
      [(null? ls) (k '())]
      [(= (car ls) 8) (k (cdr ls))]
      [else (rember8 (cdr ls) (lambda (x) (cons (car ls) x)))])))
```

We're not quite done, though, since we now have small stuff in the body of the continuation. So, we just pass it to ``k``:
我们还没有全部完成，因为我们还有一些小细节在延续体内。所以，我们只需要传递给``k``：
```scheme
(define rember8
  (lambda (ls k)
    (cond
      [(null? ls) (k '())]
      [(= (car ls) 8) (k (cdr ls))]
      [else (rember8 (cdr ls) (lambda (x) (k (cons (car ls) x))))])))
```
This is now completely CPSed, but how do we invoke it? After all, we need a ``k`` to pass in. Since ``(rember8 '() k)`` should be ``'()``,`` k`` can be the identity function ``(lambda (x) x)``:
这是完全的CPS风格，但是，我们如何调用它？别忘了，我们需要把``k``传递进去。
因为``(rember8 '() k)``应该返回``'()``，``k``应该是个恒等函数``(lambda (x) x)``：

```scheme
> (rember8 '(1 2 8 3 4 6 7 8 5) (lambda (x) x))
```

What properties can we observe about this program?
我们能看出到这个程序的什么性质？

First, all non-small stuff calls are tail calls. Here's the program with the tail calls surrounded by asterisks:
首先，所有不是小细节的部分都是尾递归。程序里所有的尾递归都被星号标记：
```scheme
(define rember8
  (lambda (ls k)
    (cond
      [(null? ls) (*k* '())]
      [(= (car ls) 8) (*k* (cdr ls))]
      [else (*rember8* (cdr ls) (lambda (x) (*k* (cons (car ls) x))))])))
```
Why don't ``null?``,`` =``,`` car``,`` cdr``, and ``cons`` count? Because they're just small stuff, and when we combine small stuff together in small ways, the combination remains small.
为什么``null?``,`` =``,`` car``, ``cdr``和 ``cons`` 不是？因为他们是小细节，即使我们把小细节组合，仍然是微不足道的。

Second, all arguments are small stuff. Yep, even the lambda in the else line, because lambda is *always* small stuff.
其次，所有的参数都是小细节。没错，即使是在else分支的lambda，因为lambda**永远**都是小细节。

Notice that this is essentially a C program. All we have to do is convert the continuations to data structures (remember how we did the same thing with closures).
注意，本质上这是一个C程序。所有我们需要做的都是把延续转换成数据结构(记住我们是如何用闭包完成同样的事)。

Let's trace ``(rember8  (lambda (x) x))``
我们来追踪一下``(rember8  (lambda (x) x))``

```scheme
ls | k
'(1 2 8 3 4 6 7 8 5) | (lambda (x) x) = id
'(2 8 3 4 6 7 8 5)   | (lambda (x) (id (cons 1 x))) = k2
'(8 3 4 6 7 8 5)     | (lambda (x) (k2 (cons 2 x))) = k3
```
Once we hit the 8, we apply ``(k (cdr ls))`` where ``k`` is ``k3`` and ``ls`` is ``'(8 3 4 6 7 8 5)``
一旦我们命中8，我们应用``(k (cdr ls))``，此时``k``是``k3``，而``ls``是``'(8 3 4 6 7 8 5)``
```scheme
(k3 '(3 4 6 7 8 5)) = (k2 (cons 2 '(3 4 6 7 8 5)))
(k2 '(2 3 4 6 7 8 5)) = (id (cons 1 '(2 3 4 6 7 8 5)))
(id '(1 2 3 4 6 7 8 5)) = '(1 2 3 4 6 7 8 5)
```
And we're done.
现在，我们完成了。

Let's try a more complicated program, ``multirember8``. Instead of just removing the first 8, it'll remove all of the 8s.
我们来尝试一个更复杂的程序，``multirember8``。不仅仅是移除第一个8，而是移除所有的8。
```scheme
(define multirember8
  (lambda (ls)
    (cond
      [(null? ls) '()]
      [(= (car ls) 8) (multirember8 (cdr ls))]
      [else (cons (car ls) (multirember8 (cdr ls)))])))
```
Now, let's start CPSing by going back to our CPSed ``rember8``:
现在，我们回到CPS风格的``rember8``进行CPS转换：
```scheme
(define multirember8
  (lambda (ls k)
    (cond
      [(null? ls) (k '())]
      [(= (car ls) 8) (multirember8 (cdr ls))] ;; uh-oh!
      [else (multirember8 (cdr ls) (lambda (x) (k (cons (car ls) x))))])))
```
What do we need to do for the second line? Since ``multirember8`` takes two arguments, we need to now pass it a continuation.
我们需要在第二行做些什么？因为``multirember8``需要2个参数，我们现在需要传递一个延续。
```scheme
(define multirember8
  (lambda (ls k)
    (cond
      [(null? ls) (k '())]
      [(= (car ls) 8) (multirember8 (cdr ls) (lambda (x) (k x)))]
      [else (multirember8 (cdr ls) (lambda (x) (k (cons (car ls) x))))])))
```
But what's ``(lambda (x) (k x))`` doing? It's taking whatever is passed to it, and passing it to ``k``. This whole expression, therefore, is equivalent to ``k``.
但是，``(lambda (x) (k x))``做了什么？它接受任何传递给它的参数，然后传递给``k``。因此，整个表达式，等同于``k``。

Eta reduction: ``(lambda (x) (M x)) = M`` if ``x`` is not free in ``M`` and ``M`` is guaranteed to terminate. ``M`` is any arbitrary expression that satisfies these rules; it doesn't have to be only a single variable like ``k``.
η约简：如果``x``在``M``中不是自由变量，并且保证``M``会终结，那么``(lambda (x) (M x)) = M``。
``M``是满足这些规则的任意表达式。它不必是像``k``一样的单变量。

So, whenever you see a tail call, you don't even have to think about eta. Just pass ``k`` to it.
所以，任何时候看到尾递归，你都不需要想到η约简，只需要把它传递给k。

```scheme
(define multirember8
  (lambda (ls k)
    (cond
      [(null? ls) (k '())][(= (car ls) 8) (multirember8 (cdr ls) k)]
      [else (multirember8 (cdr ls) (lambda (x) (k (cons (car ls) x))))])))
```