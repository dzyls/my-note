### Consumer、Function、Supplier、Predicate

---

函数式编程与OOP不同的是，OOP是将数据抽象出来，而函数式编程是将行为抽象出来。

函数式有四种常用的接口，这些接口的匿名实现类经常作为函数式编程的参数。

##### Consumer

---

Consumer这个接口一看就是一个消费者。

```java
public interface Consumer<T> {
    void accept(T t);
}
```

这个接口的accept方法，首先是无返回值的，是对流中的数据进行消耗。

T类型是方法的参数类型。

如果使用常规的写法就是这样 ：

```java
Consumer<String> consumer = new Consumer<String>(){
    @Override
    public void accept(String s) {
        System.out.println(s);
    }
};
```

等同于下面这种写法 ：

```java
Consumer<String> consumer = System.out::println;
```

示例 ：

```java
String []strArray = {"a","b","c","d","e"};
Arrays.stream(strArray).forEach(new Consumer<String>() {
    @Override
    public void accept(String s) {
        System.out.println(s);
    }
});
```

Stream中`forEach`和`forEachOrder`的参数是Consumer接口。



##### Function

---

Function接口是一个功能性接口，它的作用就是，将数据经过运算或者处理，转换成另外一种数据。

```java
public interface Function<T, R> {
    R apply(T t);    
}
```

接口的T参数类型是入参类型，R类型是返回参数类型。

示例 ：

```Java
Function<String, String> function = new Function<String, String>() {
    @Override
    public String apply(String s) {
        return s + "c";
    }
};
```

```java
String []strArray = {"a","b","c","d","e"};
Arrays.stream(strArray).map(s->s + "c").forEach(System.out::println);
```

如上，Function接口可以将数据进行转换、处理，然后供Consumer接口使用。

Stream的方法中使用到Function的有 ：

- map
- flatMap

在此说明map和flatMap的区别 ：

map是将流中的数据，替换为另外一种数据。

flatMap则是将流中的数据，换成流拆解的流元素，返回值是Stream。



举例 ：

```java
String []strArray = {"Hello","loli"};
Stream<String[]> stream = Arrays.stream(strArray).map(s -> s.split("")).distinct();
stream.forEach(System.out::print); // 打印的是两个String数组的地址
```

```java
String []strArray = {"Hello","loli"};
Arrays.stream(strArray).map(s -> s.split("")).flatMap(Arrays::stream).distinct().forEach(System.out::println);
// 打印的是 Heloi
```

flatMap干的是将流中的集合再次拆解为单个元素的流。比如说，有两箱苹果，flatMap是将两箱苹果都拆解，再把所有的苹果放到流里面，供后续操作。



##### Supplier

---

Supplier可以理解为一个容器，可以用来装数据。

```java
public interface Supplier<T> {
    T get();
}
```

示例 ：

```java
Supplier<String> supplier = new Supplier<String>() {
    @Override
    public String get() {
        return "NotExist";
    }
};
```

```java
List<String> asList = stringStream.collect(ArrayList::new, ArrayList::add,
                                                ArrayList::addAll);
```

##### Predicate

---

Predicate接口，顾名思义，就是一个断言接口，用来对数据进行判断的。

```java
public interface Predicate<T> {
    boolean test(T t);   
}
```

示例 ：

```java
String []strArray = {"a","b","c","d","e"};
System.out.println(Arrays.stream(strArray).map(s -> s + "c").filter(s -> s.contains("a")).findAny().orElse("NotExist"));
```

这个接口就很常用，也很简单。



### CompletableFuture

---

Future接口可以执行异步操作，CompletableFuture是对Future接口的扩充，它可以 ：

- 将多个异步计算的接口合并为1个
- 等待所有Future都完成（用CountDownLatch也可以虽然）
- Future完成通知

