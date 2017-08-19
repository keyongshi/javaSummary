Java参数传递

## 原理
1、对象是按引用传递的
2、Java 应用程序有且仅有的一种参数传递机制，即按值传递
3、按值传递意味着当将一个参数传递给一个函数时，函数接收的是原始值的一个副本
4、按引用传递意味着当将一个参数传递给一个函数时，函数接收的是原始值的内存地址，而不是值的副本
5、String类型因为没有提供自身修改的函数，每次操作都是新生成一个String对象，所以要特殊对待。可以认为是传值。
 

Java 应用程序中的变量可以为以下两种类型之一：引用类型或基本类型。当作为参数传递给一个方法时，处理这两种类型的方式是相同的。两种类型都是按值传递的；没有一种按引用传递。

我们用一个例子来说明

## 例子
基本类型作为参数传递时，传递的是值的拷贝，也就是说，函数内改变了拷贝，外部值是不会改变。
引用类型作为参数传递时，是把对象在内存中的地址拷贝了一份传给了参数

```java
public class Test {
    @Test
    public void test() {
        StringBuffer sbGood = new StringBuffer("good ");
        StringBuffer sbBad = new StringBuffer("bad ");
        int n = 5;
        String str = "origin string";
        System.out.println("Before change, sbGood = " + sbGood);
        System.out.println("Before change, sbBad = " + sbBad);
        System.out.println("Before change, n = " + n);
        System.out.println("Before change, str = " + str);
        changeData(sbGood, sbBad, n, str);
        System.out.println("=========================== ");
        System.out.println("After changeData(n), sbGood = " + sbGood);
        System.out.println("After changeData(n), sbBad = " + sbBad);
        System.out.println("After changeData(n), n = " + n);
        System.out.println("After changeData(n), str = " + str);
    }
    
    public void changeData(StringBuffer sbGood, StringBuffer sbBad, int n, String str) {
        sbGood.append(" World!");
        StringBuffer sb2 = new StringBuffer("Hi ");
        sbBad = sb2;
        sb2.append("World!");
        n = 6;
        str = "change";
    }
}

```
输出结果为
```
Before change, sbGood = good 
Before change, sbBad = bad 
Before change, n = 5
Before change, str = origin string
=========================== 
After changeData(n), sbGood = good  World!
After changeData(n), sbBad = bad 
After changeData(n), n = 5
After changeData(n), str = origin string
```


