# JDK动态代理实现机制

http://www.atatech.org/articles/57626

Java中动态代理的实现，关键就是这两个东西：Proxy、InvocationHandler，下面从InvocationHandler接口中的invoke方法入手，简单说明一下Java如何实现动态代理的。 
​        首先，invoke方法的完整形式如下： 

```
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable  
    {  

        method.invoke(obj, args);  

        return null;  
    }  


```

首先猜测一下，method是调用的方法，即需要执行的方法；args是方法的参数；proxy，这个参数是什么？以上invoke()方法的实现即是比较标准的形式，我们看到，这里并没有用到proxy参数。查看JDK文档中Proxy的说明，如下： 

```
A method invocation on a proxy instance through one of its proxy interfaces will be dispatched to the invoke method of the instance's invocation handler, passing the proxy instance,a java.lang.reflect.Method object identifying the method that was invoked, and an array of type Object containing the arguments. 

```

由此可以知道以上的猜测是正确的，同时也知道，proxy参数传递的即是代理类的实例。 

为了方便说明，这里写一个简单的例子来实现动态代理。 

```
//抽象角色（动态代理只能代理接口）  
public interface Subject {  

    public void request();  
}  

```

```
//真实角色：实现了Subject的request()方法  
public class RealSubject implements Subject{  

    public void request(){  
        System.out.println("From real subject.");  
    }  
}  

```

```
//实现了InvocationHandler  
public class DynamicSubject implements InvocationHandler  
{  
    private Object obj;//这是动态代理的好处，被封装的对象是Object类型，接受任意类型的对象  

    public DynamicSubject()  
    {  
    }  

    public DynamicSubject(Object obj)  
    {  
        this.obj = obj;  
    }  

    //这个方法不是我们显示的去调用  
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable  
    {  
        System.out.println("before calling " + method);  

        method.invoke(obj, args);  

        System.out.println("after calling " + method);  

        return null;  
    }  

}  

```

```
//客户端：生成代理实例，并调用了request()方法  
public class Client {  
    public static void main(String[] args) throws Throwable{  
        //这里指定被代理类 
        Subject rs=new RealSubject(); 
        Class<?> cls=rs.getClass();  
        InvocationHandler ds=new DynamicSubject(rs);  

        //以下是一次性生成代理 
        Subject subject=(Subject) Proxy.newProxyInstance(  
                cls.getClassLoader(),cls.getInterfaces(), ds);  
        //这里可以通过运行结果证明subject是Proxy的一个实例，这个实例实现了Subject接口  
        System.out.println(subject instanceof Proxy);  

        //这里可以看出subject的Class类是$Proxy0,这个$Proxy0类继承了Proxy，实现了Subject接口  
        System.out.println("subject的Class类是："+subject.getClass().toString());  

        System.out.print("subject中的属性有：");  

        Field[] field=subject.getClass().getDeclaredFields();  
        for(Field f:field){  
            System.out.print(f.getName()+", ");  
        }  

        System.out.print("\n"+"subject中的方法有：");  

        Method[] method=subject.getClass().getDeclaredMethods();  

        for(Method m:method){  
            System.out.print(m.getName()+", ");  
        }  

        System.out.println("\n"+"subject的父类是："+subject.getClass().getSuperclass());  

        System.out.print("\n"+"subject实现的接口是：");  

        Class<?>[] interfaces=subject.getClass().getInterfaces();  

        for(Class<?> i:interfaces){  
            System.out.print(i.getName()+", ");  
        }  

        System.out.println("\n\n"+"运行结果为：");  
        subject.request();  
    }  
}  

```

```
运行结果如下：此处省略了包名，*<strong>代替  
true  
subject的Class类是：class $Proxy0  
subject中的属性有：m1, m3, m0, m2,   
subject中的方法有：request, hashCode, equals, toString,   
subject的父类是：class java.lang.reflect.Proxy  
subject实现的接口是：cn.edu.ustc.dynamicproxy.Subject,   
运行结果为：  
before calling public abstract void </strong>*.Subject.request()  
From real subject.  
after calling public abstract void *<strong>.Subject.request()  


```

