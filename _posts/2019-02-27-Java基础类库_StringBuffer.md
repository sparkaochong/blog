# 第三十七章：Java基础类库(StringBuffer、StringBuilder)

## 1. 知识点
> * StringBuffer类的特点；
* StringBuffer、StringBuilder、String类的关系；

## 2. 具体内容
String类有哪些特点？
* 字符串常量就是String类的匿名对象，一旦字符串定义则不可改变；
* String类对象可以使用直接赋值或者是构造方法实例化，前者可以自动入池，又不产生垃圾空间；

在实际的开发之中，肯定都要求使用到String类，可是String类有一个缺陷：不可改变，所以如果要想面对经常修改的环境下只能够使用StringBuffer类。

在String类中可以使用"+"来实现字符串的拼接操作，而在StringBuffer类中可以使用append()方法实现字符串拼接操作。此方法定义如下：
`public StringBuffer append(数据类型 变量)`
#### 范例：利用StringBuffer修改字符串
```
public class TestDemo {
    public static void main(String[] args) {
        StringBuffer sb = new StringBuffer();
        sb.append("Hello ").append("World!");
        fun(sb);
        System.out.println(sb);
    }

    public static void fun(StringBuffer sb){
        sb.append("\n世界，你好！");
    }
}
```

此时执行的结果之中发现字符串的内容已经得到了改变，所以StringBuffer适合于修改，而String不适合于修改。
原则：在开发之中百分之95的情况下使用的都应该是String（String不适合于频繁的修改），所以可能通过循环操作String，但是如果是真的进行连接操作，还建议使用"+"。如果真的是循环修改，肯定使用StringBuffer。
`String str = "Hello " + "World " + "!!!";`
现在字符串类有两个类：String、StringBuffer，那么这两个类有什么关系呢？

| String | StringBuffer |
| ------ | ------------ |
|    public final class String extends Object implements Serializable, Comparable<String>, CharSequence    |      public final class StringBuffer extends Object implements Serializable, CharSequence        |
发现String和StringBuffer类都是CharSequence接口的子类；所以在日后一定要记住String或StringBuffer都可以向上转型微CharSequence接口实例化。
```
public class TestDemo1 {
    public static void main(String[] args) {
        String str = "Hello " + "World " + "!!!";
        CharSequence cs = str;
        System.out.println(cs.subSequence(5,8));
    }
}
```
但是现在出现了一个问题，String与StringBuffer类的对象该如何互相转换呢？对于这两个类对象的互相转换，就可以利用以下原则完成：
* String转换为StringBuffer
  * 利用StringBuffer类的构造方法；
  `public StringBuffer(String str)`
  ```
  public class TestDemo2 {
    public static void main(String[] args) {
        String str = "Hello " + "World " + "!!!";
        StringBuffer sb = new StringBuffer(str);
        System.out.println(sb);
    }
  }
  ```
  * StringBuffer类中存在append()方法；
  `public StringBuffer append(Object obj)`
  ```
  public class TestDemo3 {
    public static void main(String[] args) {
        String str = "Hello " + "World " + "!!!";
        StringBuffer sb = new StringBuffer();
        sb.append(str);
        System.out.println(sb);
    }
  }
  ```
* StringBuffer变为String
  * 所有类都存在有toString()方法，利用此方法可以将StringBuffer转为String。
  ```
  public class TestDemo4 {
    public static void main(String[] args) {
        StringBuffer sb = new StringBuffer("Hello ");
        sb.append("World ").append("!!!");
        String str = sb.toString();
        System.out.println(str);
    }
  }
  ```
  * 利用"+"实现所以数据类型向String的转换。
  
```
public class TestDemo5 {
    public static void main(String[] args) {
        StringBuffer sb = new StringBuffer("Hello ");
        sb.append("World ").append("!!!");
        String str = "" + sb;
        System.out.println(str);
    }
}
```

虽然在开发之中，要进行字符串的操作都是以String类为主，StringBuffer类也提供有一些好的操作方法，方便用户开发代码。
#### 范例：在指定位置增加新的内容
**方法：**`public StringBuffer insert(int dstOffset,数据类型 变量)`

```
public class TestDemo6 {
    public static void main(String[] args) {
        StringBuffer sb = new StringBuffer("Hello ");
        sb.append("World ").append("!!!");
        sb.insert(0,"你好").insert(1,"世界！");
        System.out.println(sb);
    }
}
```

#### 范例：反转
**方法：**`public StringBuffer reverse()`

```
public class TestDemo7 {
    public static void main(String[] args) {
        StringBuffer sb = new StringBuffer("Hello ");
        sb.append("World ").append("!!!");
        sb.reverse();
        System.out.println(sb);
    }
}
```

#### 范例：删除指定范围的数据
**方法：**`public StringBuffer delete(int start,int end)`
```
public class TestDemo8 {
    public static void main(String[] args) {
        StringBuffer sb = new StringBuffer("Hello World!!!");
//        sb.delete(5,15);
        sb.deleteCharAt(10);
        System.out.println(sb);
    }
}
```
以上的三个操作方法是掌握StringBuffer类的主要操作。
> StringBuffer类是在JDK1.0的时候提供的，而在JDK1.5又提供一个StringBuilder类，这两个类单看文档都是一样的,介绍文字是有使用差别的。StringBuffer属于线程安全的操作，性能不高。，而StringBuilder类属于非线程安全的操作，性能高。

##### 面试题：请解释String、StringBuffer、StringBuilder类的区别？
* String类的内容一旦声明了则不可改变，而StringBuffer、StringBuilder的内容可以改变；
* String、StrigBuffer、StringBuilder都是CharSequence接口的子类；
* StringBuffer是从JDK1.0提供的，属于线程安全的操作，性能较低。而StringBuilder是从JDK1.5提供的，属于非线程安全的操作(异步)，性能较高。

## 3. 知识点总结
字符串的操作首选的一定是String类，可改变的时候再选择StringBuffer、StringBuilder，多个线程访问同一资源时i，必须使用StringBuffer类。
