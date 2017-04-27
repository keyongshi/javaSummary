# Java 代理的理解

http://www.atatech.org/articles/56651

## 什么是代理

所谓代理，就是一个人或者一个机构代表另一个人或者另一个机构采取行动。在一些情况下，一个客户不想或者不能够直接引用一个对象；而代理对象可以在客户端和目标对象之前起到中介的作用。代理模式给某一个对象提供一个代理对象，并由代理对象控制对原对象的引用。

### 代理的分类

按照代理的创建时期，代理类可以分为两种。 
**静态代理**：由程序员创建或特定工具自动生成源代码，再对其编译。在程序运行前，代理类的.class文件就已经存在了。 

**动态代理**：在程序运行时，运用反射机制动态创建而成。 

### 代理模式一般涉及的角色

**抽象角色**：声明真实对象和代理对象的共同接口； 

**代理角色**：代理对象角色内部含有对真实对象的引用，从而可以操作真实对象，同时代理对象提供与真实对象相同的接口以便在任何时刻都能代替真实对象。同时，代理对象可以在执行真实对象操作时，附加其他的操作，相当于对真实对象进行封装。 

**真实角色**：代理角色所代表的真实对象，是我们最终要引用的对象。

## Java静态代理

先从最简单的静态代理开始，举个例子：
*角色：抽象角色*

```
public interface Hello {
    void say(String name);
}

```

以下是这个接口的实现类：
*角色：真实角色*

```
public class HelloImpl implements Hello {

    @Override
    public void say(String name) {
        System.out.println("Hello! " + name);
    }
}

```

假如说这个是你业务实现上使用的一个实例实现类，忽然有一天业务需求发生了变化，需要在println语句前面或者后面需要加上一些业务逻辑，怎么办？如果直接写死在say方法里面是能解决这个问题，但是不符合咱们面向扩展开放面向修改封闭的原则。
这个时候你想到了代理，写一个HelloProxy类引用HelloImpl的实例，然后调用它的say方法，在Proxy类里面实现println前后需要增加的业务逻辑。
*角色：真实角色*

```
public class HelloProxy implements Hello {

    private HelloImpl helloImpl;

    public HelloProxy() {
        helloImpl = new HelloImpl();
    }

    @Override
    public void say(String name) {
        before();
        helloImpl.say(name);
        after();
    }

    private void before() {
        System.out.println("Before");
    }

    private void after() {
        System.out.println("After");
    }
}

```

## Java动态代理

### JDK动态代理

忽然有一天你发现静态代理实现着也麻烦，因为每一个真是角色写一个相应的代理类，最后代码里全部都是Proxy的代码，不仅写着烦，代码的冗余度也很高，知道有一天你发现了JDK提供的原生的动态代理方案，我们只要实现JDK提供的接口（InvocationHandler）就行了。

```
public class DynamicProxy implements InvocationHandler {

    private Object target;

    public <T> T getProxy() {
        return (T) Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            this
        );
    }

    public DynamicProxy(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        before();
        Object result = method.invoke(target, args);
        after();
        return result;
    }

    ...
}

```

在上面这个动态代理里面有一个target属性，这个就是被代理的对象，通过新建对象时的构造函数初始化，也叫注入。

下面看看通过代码来看看应该如何使用JDK的动态代理：

```
public static void main(String[] args) {
    DynamicProxy dynamicProxy = new DynamicProxy(new HelloImpl());
    Hello helloProxy = dynamicProxy.getProxy();
    helloProxy.say("Jack");
}

```

在DynamicProxy.java代码中通过调用JDK给我们提供的Proxy静态类的工厂方法 newProxyInstance() 去动态地创建一个 Hello 接口的代理类，工厂方法中有三个参数，
​    \> 1、 真实角色的ClassLoader
​    \> 2、 真实角色实现实现的接口，也就是抽象角色
​    \> 3、 刚才我们实现的InvocationHandler，也就是动态代理对象

