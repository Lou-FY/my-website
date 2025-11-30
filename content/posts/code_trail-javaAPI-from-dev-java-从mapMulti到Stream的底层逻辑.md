---
title: 从mapMulti到Stream的底层逻辑
date: 2025-11-29
tags:
  - Java
publish: true
---
首先，mapMulti或许用的不是特别多——和flatmap相比就只是多了一个优点：不会产生对于每个元素的中间stream对象，减少了开销。但是因为对它写法的一系列疑惑，结果促使探究到了stream的链路逻辑。

## 场景与mapMulti&faltMap

```java
//场景：将一个数字字符串列表转为Integer类型的列表，同时去除不合法字符
//实现：使用更省开销的mapMulti方法
List<String> strings = List.of("1", " ", "2", "3 ", "", "3");

List<Integer> ints =
   strings.stream()
          .<Integer>mapMulti(
              (string, consumer) -> {
                  try {
                      consumer.accept(Integer.parseInt(string));
                  } catch (NumberFormatException ignored) {
              }
          })
          .toList();
IO.println("ints = " + ints);
```

1. 一些基础
   - java16后引入
   - 该方法传入一个BiConsumer
     - 需要被映射的元素
     - **调用一个Consumer，用来存放最终结果流，达到不产生中间过程流的目的**（相比于`flatMap`）
2. 因为这里逻辑稍复杂，防止编译器混乱，泛型参数类型推断出错，手动进行了参数类型指定。
   - 对于方法：泛型写在方法名前「更准确是写在返回类型前」
   - 对于类：泛型写在类名后
3. 代码块逻辑：
   1. 若元素能被`Integer.parseint`准确转化，就将此结果传入一个结果流中，而不是形成一个个小的stream对象再进行展平（flatMap）
   2. 若失败，报出受检异常，并被catch捕获处理

## 传的consumer是什么东西？

怎么会蹦一个consumer出来？内部定义的？定义这个干嘛？

1. 要弄明白这个问题就需要深入stream的底层了

2. 直觉式解释：
   1. stream以“pipeline”式处理流式数据闻名。源数据经过一系列中间操作累积处理逻辑，最后在最终操作那一口气处理。那么是怎么将这些操作逻辑累积下去的呢？
   2. 答：consumer。每次中间操作都会进行这样的逻辑：承载接收上流操作、将本次操作加入——“`consumer.accept()`”，传递到终端操作处，统一处理
   3. mapMulti这里是**显式地**将内部进行的consumer操作作为参数传递，一旦有合法字符转化了，就将它传递。从而达成不产生中间流的作用

3. 实际：`AbstractPipeline` + `Sink`。不过还是拿consumer来搭建心智模型：
   ```java
   [源数据] → [map1] → [filter] → [map2] → [终止操作 toList]
   headSink  →  map1Sink  →  filterSink  →  map2Sink  →  terminalSink
   ```

   - **Stream 是惰性的**：只有调用终止操作（如 `toList()`）时，才真正开始把元素从源头流过整个管道。
   - 终止操作首先创建了**最底层的“真实 consumer”**：比如 `x -> result.add(x)`。
   - 然后，从最后一个中间操作开始，**每一层都拿到“下游 consumer”并返回一个“包了一层逻辑的上游 consumer”**。
   - 这一层层 wrap 下来，最外层的那个 consumer，就对应管道最前面的操作（第一个 map/filter）。
   - 当源数据被遍历时，只调用这个最外层的 `head.accept(x)`，它内部会按顺序调用各个中间操作逻辑，最后传到终止操作的 consumer 上。

   ```java
   //伪实现
   //终端操作：
   List<T> toList() {
       List<T> result = new ArrayList<>();
   
       // 1. 在终止操作里，先创建最底层的 consumer
       Consumer<T> terminal = x -> result.add(x);
   
       // 2. 从“最后一个中间操作”开始，往前一层层 wrap
       Consumer<?> head = terminal;
       Stage<?> s = this;         // this = 最后一个 stage（最近的 map/filter）
       while (s != null) {
           head = s.wrap(head);   // 每一层都“包住”下游 consumer
           s = s.upstream;        // 然后跳到上游那层
       }
   
       // 3. 最终得到 head，是最前面的那个 map/filter 封装出来的“总入口”
       for (Object element : getSource()) {
           ((Consumer<Object>) head).accept(element); // 启动整条链
       }
   
       return result;
   }
   
   //中间操作：
   class FilterStage<T> extends Stage<T> {
       private final Predicate<T> pred;
   
       FilterStage(Stage<T> upstream, Predicate<T> pred) {
           super(upstream);
           this.pred = pred;
       }
   
       @Override
       <X> MyConsumer<T> wrap(MyConsumer<X> downstream) {
           return (T value) -> {
               if (pred.test(value)) {
                   ((MyConsumer<T>) downstream).accept(value);
               }
               // 否则就“拦截”掉，不往下传
           };
       }
   }
   ```

4. 传的consumer就是该步操作产生的新consumer（将来会作为upStream向上传递，等待被包装）。
   ```java
   //伪实现
   class MapMultiStage<T, R> extends Stage<R> {
       private final BiConsumer<T, MyConsumer<R>> mapper;
   
       MapMultiStage(Stage<T> upstream, BiConsumer<T, MyConsumer<R>> mapper) {
           super(upstream);
           this.mapper = mapper;
       }
   
       @Override
       <X> MyConsumer<T> wrap(MyConsumer<X> downstream) {
           // 返回的这个“新消费者”，就是你前面 lambda 里的第二个参数的真实来源
           MyConsumer<R> asDownstream = (R r) -> ((MyConsumer<R>) downstream).accept(r);
   
           return (T value) -> {
               // 注意这里：把 asDownstream 作为“参数传给你的 lambda”
               mapper.accept(value, asDownstream);
               // 你在 lambda 里写的 consumer.accept(...)，本质就是调的 asDownstream.accept(...)
           };
       }
   }
   ```

   

## 终止操作之后，处理元素的方式？

都是一个元素一个元素处理的么？

1. 如map、filter都是逐元素的、无状态的「和之前的经过无关」

   - `来一个->处理一下->传一个`

2. 如sorted、distinct是有状态的

   - distinct通过存一张set来进行重复元素去除；sorted通过新建list等来进行排序

     - 但distinct：`来一个->通过了->就往下传一个`

     - 但sorted：因为要排序，需要等上游元素全到了，再进行排。`上游元素不齐->不传`
       ```java
       ///sorted伪代码
       // 1. 先把上游所有元素收集起来
       List<T> buffer = new ArrayList<>();
       upstream.forEach(buffer::add);  // 如果是无限流，这步永远不会结束
       
       // 2. 排序
       Collections.sort(buffer);
       
       // 3. 再一个个往下游传
       for (T t : buffer) {
           downstream.accept(t);
       }
       ```

3. `limit(max)`：来一个放行一个，但会在内部进行计数，通过的数量达到max了，就截断，不再向上游要数据，如果下游有sorted，就会进行排序

```java
///不会有结果
var ints =IntStream.iterate(0,i->i+1)
                .map(i->i+3)
                .distinct()
                .sorted()
                .limit(5)//放在这里
                .toArray();
///有结果
var ints =IntStream.iterate(0,i->i+1)
                .map(i->i+3)
  							.limit(5)//放在这里
                .distinct()
                .sorted()
                .toArray();
```



## log

- 2025-11-30-20:27:21
  - 增加“终止操作之后，处理元素的方式？”段落