PS：这个结果的信息非常重要，至少对我来说。因为我在动态代理犯晕的根源就在于将上面的subject.request()理解错了，至少是被表面所迷惑，没有发现这个subject和Proxy之间的联系，一度纠结于最后调用的这个request()是怎么和invoke()联系上的，而invoke又是怎么知道request存在的。其实上面的true和class Proxy0就能解决很多的疑问，再加上下面将要说的

Proxy0的源码，完全可以解决动态代理的疑惑了。**

从以上代码和结果可以看出，我们并没有显示的调用invoke()方法，但是这个方法确实执行了。下面就整个的过程进行分析一下： 

从Client中的代码看，可以从newProxyInstance这个方法作为突破口，我们先来看一下Proxy类中newProxyInstance方法的源代码： 

```
public static Object newProxyInstance(ClassLoader loader,  
        Class<?>[] interfaces,  InvocationHandler h)  
throws IllegalArgumentException  
{  
    if (h == null) {  
        throw new NullPointerException();  
    }  

    /* 
     * Look up or generate the designated proxy class. 
     */  
    Class cl = getProxyClass(loader, interfaces);  

    /* 
     * Invoke its constructor with the designated invocation handler. 
     */  
    try {  
           /* 
            * Proxy源码开始有这样的定义： 
            * private final static Class[] constructorParams = { InvocationHandler.class }; 
            * 获取Proxy里面的构造方法
            * cons即是形参为InvocationHandler类型的构造方法 
           */
        Constructor cons = cl.getConstructor(constructorParams);  
        return (Object) cons.newInstance(new Object[] { h });  
    } catch (NoSuchMethodException e) {  
        throw new InternalError(e.toString());  
    } catch (IllegalAccessException e) {  
        throw new InternalError(e.toString());  
    } catch (InstantiationException e) {  
        throw new InternalError(e.toString());  
    } catch (InvocationTargetException e) {  
        throw new InternalError(e.toString());  
    }  
}  


```

Proxy.newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)做了以下几件事:
​    （1）根据参数loader和interfaces调用方法 getProxyClass(loader, interfaces)创建代理类Proxy0.

Proxy0并在构造方法中把DynamicSubject传过去,接着

Proxy0调用父类Proxy的构造器,为h赋值,如下：

```
class Proxy{  
    InvocationHandler h=null;  
    protected Proxy(InvocationHandler h) {  
        this.h = h;  
    }  
    ...  
}  

```

来看一下这个继承了Proxy的$Proxy0的源代码： 

```
public final class $Proxy0 extends Proxy implements Subject {  
    private static Method m1;  
    private static Method m0;  
    private static Method m3;  
    private static Method m2;  

    static {  
        try {  
            m1 = Class.forName("java.lang.Object").getMethod("equals",  
                    new Class[] { Class.forName("java.lang.Object") });  

            m0 = Class.forName("java.lang.Object").getMethod("hashCode",  
                    new Class[0]);  

            m3 = Class.forName("*<strong>.RealSubject").getMethod("request",  
                    new Class[0]);  

            m2 = Class.forName("java.lang.Object").getMethod("toString",  
                    new Class[0]);  

        } catch (NoSuchMethodException nosuchmethodexception) {  
            throw new NoSuchMethodError(nosuchmethodexception.getMessage());  
        } catch (ClassNotFoundException classnotfoundexception) {  
            throw new NoClassDefFoundError(classnotfoundexception.getMessage());  
        }  
    } //static  

    public $Proxy0(InvocationHandler invocationhandler) {  
        super(invocationhandler);  
    }  

    @Override  
    public final boolean equals(Object obj) {  
        try {  
            return ((Boolean) super.h.invoke(this, m1, new Object[] { obj })) .booleanValue();  
        } catch (Throwable throwable) {  
            throw new UndeclaredThrowableException(throwable);  
        }  
    }  

    @Override  
    public final int hashCode() {  
        try {  
            return ((Integer) super.h.invoke(this, m0, null)).intValue();  
        } catch (Throwable throwable) {  
            throw new UndeclaredThrowableException(throwable);  
        }  
    }  

    public final void request() {  
        try {  
            super.h.invoke(this, m3, null);  
            return;  
        } catch (Error e) {  
        } catch (Throwable throwable) {  
            throw new UndeclaredThrowableException(throwable);  
        }  
    }  

    @Override  
    public final String toString() {  
        try {  
            return (String) super.h.invoke(this, m2, null);  
        } catch (Throwable throwable) {  
            throw new UndeclaredThrowableException(throwable);  
        }  
    }  
}  

```

