---
title: 'Memoization'
date: 2012-05-30

tags: [Python, Racket]
---

最近在《JavaScript The Good Parts》和 [Udacity 的 CS212 这门课](http://www.udacity.com/view#Course/cs212/CourseRev/apr2012) 里都遇到了 memoization 这个概念。所以打算记下来方便以后参考。

个人理解，memoization 就是一种 cache，用来减少耗时（资源）的程序段的运行次数。以一个 Python 实现的计算 Fibonacci 数列的程序为例：

```python
def fib(n):
    if n < 2:
        return n
    else:
        return fib(n-2) + fib(n-1)
```

重复的计算非常多。如果运用 memoization，我们可以将每次的计算结果存在一个 cache 里，这样每次计算前先查看 cache，如果有需要的值，直接取，没有再计算：

```python
def gen_fib():
    cache = {0:0, 1:1}
    def fib_memo(n):
        if n not in cache:
            cache[n] = fib_memo(n-2) + fib_memo(n-1)
        return cache[n]
    return fib_memo
```

这里用了 closure 在外层函数持久保存一个 cache。可以用 python 的 time 模块提供的函数来比较一下前后两个版本的 fib 消耗的时间：

```python
import time

def timedcall(fn, *args):
    "Call function with args; return the time in seconds and result."
    t0 = time.clock()
    result = fn(*args)
    t1 = time.clock()
    return t1-t0, result

def fib(n):
    if n < 2:
        return n
    else:
        return fib(n-2) + fib(n-1)

def gen_fib():
    cache = {0:0, 1:1}
    def fib_memo(n):
        if n not in cache:
            cache[n] = fib_memo(n-2) + fib_memo(n-1)
        return cache[n]
    return fib_memo

print "fib: " + str(timedcall(fib, 40))
fib_memo = gen_fib()
print "fib_memo: " + str(timedcall(fib_memo, 40))
```

我这里执行之后得到的结果是：

```python
fib: (108.5130336397503, 102334155)
fib_memo: (4.833016485861208e-05, 102334155)
```

时间相差了足有 7 个数量级。
最后这段代码可以作为一个 decorator 来为任何一个函数实现 memoization：

```python
def memo(f):
    """Decorator that caches the return value for each call to f(args).
    Then when called again with same args, we can just look it up."""
    cache = {}
    def _f(*args):
        try:
            return cache[args]
        except KeyError:
            result = f(*args)
            cache[args] = result
            return result
        except TypeError:
            # some element of args can't be a dict key
            return f(*args)
    _f.cache = cache
    return _f

#Take fib for an example
def fib(n):
    if n < 2:
        return n
    else:
        return fib(n-2) + fib(n-1)
fib = memo(fib)

#or

@memo
def fib(n):
    if n < 2:
        return n
    else:
        return fib(n-2) + fib(n-1)
```

下面是 Racket 的一个实现：

```lisp
(define (make-cached-function func n)
  (letrec ([cache (make-vector n #f)]
           [next-to-replace 0]
           [vector-assoc
            (lambda (arg-lst vec)
              (letrec ([loop
                        (lambda (i)
                          (if (= i (vector-length vec))
                              #f
                              (let ([x (vector-ref vec i)])
                                (if (and (cons? x)
                                         (equal? (car x) arg-lst))
                                    (second x)
                                    (loop (+ i 1))))))])
                (loop 0)))])
    (lambda args
      (or (vector-assoc args cache)
          (let ([ans (apply func args)])
            (and ans
                 (begin
                   (vector-set! cache next-to-replace
                                (list args ans))
                   (set! next-to-replace
                         (if (= (+ next-to-replace 1) n)
                             0
                             (+ next-to-replace 1)))
                   ans)))))))

(define (fib n)
  (if (< n 2)
      n
      (+ (fib (- n 2)) (fib (- n 1)))))

(define fib-cached
  (make-cached-function
   (lambda (n)
     (if (< n 2)
         n
         (+ (fib-cached  (- n 2)) (fib-cached  (- n 1)))))
   100))

> (time (fib 35))
cpu time: 1181 real time: 1230 gc time: 0
9227465

> (time (fib-cached 35))
cpu time: 0 real time: 0 gc time: 0
9227465
```

一个更简洁的写法是用 Racket 的 hash (thanks to [StackOverflow](http://stackoverflow.com/questions/23170706/is-there-a-valid-usecase-for-redefining-define-in-scheme-racket))：

```lisp
(define (memoize fn)
  (let ((cache (make-hash)))
    (lambda arg (hash-ref! cache arg (thunk (apply fn arg))))))

(define fib
  (memoize
   (lambda (n)
     (if (< n 2) n (+ (fib (sub1 n)) (fib (- n 2)))))))

> (time (fib 35))
cpu time: 0 real time: 0 gc time: 0
9227465
```