如果需求逻辑发生了变化，相比静态代理类，如果多个类需要增加同一部分逻辑（日志打印或者时间统计）只需要写一个InvocationHandler，而Proxy.newProxyInstance()逻辑不用更改就可以生成代理类，而静态代理需要为每个类写代理类。然而使用JDK动态代理是需要有第二个参数的，也就是***真实角色需要实现接口***，那有没有不需要实现接口就可以生成的动态代理的方法呢？经过网上一番搜寻，发现了这种方法——CGLIB类库。

### CGLIB动态代理

CGlib是一个强大的,高性能,高质量的Code生成类库。它可以在运行期扩展Java类与实现Java接口。然这些实际的功能是asm所提供的，asm又是什么？Java字节码操控框架，具体是什么大家可以上网查一查，毕竟我们这里所要讨论的是动态代理，cglib就是封装了asm，简化了asm的操作，实现了在运行期动态生成新的class。它通过字节码为原有类创建子类，并在子类中采用方法拦截的技术拦截所有父类方法的调用，顺势织入横切逻辑。JDK动态代理与CGLib动态代理均是实现Spring
 AOP的基础。
**\*注意：CGLIB是通过继承来生成的代理，对于声明为final类型的类是无能为力的***
下面看一下CGLIB动态代理的使用：

```
public class CGLibProxy implements MethodInterceptor {
    private Enhancer enhancer = new Enhancer();

    public <T> T getProxy(Class<T> cls) {
        enhancer.setSuperclass(cls); 
        enhancer.setCallback(this);
        return (T) enhancer.create();
    }

    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        // 织入逻辑before
        before();
        Object result = proxy.invokeSuper(obj, args);
        // 织入逻辑after
        after();
        return result;
    }

    ...
}

```

要使用CGLIB生成动态代理需要实现提供的MethodInterceptor接口，实现intercept方法，其中的Enhancer生成一个原有类的子类，并且设置好callback ， 则原有类的每个方法调用都会转成调用实现了MethodInterceptor接口的proxy的intercept() 函数，在intercept()函数里，你可以在执行Object result=proxy.invokeSuper(obj,args);来执行原有函数，在执行前后加入自己的东西，改变它的参数，也可以完全干别的。说白了，就是AOP中的around advice。

```
public static void main(String[] args) {
    CGLibProxy cgLibProxy = new CGLibProxy();
    HelloImpl helloProxy = cgLibProxy.getProxy(HelloImpl.class);

    helloProxy.say("Jack");
}

```

除此之外，CGLIB还提供了方法过滤器逻辑，可以选择性的拦截原有类的方法，假如说有业务需求需要仅对HelloImpl的say方法进行织入，其他方法都正常通过，可以通过以下方式实现；

```
public class accept implements CallbackFilter {
    public int accept(Method arg0) {
        // 如果是say方法的话则执行index为0的MethodInterceptor
        // 也就是CGLibProxy
        if("say".equalsIgnoreCase(arg0.getName())){
            return 0;
        }
        // 不为say的话返回index为1拦截器
        return 1;  
    }

}
```

另外在CGLibProxy的getProxy方法中需要修改 
enhancer.setCallbacks(new Callback[]{this,NoOp.INSTANCE}); 另外还要新增这个过滤器，即enhancer.setCallbackFilter(new AuthProxyFilter());
修改以上两行的原因是，enhancer的CallbackFilter的本质是根据方法选取过滤器的index，上面一句设置了两个CallBacks分别是this(CgLibProxy)和NoOp.INSTANCE，两者的index分别是0和1；accept方法判断是say方法的话就选取this（返回index-0）为拦截器，不是say的话则选取NoOp.INSTANCE(返回index-1)作为拦截器，也就什么都不做。

**\*注意事项：CGLib创建的动态代理对象性能比JDK创建的动态代理对象的性能高不少，但是CGLib在创建代理对象时所花费的时间却比JDK多得多，所以对于单例的对象，因为无需频繁创建对象，用CGLib合适，反之，使用JDK方式要更为合适一些。***