# 第三十六章：Java基础类库(System类)

## 1. 知识点
> 1. System类的基本使用；
> 2. 内存释放操作；

## 2. 具体内容
说到System类一定首先想到两个方法：
* 输出：System.out.println();
* 数组拷贝：System.arraycopy();
  * 完整定义：`public static void arraycopy(Object src,int srcPos,Object dest,int destPos,int length)`

在System类中定义有取得当前日期时间的方法：`public static long currentTimeMillis()`

#### 范例：利用此方法实现操作花费时间的统计
```
public class TestDemo {
    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        long end = System.currentTimeMillis();
        System.out.println("程序运行时间：" + (end-start));
    }
}
```

但是在System类中定义有这样一个方法：`public static void gc()`，但是这个方法并不是实现的新的方法，是继续调用了Runtime类中的gc()方法。

在新对象实例化的时候可以利用构造方法进行相应的处理操作，但是在某一个对象回收之后，那么该如何在回收前给它一些处理的机会呢？在Object类中提供有一个对象回收前的释放操作：
* 方法：`protected void finalize() throws Throwable`，此处可以抛出Error或Exception，但是不管抛出谁，最终都一定要被回收。

#### 范例：观察对象回收
```
class Person{

    public Person(){
        System.out.println("张三诞生了！");
    }

    @Override
    protected void finalize() throws Throwable {
        System.out.println("张三挂了！");
        throw new Exception("老子还要再活500年！");
    }
}
public class TestDemo1 {
    public static void main(String[] args) {
        Person per = new Person();
        per = null;
        System.gc();
    }
}
```

##### 面试题：请解释final、finally、finalize的区别？
* final：定义不能够被继承的父类、不能够被子类所复写的方法、定义常量；
* finally：实在异常处理中进行异常处理的统一出口；
* finalize：是Object类的一个方法`protected void finalize() throws Throwable`，是在对象回收前进行对象收尾的操作；

## 3. 知识点总结
> 1. System.currentTimeMillis()；
> 2. gc永远只有一个方法，在Runtime类中定义的；
