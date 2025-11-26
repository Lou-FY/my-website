---
title: "lambda写法：不捕获外部变量（non-capturing）"
date: 2025-11-15
tags:
  - coding
  - 最佳实践
publish: true  
---
 #最佳实践 
一般来说不会有这个问题，老老实实进行“定义接口->写实现类/匿名类->new对象->调用方法”自然会进行变量层面的抽象。
但lambda这种语法糖太简洁了，导致写的时候会忍不住追求更方便写法，导致jvm运行时的不方便

```java
List<String> strings = List.of("one", "two", "three", "four", "five", "six", "seven");
Map<Integer, String> map = new HashMap<>();
for (String word: strings) {
    int length = word.length();
    map.merge(length, word, 
              (existingValue, newWord) -> existingValue + ", " + newWord);//如果length不在map中，会绑定到word；如果存在，会用现有值和word调用双函数，双函数的结果替换当前值
}
map.forEach((key, value) -> IO.println(key + " :: " + value));

```
1. 为什么这些lambda明明可以直接用外面的变量，如上面的newword，但还要把它作为一个参数传进来？
   - 可以让lambda变成“non-capturing”（不捕获外部变量），性能更好 
   - 对于non-capturing，JVM 可以把它当成**纯函数对象、无状态对象**来优化
     - 可以实现成**一个没有字段的单例对象**，甚至整个程序里就用这一份；一次创建，到处复用。
     - 不需要每次创建新对象；
     - 更容易被 JIT 内联、优化。
   - 如果进行捕获了，lambda 用到了外部的变量，编译器必须生成一个**带字段的对象**，里面存着 bonus 的值。在某些场景下，甚至每次使用都要 new 一份（或者至少要为每个不同捕获环境生成不同实例），**分配成本更高，优化空间更小**。