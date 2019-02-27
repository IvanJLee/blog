---
title: Kotlin中的数字类型的相等性
tags: Kotlin
date: 2019-02-27 14:44:10
---



## 相等性

### 相等性分类
Kotlin中有2种相等性
- 结构型相等（通过equals比较）
- 引用型相等（比较两个引用是否指向同一个对象）

可以理解为结构型相等是值比较，和Java中的equals类似，引用型相等是通过比较引用指向的对象是否为同一个，和Java中的==类似。

Kotlin中，结构性相等使用双等号==来表示（否定形式是!=），引用型相等使用恒等号===表示（否定形式是!==）。

### 数字类型的相等性

在Kotlin中，万物皆对象，对于数字类型，可以直接调用其包装类型的函数。但是，对于数字类型来说，在有些场景中，它们又在运行时被表示称基本类型，而不是包装类型。

Kotlin运行在在JVM中时，数字类型通常被存成基本类型而不是包装类型。但是，如果数字被声明成了可空类型，则会被存储成包装类型。

#### 整数的相等性

```Kotlin
fun primitive() {
    val a: Int = 128
    println(a == a) // 1
    val primitiveA: Int = a
    println(a === primitiveA) // 2
    val boxedA: Int? = a
    val anotherBoxedA: Int? = a
    println(boxedA === primitiveA)// 3
    println(boxedA === anotherBoxedA)// 4
    println(boxedA == primitiveA)// 5
    println(boxedA == anotherBoxedA)//6
}
```

在这段代码中，注释1～5中的打印结果分别对应：

**1、数字类型始终和自身相等**

a为非空变量，所以a为基本类型，无论是a==a，还是a===a都是true.

**2、基本类型整数，值相等即满足结构型相等和引用型相等**

a和primitiveA都是非空变量，都被存储为基本类型，只要它们的值相等，无论是==还是===，比较的结果都是true.

**3、装箱类型，即便是值相等，只要类型不同，则不满足引用型相等**

由于被声明成了可空变量，boxedA被存储为装箱类型，primitiveA是非空变量，被存储为基本类型，装箱类型和基本类型相比，类型不同，所以不满足引用型相等，boxedA === primitiveA为false.

**4、装箱类型，即使值相等，只要引用不指向同一个对象，不满足引用型相等**

boxedA和anotherBoxedA都是可空变量，都被存储为装箱类型，值都是128，赋值操作会创建出不同的Int对象，所以不满足引用型相等，boxedA === anotherBoxedA值为false.

Kotlin在处理整数的时候，和Java中的Integer一样，0到127都有缓存池，数值小于128时，取到的是同一个对象，大于127则会创建新的对象。所以在上述代码块中，a的值是128，创建了不同的对象，使用===比较，值为false. 将上述代码中a的值改成126，取到的都是缓存池中的对象，因此，即使使用===比较，值也是true.

**5、值相等的数字，满足结构型相等**

在注释5和6中，primitiveA，boxedA，anotherBoxedA值都是128，primitiveA是基本类型，boxedA和anotherBoxedA是装箱类型，使用==比较，只比较它们的值，都相等，都满足结构型相等。

#### 浮点数的相等性

>当相等性检测的两个操作数都是静态已知的（可空或非空的）Float 或 Double 类型时，该检测遵循 IEEE 754 浮点数运算标准。
>
>否则会使用不符合该标准的结构相等性检测，这会导致 NaN 等于其自身，而 -0.0 不等于 0.0。