接着把得到的\$Proxy0实例强制转换成Subject，并将引用赋给subject。当执行subject.request()方法时，就调用了$Proxy0类中的request()方法，进而调用父类Proxy中的h的invoke()方法.即InvocationHandler.invoke()。 

PS：1、需要说明的一点是，Proxy类中getProxyClass方法返回的是Proxy的Class类。之所以说明，是因为我一开始犯了个低级错误，以为返回的是“被代理类的Class类”--！
​    2、从$Proxy0的源码可以看出，动态代理类不仅代理了显示定义的接口中的方法，而且还代理了java的根类Object中的继承而来的equals()、hashcode()、toString()这三个方法，并且仅此三个方法。** 

下面分析一下$Proxy0的源码是怎么生成的，咱们从最关键的getProxyClass0方法开始分析：

```
private static Class<?> getProxyClass0(ClassLoader loader,  
                                           Class<?>... interfaces) {  
        if (interfaces.length > 65535) {  
            throw new IllegalArgumentException("interface limit exceeded");  
        }  

        // If the proxy class defined by the given loader implementing  
        // the given interfaces exists, this will simply return the cached copy;  
        // otherwise, it will create the proxy class via the ProxyClassFactory  
        return proxyClassCache.get(loader, interfaces);  
    }  

```

这里用到了缓存，先从缓存里查一下，如果存在，直接返回，不存在就新创建。在这个get方法里，我们看到了如下代码：

```
Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));

```

此处提到了apply()，是Proxy类的内部类ProxyClassFactory实现其接口的一个方法，具体实现如下：

```
public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {  

            Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);  
            for (Class<?> intf : interfaces) {  
                /* 
                 * Verify that the class loader resolves the name of this 
                 * interface to the same Class object. 
                 */  
                Class<?> interfaceClass = null;  
                try {  
                    interfaceClass = Class.forName(intf.getName(), false, loader);  
                } catch (ClassNotFoundException e) {  
                }  
                if (interfaceClass != intf) {  
                    throw new IllegalArgumentException(  
                        intf + " is not visible from class loader");  
                }...  

```

看到Class.forName()的时候，我想大多数人会笑了，终于看到熟悉的方法了，没错！这个地方就是要加载指定的接口，既然是生成类，那就要有对应的class字节码，我们继续往下看：

```
/* 
 * Generate the specified proxy class. 
 */  
   byte[] proxyClassFile = ProxyGenerator.generateProxyClass(  
   proxyName, interfaces, accessFlags);  
    try {  
          return defineClass0(loader, proxyName,  
          proxyClassFile, 0, proxyClassFile.length);  

```

这段代码就是利用ProxyGenerator为我们生成了最终代理类的字节码文件，即getProxyClass0()方法的最终返回值。因为有了上面的生成字节码的代码，那我们可以模仿这一步，自己生成字节码文件看看，所以，我用如下代码，生成了这个最终的代理类。

```
package com.adam.java.basic;  

import java.io.FileOutputStream;  
import java.io.IOException;  
import sun.misc.ProxyGenerator;  

public class DynamicProxyTest {  

    public static void main(String[] args) {  
        UserService userService = new UserServiceImpl();  
        MyInvocationHandler invocationHandler = new MyInvocationHandler(  
                userService);  

        UserService proxy = (UserService) invocationHandler.getProxy();  
        proxy.add();  

        String path = "C:/$Proxy0.class";  
        byte[] classFile = ProxyGenerator.generateProxyClass("$Proxy0",  
                UserServiceImpl.class.getInterfaces());  
        FileOutputStream out = null;  

        try {  
            out = new FileOutputStream(path);  
            out.write(classFile);  
            out.flush();  
        } catch (Exception e) {  
            e.printStackTrace();  
        } finally {  
            try {  
                out.close();  
            } catch (IOException e) {  
                e.printStackTrace();  
            }  
        }  
    }  
}  

```

