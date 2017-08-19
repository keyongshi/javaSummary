### 为什么需要拷贝
在实际工作过程中，我们会有这样的场景：有一个对象A，在某个时刻包含了一些属性值，而此时可能会需要一个和A完全相同新对象B，也就是说，A与B是两个独立的对象，但B的初始值是由A对象确定的，并且此后对B任何改动都不会影响到A中的值。

### Clone实现拷贝
在Java语言中，B = A显然不能满足刚才的场景，需要对A进行拷贝。而最常见的拷贝就是使用clone方法.

将需要拷贝的类实现Cloneable接口，并@Override clone方法

```java
public class Star implements Cloneable{ 
    private int id;
    private Movie movie;
    
    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }    
    
    public Movie getMovie() {
        return movie;
    }

    public void setMovie(Movie movie) {
        this.movie = movie;
    }
    
    public Object clone(){ 
        Star o = null; 
        try{ 
            o = (Star)super.clone(); 
        }catch(CloneNotSupportedException e){ 
            e.printStackTrace(); 
        } 
        return o; 
    } 
    
    public String toString() {
        return "star id:" + id + ", movie, name:" + movie.getName();
    }
}
```
```java
public class Movie {
    private String name;
    
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

测试代码如下：

```java
public class test{
    @Test
    public void test() {
        Star star1 = new Star();
        Movie movie = new Movie();
        movie.setName("movie 1");
        star1.setMovie(movie);
        star1.setId(1);

        System.out.println(star1.toString());

        Star star2 = (Star)star1.clone();
        star2.setId(2);
        star2.getMovie().setName("movie 2");

        System.out.println(star1.toString());
        System.out.println(star2.toString());
    }
    
}
```
结果如下：
```
star id:1, movie, name:movie 1
star id:1, movie, name:movie 2
star id:2, movie, name:movie 2
```

我们能看到clone出来的star2对象，将id改为2之后，并没有影响star1的id

### 浅拷贝和深拷贝

但是从另一个细节来看，star2对象中的movie，修改了name之后，影响了star1对象中movie的name。为了解决这个问题，就需要了解到浅拷贝和深拷贝

浅拷贝是指拷贝对象时仅仅拷贝对象本身（包括对象中的基本变量），而不拷贝对象包含的引用指向的对象。

深拷贝不仅拷贝对象本身，而且拷贝对象包含的引用指向的所有对象。

举例来说更加清楚：
- 对象star1中包含对Movie的引用，Movie中包含对name的引用。浅拷贝star1得到star2，star2 中依然包含对movie的引用，movie中依然包含对name的引用。
- 深拷贝则是对浅拷贝的递归，深拷贝star1得到star2，star2中包含对movie2（movie的copy）的引用，movie2 中包含对name2（name的copy）的引用。


知道原理之后，将上面的代码改成
```java
public class Star implements Cloneable {
    private int id;
    private Movie movie;

    public Object clone(){
        Star o = null;
        try{
            o = (Star)super.clone();
        }catch(CloneNotSupportedException e){
            e.printStackTrace();
        }
        o.setMovie((Movie)movie.clone());
        return o;
    }

    public String toString() {
        return "star id:" + id + ", movie, name:" + movie.getName();
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public Movie getMovie() {
        return movie;
    }

    public void setMovie(Movie movie) {
        this.movie = movie;
    }
}
```
```java
public class Movie implements Cloneable {
    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Object clone() {
        Movie movie = null;
        try {
            movie = (Movie)super.clone();
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
        }
        return movie;
    }
}
```

修改之后，运行程序，结果如下：
```
star id:1, movie, name:movie 1
star id:1, movie, name:movie 1
star id:2, movie, name:movie 2
```
star2的修改完全不会影响到star1中的内容

### clone细节
从前面的代码可知：
- 重载了clone()方法,并且将方法属性设置为public。在clone()方法中调用了super.clone()，调用了java.lang.Object类的clone()方法。该方法是一个native方法，native方法的效率一般来说都是远高于java中的非native方法。这也解释了为什么要用Object中clone()方法而不是先new一个类，然后把原始对象中的信息赋到新对象中的方式。
- 类实现了Cloneable接口。该接口不包含任何方法，为什么要实现呢？其实这个接口仅仅是一个标志，而且这个标志也仅仅是针对Object类中clone()方法的，如果clone类没有实现Cloneable接口，并调用了Object的clone()方法（也就是调用了 super.Clone()方法），那么Object的clone()方法就会抛出CloneNotSupportedException异常
```
protected native Object clone() throws CloneNotSupportedException;
```

### 其他深拷贝方法
当然还有其他的深拷贝方法，例如将对象串行化或使用第三方库的copy，代码如下，不进行详细讲解

```java
public class Teacher implements Serializable {
    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

}
```
```java
public class Student implements Serializable {
    private String name;
    private Teacher teacher;

    public Object deepClone() throws IOException,
        ClassNotFoundException {
        // 将对象写到流里
        ByteArrayOutputStream bo = new ByteArrayOutputStream();
        ObjectOutputStream oo = new ObjectOutputStream(bo);
        oo.writeObject(this);
        // 从流里读出来
        ByteArrayInputStream bi = new ByteArrayInputStream(bo.toByteArray());
        ObjectInputStream oi = new ObjectInputStream(bi);
        return (oi.readObject());
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Teacher getTeacher() {
        return teacher;
    }

    public void setTeacher(Teacher teacher) {
        this.teacher = teacher;
    }
}
```
```java
public class SerializableTest {

    @Test
    public void test() {
        long t1 = System.currentTimeMillis();
        Teacher p = new Teacher();
        p.setName("wangwu");
        Student s1 = new Student();
        s1.setName("zhangsan");
        s1.setTeacher(p);
        Student s2 = null;
        try {
            s2 = (Student) s1.deepClone();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        s2.getTeacher().setName("lisi");
        System.out.println("name=" + s1.getTeacher().getName()); // 学生1的教授不改变。
        long t2 = System.currentTimeMillis();
        System.out.println(t2-t1);
    }
}
```