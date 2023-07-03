---
title: '浅析 Subtyping'
date: '2018-11-13T01:14:13+08:00'
draft: false
tags: ['Typed Racket', 'Java', 'Subtyping']
authors: ['default']
---

## 概述

类型可以看作是满足某一条件的一类值的集合的名称。编程语言中的静态类型系统是为了（静态地）防止值与类型的错误匹配，从而在程序运行前就发现和类型相关的错误。**子类型（Subtyping）**概念的引入是为了增加类型系统的表达力同时也增强了它的实用性。先通过 Typed Racket 来更好的理解一些通用的概念（本文中的程序可以通过在 [DrRacket](https://racket-lang.org/) 编辑器的第一行加上`#lang typed/racket`来运行）。下面是用`struct`定义的一个 point 类型`pt`，以及计算`pt`到原点距离的函数`dist-to-origin`：

```racket
(struct pt ([x : Real] [y : Real]))

(: dist-to-origin (-> pt Real))
(define (dist-to-origin pt)
  (sqrt (+ (sqr (pt-x pt))
           (sqr (pt-y pt)))))

(dist-to-origin (pt 3 4))
;; => 5
```

第三行`(: dist-to-origin (-> pt Real))`用来声明`dist-to-origin`的参数和返回值类型分别为`pt`和`Real`。如果现在有另一个 point 类型`color-pt`，除横纵坐标（x，y）外增加了一个颜色属性:

```racket
;; color-pt 继承自 pt，同时也增加了新属性 c
(struct color-pt pt ([c : String]))
```

一个很自然的想法是希望类型系统也允许`dist-to-origin`接受`color-pt`这个参数类型，毕竟计算到原点的距离只需要 x，y 两个属性就足够。这也正是子类型系统的核心思想：允许一个类型的值同时也属于另一个含有更少信息的类型，前者叫做后者的子类型。在上例中`pt`相较于`color-pt`含有更少的信息（没有颜色属性），所以`color-pt`的值同时也属于`pt`类型，`color-pt`是`pt`的一个子类型。引入了子类型这个概念也使得类型系统有了一个新的特性，可替代性：程序中接受某一类型值的位置可以替换成子类型的值而不会造成类型错误。所以 Typed Racket 允许`dist-to-origin`函数接受`color-pt`类型的参数：

```racket
(dist-to-origin (color-pt 3 4 "red"))
;; => 5
```

## 深度 Subtyping

为了增强类型系统的表达力，Typed Racket、ML 等语言引入了多态（polymorphism），或者更准确的说，参数多态（parametric polymorphism）的概念，所以就有了含有类型参数的类型（类似的概念在 Java 中被叫做 generics，而 polymorphism 则被 Java 的设计者用来指代 OOP 的一个核心特性，dynamic dispatch）。比如`(Listof t)`和`'a list`分别是 Typed Racket 和 ML 中的列表类型，`t`和`'a`是类型参数，用来表示列表中元素的类型。一个很自然的问题是，列表会因为所含元素的类型的父子关系而也有了相同的父子关系吗？比如在 Typed Racket 中，`(Listof color-pt)`是`(Listof pt)`的子类型吗？答案是，对于列表类型来说，这个父子关系是成立的，使得这个关系成立的规则就叫做 **深度（Depth）Subtyping**。用一个例子来说明：

```racket
(: sum-of-dist (-> (Listof pt) Real))
(define (sum-of-dist pt-list)
  (foldl (lambda ([p : pt] [accu : Real])
           (+ (dist-to-origin p) accu))
         0
         pt-list))

(sum-of-dist (list (pt 3 4) (pt 3 4)))
;; => 10

(sum-of-dist
  (list (color-pt 3 4 "red")
        (color-pt 3 4 "green")))
;; => 10
```

用`sum-of-dist`来对列表中所有点到原点的距离求和。虽然声明的参数类型是`(Listof pt)`，它也接受了`(Listof color-pt)`这个类型的参数，进而说明`(Listof color-pt)`是`(Listof pt)`的子类型。对于列表来说，使得 depth subtyping 成立的一个关键特性是它的**不可更改性（immutability）**。如果将上例中的 List 换成 Mutable List，那么 depth subtyping 将不再成立：

```racket
(: m-pts (MListof pt))
(define m-pts
  (mcons (pt 1 1) '()))

(: m-color-pts (MListof color-pt))
(define m-color-pts
  (mcons (color-pt 1 1 "yellow") '()))

(: replace-first (-> (MListof pt) Real Real Void))
(define (replace-first pts x y)
  (when (not (null? pts))
      (set-mcar! pts (pt x y))))

(replace-first m-pts 2 2)
m-pts
;; => (mcons (pt 2 2) '())

(replace-first m-color-pts 2 2)
;; Type Checker: type mismatch
;;  expected: (MListof pt)
;;  given: (MListof color-pt) in: m-color-pts
```

为什么是否 mutable 会决定 depth subtyping 的成立与否？假设上例中 18 行的函数调用被类型系统所允许，那么接下来如果执行`(color-pt-c (mcar m-color-pts))`将会导致运行时错误，这个错误的原因是值与类型的错误匹配，而这本该是类型系统在程序运行前就检查出进而阻止的。像这样没能阻止不该发生的错误的类型系统，我们说它是 **unsound** 的。所以 depth subtyping 对于 immutability 的依赖是为了保证类型系统的 soundness。

## 函数的 Subtyping

把函数作为值来传递时所要遵从的 subtyping 规则要更复杂一些。因为函数的类型由它的参数和返回值的类型所决定，所以相应的 subtyping 规则需要从参数和返回值两方面分别考虑。下面的`transform-then-dist`是一个高阶函数，它接受一个类型为`(-> color-pt pt)`的函数`f`以及一个`color-pt`类型的`cpt`作为参数，用`f`将`cpt`转换后求到原点的距离：

```racket
(: transform-then-dist (-> (-> color-pt pt) color-pt Real))
(define (transform-then-dist f cpt)
  (dist-to-origin (f cpt)))

(: f1 (-> color-pt pt))
(define (f1 cp)
  (pt (pt-x cp) (pt-y cp)))

(transform-then-dist f1 (color-pt 3 4 "blue"))
;; => 5
```

`f1`的返回值作为参数又传给了`dist-to-origin`函数，而在前面已经知道`dist-to-origin`是可以接受`pt`以及它的子类型`color-pt`作为参数，所以如果把`f1`换成一个返回值为`pt`子类型的函数应该也会被类型系统所接受：

```racket
(: f2 (-> color-pt color-pt))
(define (f2 cpt) cpt)

(transform-then-dist f2 (color-pt 3 4 "blue"))
;; => 5
```

因此`f2`也就是`f1`的子类型。一般来说，如果函数 2 的返回值是函数 1 的返回值的**子类型**，那么函数 2 本身也是函数 1 的子类型。返回值类型的父子关系与函数类型的父子关系保持一致的这种关系叫做**协变的（covariant）**。现在再从参数类型的角度考虑。`f1`（以及`f2`）在`transform-then-dist`的实现中是作为消费者来消费`transform-then-dist`的另一个参数`cpt`。作为消费者来说，如果要消费的值的类型（也就是声明的参数类型）是 t1，然后得到了一个比 t1 含有更多信息的 t2 类型的值，那么它的需求也是会得到满足。因为 t2 比 t1 含有更多信息，那么 t2 是 t1 的子类型。对于`f1`来说，t1 是它的参数类型`color-pt`，t2 是`transform-then-dist`传给它的类型即参数`cpt`的类型，也是`color-pt`。现在的情况是对于`f1`来说，它要消费的`cpt`的类型（t2）是固定的`color-pt`，那么把`f1`替换成任何参数类型是`f1`参数的父类型的函数（这里称作`f3`），等到的程序也应该是合法的。还是得通过一个例子来使这个概念阐述的更明白：

```racket
(: f3 (-> pt pt))
(define (f3 p) p)

(transform-then-dist f3 (color-pt 3 4 "blue"))
;; => 5
```

因为`transform-then-dist`的参数`f`声明的类型是`(-> color-pt pt)`，所以`f3`的类型`(-> pt pt)`是`(-> color-pt pt)`的子类型。更宽泛的说，如果函数 2 的参数是 函数 1 参数的**父类型**，那么函数 2 本身也是函数 1 的子类型。参数类型的父子关系与函数类型的父子关系相反的这种关系叫做**逆变的（contravariant）**。所以对于函数的 subtyping 规则只需记住，函数与参数的类型是逆变的，与返回值的类型是协变的。

## Java 中 的 Depth Subtyping

Java 中实现 subtyping 的途径是继承类和实现接口。但是需要认清的是，类（class）和类型（type）是不同的两个概念，类定义了对象的行为，而类型描述的是对象含有哪些 field 和 method（在这一点上接口和类型有更多的共同点）。

在[前面](#深度-subtyping)提到过 depth subtyping 必须建立在 immutablity 先决条件上才能保证类型系统的 soundness。但是 Java 中的数组是 mutable 的，同时也允许 depth subtyping，这就导致有些情况下程序会抛出运行时异常`ArrayStoreException`：

```java
ColorPoint[] cpts = {new ColorPoint(1, 2, "red")};
Point[] pts = cpts;
pts[0] = new Point(1, 2); // throws unchecked java.lang.ArrayStoreException
System.out.println(cpts[0].color);
```

Java 选择把检查放在写值入数组的时候，这样保证了数组元素类型的一致，也减少了在每次读元素时再检查造成的性能开销。Generics 被加到 Java 里后不再自动支持 depth subtyping，`List<ColorPoint>` 不再是`List<Point>`的子类型，如果想得到类似 depth subtyping 的表达力，可以使用叫做 **bounded wildcards** 的特殊语法：

```java
List<Point> pts = Lists.newArrayList(new Point(1, 2));
List<ColorPoint> cpts = Lists.newArrayList(new ColorPoint(1, 2, "red"));
List<? extends Point> subPoints = cpts;
List<? super ColorPoint> superColorPoints = pts;
```

但是为了保证类型系统的 soundness，用 bounded wildcards 声明的变量的使用上也有一些限制：1) 对`? extends T`声明的变量读操作可以得到类型为`T`的值，但不能对其进行写操作；2) 可以将`T`类型的值写入`? super T`声明的变量，但不能对其进行读操作。

```java
Point p = subPoints.get(0);
subPoints.add(new Point(1, 2)); // 编译错误

superColorPoints.add(new ColorPoint(1, 2, "red"));
ColorPoint cp = superColorPoints.get(0); // 编译错误
```

## 参考

- [Programming Languages, Part C](https://www.coursera.org/learn/programming-languages-part-c) on Coursera
- [The Typed Racket Guide](https://docs.racket-lang.org/ts-guide/index.html)
- [Effective Java, 3rd Edition](https://www.oreilly.com/library/view/effective-java-3rd/9780134686097/)