上面测试方法里的proxy.add()，此处的add()方法，就已经不是原始的UserService里的add()方法了，而是新生成的代理类的add()方法，我们将生成的$Proxy0.class文件用jd-gui打开，我去掉了一些代码，add()方法如下：

```
public final void add()  
    throws   
  {  
    try  
    {  
      this.h.invoke(this, m3, null);  
      return;  
    }  
    catch (Error|RuntimeException localError)  
    {  
      throw localError;  
    }  
    catch (Throwable localThrowable)  
    {  
      throw new UndeclaredThrowableException(localThrowable);  
    }  
  }  

```

核心就在于this.h.invoke(this. m3, null);此处的h是啥呢？我们看看这个类的类名：
public final class $Proxy0 extends Proxy implements UserService
不难发现，新生成的这个类，继承了Proxy类实现了UserService这个方法，而这个UserService就是我们指定的接口，所以，这里我们基本可以断定，JDK的动态代理，生成的新代理类就是继承了Proxy基类，实现了传入的接口的类。那这个h到底是啥呢？我们再看看这个新代理类，看看构造函数：

```
public $Proxy0(InvocationHandler paramInvocationHandler)  
    throws   
  {  
    super(paramInvocationHandler);  
  }  

```

构造函数里传入了一个InvocationHandler类型的参数，看到这里，我们就应该想到之前的一行代码：

```
return cons.newInstance(new Object[]{h}); 

```

这是newInstance方法的最后一句，传入的h，就是这里用到的h，也就是我们最初自己定义的MyInvocationHandler类的实例。所以，我们发现，其实最后调用的add()方法，其实调用的是MyInvocationHandler的invoke()方法。我们再来看一下这个方法，找一下m3的含义，继续看代理类的源码：

```
static  
  {  
    try  
    {  
      m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] { Class.forName("java.lang.Object") });  
      m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);  
      m3 = Class.forName("com.adam.java.basic.UserService").getMethod("add", new Class[0]);  
      m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);  
      return;  
    }  

```

惊喜的发现，原来这个m3，就是原接口的add()方法，看到这里，还有什么不明白的呢？我想2,3,4问题都应该迎刃而解了吧？我们继续，看看原始MyInvocationHandler里的invoke()方法：

```
    @Override  
    public Object invoke(Object proxy, Method method, Object[] args)  
            throws Throwable {  
        System.out.println("----- before -----");  
        Object result = method.invoke(target, args);  
        System.out.println("----- after -----");  
        return result;  
    }  

```

m3就是将要传入的method，所以，为什么先输出before，后输出after，到这里是不是全明白了呢？这，就是JDK的动态代理整个过程，不难吧？
最后，我稍微总结一下JDK动态代理的操作过程：
一个典型的动态代理创建对象过程可分为以下四个步骤：
1.通过实现InvocationHandler接口创建自己的调用处理器 IvocationHandler handler = new InvocationHandlerImpl(...);
2.通过为Proxy类指定ClassLoader对象和一组interface创建动态代理类
Class clazz = Proxy.getProxyClass0(classLoader,new Class[]{...});
3.通过反射机制获取动态代理类的构造函数，其参数类型是调用处理器接口类型
Constructor constructor = clazz.getConstructor(new Class[]{InvocationHandler.class});
4.通过构造函数创建代理类实例，此时需将调用处理器对象作为参数被传入
Interface Proxy = (Interface)constructor.newInstance(new Object);
为了简化对象创建过程，Proxy类中的newInstance方法封装了2~4，只需两步即可完成代理对象的创建。
生成的ProxySubject继承Proxy类实现Subject接口，实现的Subject的方法实际调用处理器的invoke方法，而invoke方法利用反射调用的是被代理对象的的方法（Object
 result=method.invoke(proxied,args)）