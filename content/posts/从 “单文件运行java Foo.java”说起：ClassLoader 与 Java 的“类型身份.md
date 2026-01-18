---
title: "从 “单文件运行java Foo.java”说起：ClassLoader 与 Java 的“类型身份"
date: 2026-01-18
tags:
  - 
publish: true  
---

Java 21/22 之后，`java Foo.java` 这种“单文件运行（single-file）”变得很顺手：写完一个文件，直接运行，不用显式 javac。

它非常适合做小实验、写脚本、快速验证想法。但一旦代码不再是“单文件自包含”，例如你把 Person、Order、Utils 等拆成多个 `.java` 文件，继续用单文件方式运行，就可能遇到一些让人摸不着头脑的运行时错误——甚至出现经典的：

> Person cannot be cast to Person

这篇笔记就从 single-file 的运行方式出发，解释背后的根因：**ClassLoader 与 Java 的“类型身份”**，以及如何避免踩坑。

## 简简简简简述“单文件运行”

传统方式：

```bash
javac Demo.java Person.java
java Demo
```

单文件方式（source launcher）：

```bash
java Demo.java
```

但本质上，第二种是：

1. Java 启动器先把 `Demo.java` 文件编译成字节码（**通常在临时位置/内存中**）
2. 然后立即运行编译产物

但当程序不止一个源文件时，事情开始变复杂。

## 多源文件情况

假设有：

- `Person.java`：一个 public class Person
- `Demo.java`：演示代码（无论用传统 public static void main，还是 Java 21/22 的“隐式声明类/无名类”写法，编译器最终都会生成某个 class 来承载入口逻辑）

<u>运行时</u>至少会涉及两个类：

- Person
- 承载入口逻辑的那个类（比如 Demo）

## ClassLoader

有很多种

- <u>JDK官方的各种类</u>
  - **`bootstrap loader`**——用getClassLoader()查看ClassLoader，会返回null
- <u>自定义类</u>
  - 可能是**`AppClassLoader`**也可能是**`MemoryClassLoader`**
    - 前者是当直接单文件运行 `java Demo.java` 会使用的
    - 后者是当先`javac`编译，然后再`java` 执行会使用的

## JVM 如何判断“是不是同一个类型”？

在 JVM 里，一个类的“身份”不是只由类名决定，而是由下面两部分共同决定：

<u>**类型身份 =（定义它的 ClassLoader）+（类的全限定名）**</u>

因此：

- 即使都是 `com.example.Person`

- 但如果`.class`类文件来自**不同的 ClassLoader**（或同名类被不同方式定义/加载）

- JVM 就会认为它们是**两种不同的类型**。互相不认识

  - 著名报错：`ClassCastException: com.example.Person cannot be cast to com.example.Person`

  - Person@LoaderA 不能转成 Person@LoaderB

对于上述说的「Person」和「承载入口逻辑的那个类」，如果对承载入口逻辑的那个类直接单文件运行，就很容易出发这种情况：

## 为什么 single-file 更容易触发“同名类变成两个类型”？

因为 `java Demo.java` 这类 source launcher 往往引入了“临时编译产物”和“特殊的加载路径/加载方式”。

与此同时，你的工程目录里可能还存在另一份 Person.class 来源，例如：

- 之前用 javac 编译残留的输出
- IDE（IntelliJ / Eclipse）编译输出目录里的 class
- Maven/Gradle 的 target/classes 或 build/classes
- source launcher 为依赖源码又编出的一份 class（位置不同）

结果是：同一个全限定名 Person，在同一次 JVM 运行中，被“从不同地方”定义了两份。

从 JVM 的角度看：

- 它们不是“重复”，而是“两个不同的类型”
- 所以你写的代码在编译时看似都对，运行时却可能崩在类型检查或链接阶段

常见错误包括：

- ClassCastException: X cannot be cast to X
- NoSuchMethodError（编译时有某方法，运行时那份 class 没有）
- LinkageError（类定义冲突/重复定义等）

## 可以牢记的

当你看到“X 不能转成 X”时，不要怀疑人生，可以直接用这个模型解释：

- 代码里写的 Person，其实指向某个具体的 `Person.class`

- 你传进来的对象，也属于某个具体的 `Person.class`

- 如果它们来自不同的 ClassLoader / 不同的类定义来源

  → JVM 认为它们是不同类型

  → 即使包名类名完全一致，也不能互转



**构建与运行使用同一套输出与 classpath**

