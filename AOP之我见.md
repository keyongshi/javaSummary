# AOP之我见

http://www.atatech.org/articles/57336

## 什么是AOP

AOP（Aspect-Oriented Programming）：面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。
听起来还是不太容易理解，面向切面编程，首先得有切面吧，切面又是什么呢？切面其实就是你对逻辑代码得一个关注点，比如说一个类里面得一个方法（可以是随便某个方法），可是为什么又要面向切面编程呢？这就要提到编码规范里面得面向修改关闭，面向扩展开放得原则了，举个栗子，话说你某天业务逻辑要修改了，需要修改你关注点的逻辑，但是你想了一下上面这个原则，你又不想直接修改代码，这时候就可以使用AOP这个技术了，如果类似的关注点在多个地方出现（比如说纪录日志的关注点），那AOP的用处就更大了，下面将通过示例代码具体讲解AOP。

## AOP示例

下面是一个关于吃饭的代码，首先声明了一个接口：

```
public interface Lunch {
    void eat(String menu);
}

```

下面是一个实现类

```
public class LunchImpl implements Lunch {

    @Override
    public void eat(String menu) {
        System.out.println("eat： " + menu);
    }
}

```

有一天需求更改需要增加以下逻辑：饭前要洗手，饭后要刷碗，而且一日三餐的接口都需要更改，以下分析以下通常的集中处理方式。

------

### 基本AOP

#### **1、修改原有代码**

```
public class LunchImpl implements Lunch {

    @Override
    public void eat(String menu) {
        before();
        System.out.println("eat! " + menu);
        after();
    }

    private void before() {
        System.out.println("Wash hand！");
    }

    private void after() {
        System.out.println("Brush the bowl！");
    }
}

```

这是最简单但是也是问题最多的方法，修改原有代码会代码不断的回归测试，冗余的代码，三餐的代码都要修改，每个类里面都要加上以上逻辑，所以这种方式被否决。

#### **2、静态代理**

```
public class LunchProxy implements Lunch {

    private LunchImpl lunchImpl;

    public LunchProxy(LunchImpl lunchImpl) {
        this.lunchImpl = lunchImpl;
    }

    @Override
    public void eat(String menu) {
        before();
        lunchImpl.eat(menu);
        after();
    }

    private void before() {
        System.out.println("Wash hand！");
    }

    private void after() {
        System.out.println("Brush the bowl！");
    }
}

```

调用代码

```
public class Client {

    public static void main(String[] args) {
        Lunch lunchProxy = new LunchProxy(new LunchImpl());
        lunchProxy.eat("Kung Pao Chicken");
    }
}

```

这种方式比前一种要好一点，没有修改原有的代码，但是还有一个问题就是你需要为三餐写Proxy，将来会有很多XXXProxy的代码，也不方便维护。

#### **3、JDK动态代理**

```
public class JDKDynamicProxy implements InvocationHandler {

    private Object target;

    public JDKDynamicProxy(Object target) {
        this.target = target;
    }

    @SuppressWarnings("unchecked")
    public <T> T getProxy() {
        return (T) Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            this
        );
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        before();
        Object result = method.invoke(target, args);
        after();
        return result;
    }

    private void before() {
        System.out.println("Wash hand！");
    }

    private void after() {
        System.out.println("Brush the bowl！");
    }
}

```

调用代码：

```
public class Client {

    public static void main(String[] args) {
        Lunch lunch = new JDKDynamicProxy(new LunchImpl()).getProxy();
        lunch.eat("Kung Pao Chicken");
    }
}

```

这样就能解决以上几种方案的问题，但是还有一个问题，JDK 提供的动态代理只能代理接口，不能代理没有接口的类。

#### **4、CGLib 动态代理**

CGLib能代理没有接口的类，弥补了JDK动态代理的不足。

```
public class CGLibDynamicProxy implements MethodInterceptor {

    private static CGLibDynamicProxy instance = new CGLibDynamicProxy();

    private CGLibDynamicProxy() {
    }

    public static CGLibDynamicProxy getInstance() {
        return instance;
    }

    @SuppressWarnings("unchecked")
    public <T> T getProxy(Class<T> cls) {
        return (T) Enhancer.create(cls, this);
    }

    @Override
    public Object intercept(Object target, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        before();
        Object result = proxy.invokeSuper(target, args);
        after();
        return result;
    }

    private void before() {
        System.out.println("Wash hand！");
    }

    private void after() {
        System.out.println("Brush the bowl！");
    }
}

```

调用代码：

```
public class Client {

    public static void main(String[] args) {
        Lunch lunch = CGLibDynamicProxy.getInstance().getProxy(LunchImpl.class);
        lunch.eat("Kung Pao Chicken");
    }
}

```

至此，几种基本的AOP的实现方式已经介绍完，下面介绍使用Spring实现AOP，它为我们做了很多，使用起来很简单。

------

### Spring的AOP框架

首先介绍一下Spring一些术语的定义：
先了解AOP的相关术语: 

> - 通知(Advice): 通知定义了切面是什么以及何时使用。描述了切面要完成的工作和何时需要执行这个工作。 
> - 连接点(Joinpoint): 程序能够应用通知的一个“时机”，这些“时机”就是连接点，例如方法被调用时、异常被抛出时等等。 
> - 切入点(Pointcut) 通知定义了切面要发生的“故事”和时间，那么切入点就定义了“故事”发生的地点，例如某个类或方法的名称，Spring中允许我们方便的用正则表达式来指定 
> - 切面(Aspect) 通知和切入点共同组成了切面：时间、地点和要发生的“故事” 
> - 引入(Introduction) 引入允许我们向现有的类添加新的方法和属性(Spring提供了一个方法注入的功能） 
> - 目标(Target) 即被通知的对象，如果没有AOP,那么它的逻辑将要交叉别的事务逻辑，有了AOP之后它可以只关注自己要做的事 
> - 代理(proxy) 应用通知的对象，详细内容参见设计模式里面的代理模式 
> - 织入(Weaving) 把切面应用到目标对象来创建新的代理对象的过程，织入一般发生在如下几个时机: (1)编译时：当一个类文件被编译时进行织入，这需要特殊的编译器才可以做的到，例如AspectJ的织入编译器 (2)类加载时：使用特殊的ClassLoader在目标类被加载到程序之前增强类的字节代码 (3)运行时：切面在运行的某个时刻被织入,SpringAOP就是以这种方式织入切面的，原理应该是使用了JDK和CGLIB的动态代理技术

下面介绍使用Spring AOP的几种方式，网上流行的几种使用方式如下：

1. 基于Spring AOP框架接口（手动创建Proxy）
2. 基于Spring AOP框架接口（自动创建Proxy）
3. Spring + AspectJ（基于注解：通过 AspectJ execution 表达式拦截方法）
4. Spring + AspectJ（基于配置）
5. Spring + AspectJ（基于注解：@annotation拦截方法）
6. Spring + AspectJ（引入增强）

#### **1、基于代理的AOP（手动创建Proxy）**

概念中的连接点和通知（因为不仅描述了时间而且也描述了要完成的工作），实现Spring的连接点要继承的几个接口如下：

- Before(前置通知)：org.apringframework.aop.MethodBeforeAdvice
- After-returning(返回通知 == final通知)：org.springframework.aop.AfterReturningAdvice
- After-throwing(抛出通知)：org.springframework.aop.ThrowsAdvice
- Introduction(引入通知)：org.springframework.aop.IntroductionInterceptor
- Around(环绕通知)：org.aopaliance.intercept.MethodInterceptor 

**注意：环绕通知是Spring使用AOP联盟的接口，环绕通知可以实现前置通知、后置通知、抛出通知。**

使用基于代码AOP有以下几个步骤：

1. 根据需要实现以上的接口，实现连接点和通知
2. 利用连接点和通知，设置切点，定义切面
3. 配置ProxyFactoryBean来生成代理

下面以环绕通知为例，阐述一下上面的步骤：
1、实现连接点和通知，以下以一个前置通知各环绕通知为示例：
前置通知：

```
public class LunchBeforeAdvice implements MethodBeforeAdvice {

    @Override
    public void before(Method method, Object[] args, Object target) throws Throwable {
        System.out.println("Wash Hand！");
    }
}

```

环绕通知：

```
public class LunchAroundInterceptor implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        before();
        Object result = invocation.proceed();
        after();
        return result;
    }

    private void before() {
        System.out.println("Wash Hand！");
    }

    private void after() {
        System.out.println("Brush the bowl！");
    }
}

```

2、接着配置切面，切面可以分为几类，分别是:

- Advice(org.aopalliance.aop.Advice):表示一个method执行前或执行后的动作。
- MethodInterceptor(org.aopalliance.intercept.MethodInterceptor):AOP联盟的方法拦截器（理论上拦截到方法之后可以对切点随便处理）。
- Advisor(org.springframework.aop.Advisor):Advice和Pointcut组成的独立的单元，并且能够传给proxy factory 对象。

**以上三者的区别：前两者只包含通知和连接点属性，它们直接作为切面的话，是无法定义切入点的，默认将会对整个被代理类(target)的所有方法作为切入点；而Advisor会定义切入点，可以设置全切面的三个元素。**
首先利用xml配置文件配置一个Advice：

```
<bean id="lunchBeforeAdvice" class="xxx.xxx.xxx.LunchBeforeAdvice"/>

```

配置一个MethodInterceptor:

```
<bean id="lunchAroundInterceptor" class="xxx.xxx.xxx.LunchAroundInterceptor"/>

```

其实也就是把上面实现的代码声明成了Spring的bean而已。

利用Advice和MethodInterceptor配置Advisor，Advisor有很多种，下面只是其中两种：
使用Pointcut切点来定义Advisor：

```
     <!-- 定义切入点 匹配所有的eat方法-->  
    <bean id ="eatPointcut" class="org.springframework.aop.support.JdkRegexpMethodPointcut">  
           <property name="pattern" value=".*eat"/> 
    </bean>  

    <!-- 切面 通知+连接点+结合 -->  
    <bean id="lunchAdvisor" class="org.springframework.aop.support.DefaultPointcutAdvisor">  
         <!--通知+连接点-->
         <property name="advice" ref="lunchBeforeAdvice"/>
         <!--切点-->
         <property name="pointcut" ref="eatPointcut"/>  
    </bean>  

```

使用正则来定义Advisor：

```
<!--切面-->
<bean id="lunchAdvisor" class="org.springframework.aop.support.RegexpMethodPointcutAdvisor">       
        <!--通知+连接点-->
        <property name="advice" ref="lunchBeforeAdvice"/>                  <!--相当于切点-->
        <property name="pattern" value=".*eat"/>  
    </bean>  

```

解释一下上面的Advisor，它设置了一个正则表达式作为切点，除此之外也可以使用NameMatchMethodPointcutAdvisor指定方法名来拦截。

**注意advice也可以设置为lunchAroundInterceptor。因为MethodInterceptor继承了Advice接口**
3、配置ProxyFactoryBean来生成代理
关于这个类的详细介绍可以查看这个链接：[使用ProxyFactoryBean创建AOP代理](http://www.cnblogs.com/wasp520/p/3143139.html)

```
<bean id="proxyBean"
        class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="proxyInterfaces" value="xxx.xxx.xxx.Lunch" />
        <property name="target">
            <bean class="xxx.xxx.xxx.LunchImpl" />
        </property>
        <property name="interceptorNames">
            <list>
                <value>lunchBeforeAdvice</value>
                <value>lunchAdvisor</value>
                <value>lunchAroundInterceptor</value>
            </list>
        </property>
    </bean>

```

**注意：阅读ProxyFactoryBean源码得知interceptorNames属性可以注入继承MethodInterceptor,Advisor,Advice这些接口的类(MethodInterceptor继承了Advice接口)**

完成这些配置，当通过Spring获得lunchImpl，调用它的方法时候就能看到aop的效果了。

另外附上AOP一些基本类的关系图供参考：
[![spring-aop-class-relation](http://img2.tbcdn.cn/L1/461/1/a43587ee9ebfb09fb42882c0098a1a5c317479e6)](http://img2.tbcdn.cn/L1/461/1/a43587ee9ebfb09fb42882c0098a1a5c317479e6)

#### **基于代理的AOP（自动创建Proxy）**

采用org.springframework.aop.framework.ProxyFactoryBean的配置，每次需要增加一个target类（被代理的类）都需要在proxyInterfaces和target中增加接口及其实现，当target很多的时候会很很难梳理。
下面将会介绍两种自动创建代理的配置，采用这种配置策略，完全可以避免增量式配置，所有的事务代理由系统自动创建。
这种配置方式依赖于Spring提供的bean后处理器，该后处理器用于为每个bean自动创建代理，此处的代理不仅可以是事务代理，也可以是任意的代理，只需要有合适的拦截器即可。

1. 采用BeanNameAutoProxyCreator配置代理

   ```
   <bean  class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">
   <property name="beanNames">
       <list>
           <value>*Impl</value>
       </list>
   </property>
   <property name="interceptorNames">
       <list>
           <value>lunchBeforeAdvice</value>
           <value>lunchAdvisor</value>
           <value>lunchAroundInterceptor</value>
       </list> 
   </property>
   </bean>

   ```

   beanNames属性就是配置被代理的target类，因为支持正则，例如以上配置会主动为所有beanName以Impl结尾的类代理。

2. 采用DefaultAdvisorAutoProxyCreator配置代理

   ```
   <bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/> 

   ```

   DefaultAdvisorAutoProxyCreator这个类功能更为强大，这个类的奇妙之处是他实现了BeanProcessor接口,当ApplicationContext读如所有的Bean配置信息后，这个类将扫描上下文，**寻找所有的Advistor(一个Advisor是一个切入点和一个通知的组成)**，将这些Advisor应用到所有符合切入点的Bean中。
   **注意：这些必须是advisor而不仅仅是拦截器或者其它通知。这点是必要的因为必须有一个切入点被评估，以便检查每个通知候选bean定义的合适性。**

**以上两个类因为都继承自ProxyConfig类，都含有两个参数分别是proxyTargetClass，这个参数是boolean类型的，默认是false，会对所有类生成JDK代理，如果该类没有接口则报错，如果设置为true，则会对没有实现接口的类生成CGLIB代理。**

如果以上两者不符合你的要求，你可以到org.springframework.aop.framework.autoproxy包里面找合适的代理类创建器，你也可以通过继承AbstractAdvisorAutoProxyCreator自己定制。

下面是类图供参考：
[![spring-aop-class-relation](http://img1.tbcdn.cn/L1/461/1/8b3920057c3ef41bf8a6113500d5e54bdf7517d3)](http://img1.tbcdn.cn/L1/461/1/8b3920057c3ef41bf8a6113500d5e54bdf7517d3)

### http://www.atatech.org/articles/57357

#### **Spring + AspectJ（基于注解：通过 AspectJ execution 表达式拦截方法）**

Spring提供了注解来实现AOP：

```
//声明切面
@Aspect
//声明组件bean
@Component
public class LunchAspect {  
    //配置切入点,该方法无方法体,主要为方便同类中其他方法使用此处配置的切入点
    @Pointcut("execution(* xxx.xxx.xxx.eat(..))")
    public void eatpoint(){}
    /*
     * 配置前置通知,使用在方法aspect()上注册的切入点
     * 同时接受JoinPoint切入点对象,可以没有该参数
     */
    @Before("eat()")
    public void beforeEat(){
        System.out.println("Wash Hand！");
    }

    //配置后置返回通知,使用在方法aspect()上注册的切入点
    @AfterReturning("eat()")
    public void afterEat(){
        System.out.println("Brush the bowl！");
    }
}

```

一个@Aspect其实就相当于一个Advisor，因为它包含了通知（做什么），连接点（时间），切点（地点）的结合，这种方式使用了注解并采用了**AspectJ切点表达式**，
重点是这个切入点表达式execution (* xxx.xxx.xxx.impl..*.eat(..))
整个表达式可以分为五个部分,其语法如下所示：
 1、execution(): 表达式主体。
 2、第一个*号：表示返回类型，*号表示所有的类型。
 3、包名：表示需要拦截的包名，后面的两个句点表示当前包和当前包的所有子包，com.sample.service.impl包、子孙包下所有类的方法。
 4、第二个*号：表示类名，*号表示所有的类。
 5、eat(..):eat表示方法名，后面括弧里面表示方法的参数，两个句点表示任何参数。
具体切点表达式的语法读者可以参考[AspectJ的Execution表达式](http://blog.csdn.net/yakoo5/article/details/17001381)；

注意：**使用@Around注解可以创建环绕通知，被环绕通知的方法必须接受一个ProceedingJoinPoint对象作为方法入参，并在对象上调用proceed()方法。**

除此之外，需要新增一下spring配置：

```
        <!--扫描包 -->
       <context:component-scan base-package="com.tgb" annotation-config="true"/> 
       <!-- 激活自动代理功能 -->
       <aop:aspectj-autoproxy  proxy-target-class="true" />  

```

通过激活自动代理的功能，定义的aspect（切面）会自动根据execution表达式匹配的类生成代理然后切入。

#### **Spring + AspectJ（基于配置）**

配置形式和注解形式是一一对应的，本质上是等价的，下面通过配置说明：

```
    <!-- 系统服务组件的切面Bean -->
    <bean id="lunchAspect" class="xxx.xxx.xxx.LunchAspect"/>
    <!-- AOP配置 -->
    <aop:config>
        <!-- 声明一个切面,并注入切面Bean,相当于@Aspect -->
        <aop:aspect id="simpleAspect" ref="lunchAspect">
            <!-- 配置一个切入点,相当于@Pointcut -->
            <aop:pointcut expression="execution(* *.eat(..))" id="simplePointcut"/>
            <!-- 配置通知,相当于@Before、@After、@AfterReturn、@Around、@AfterThrowing -->
            <aop:before pointcut-ref="simplePointcut" method="before"/>
            <aop:after pointcut-ref="simplePointcut" method="after"/>
            <aop:after-returning pointcut-ref="simplePointcut" method="afterReturn"/>
            <aop:after-throwing pointcut-ref="simplePointcut" method="afterThrow" throwing="ex"/>
        </aop:aspect>
    </aop:config>

```

相当于把LunchAspect一个普通类通过配置变成了一个切面，此时的LunchAspect类里面只需要before,after,afterReturn,afterThrow方法就行了，切面配置的东西都放在了xml配置文件里面了。

#### **Spring + AspectJ（基于注解：@annotation拦截方法）**

本人觉得这种方式算是最有用的一种方式了，你可以利用这种方式自定义注解，并拦截被标记注解的方法进行处理，下面介绍一下使用方式：
首先你需要定义一个注解：

```
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Flag {
}

```

这个注解可以标记在方法上，在运行时生效。
接着对之前的LunchAspect类的AspectJ切点表达式进行修改，

```
@Pointcut("@annotation(xxx.xxx.Flag)")

```

更改为@annotation()表达式，这种方式只需在括号内定义需要拦截的注解名称。

然后在LunchImpl的eat方法上加上这个注解，

```
...
@Flag
public void eat(String menu) {
        System.out.println("eat： " + menu);
}
...

```

如此就可以自定义自己的注解并在切面里自定义处理方法了，使用这种方式对于统计接口响应时间、失败率等一些“杂活”很有用。

#### **Spring + AspectJ（引入增强）**

“引入”切面可以为Spring Bean添加新的方法，切面只是实现了它们所包装Bean的相同接口的代理，**“引入”让代理还能发布新的接口，这样切面所通知的Bean看起来实现了新的接口，即便底层实现类并没有实现这些接口。**
首先我们定义一个Aspect类：

```
@Aspect
@Component
public class LunchAspect {
    // 我们引入的新接口，也就是为LunchImpl动态新增的接口
    @DeclareParents(value = "aop.demo.LunchImpl", defaultImpl = SleepImpl.class)
    private Sleep sleep;
}

```

我们在LunchAspect类中引入了一个新接口，也就是给LunchImpl增强功能的接口，也是运行时需要动态实现的接口。在这个接口上标注了 @DeclareParents 注解，该注解有两个属性：
value：目标类
defaultImpl：引入接口的默认实现类

下面实现SleepImpl

```
public class SleepImpl implements Sleep {

    @Override
    public void nap(String name) {
        System.out.println(name + "！ Take a nap！");
    }
}

```

以上的实现会在运行的时候自动增强到LunchImpl中，也就是无需修改LunchImpl的源代码，就给它新增加了一个接口并附带了实现。

下面看一下调用代码：

```
public class Client {

    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("xxx/xxx/xxx/xxx.xml");
        Lunch lunch = (Lunch) context.getBean("lunchImpl");
        lunch.eat("Kung Pao Chicken!");

        Sleep sleep = (Sleep) lunch; // 强制转型为 Sleep 接口
        sleep.nap("Ling You");
    }
}

```

从Spring上下文中获取lunchImpl对象（代理对象），可转型为自己静态实现的接口Lunch，也可转型为自己动态实现的接口Sleep，功能很强大。

下面附上一张从网上找到的思维导图：
[![AOP](http://img3.tbcdn.cn/L1/461/1/32d621ad5ab4b88b51966a573050d8d463b3ea2d)](http://img3.tbcdn.cn/L1/461/1/32d621ad5ab4b88b51966a573050d8d463b3ea2d)

AOP对比表格：

| 通知类型                     | 基于Spring AOP 框架接口                 | 基于AspectJ注解     | 基于aop:config        |
| ------------------------ | --------------------------------- | --------------- | ------------------- |
| Before Advice（前置增强）      | MethodBeforeAdvice                | @Before         | aop:before          |
| AfterAdvice（后置增强）        | AfterReturningAdvice              | @After          | aop:after           |
| AroundAdvice（环绕增强）       | MethodInterceptor                 | @Around         | aop:around          |
| ThrowsAdvice（抛出增强)       | ThrowsAdvice                      | @AfterThrowing  | aop:after-throwing  |
| IntroductionAdvice（引入增强） | DelegatingIntroductionInterceptor | @DeclareParents | aop:declare-parents |

通过这个表的对比，发现这四种方式是互通的。

我所知道的比较实用的AOP技术都在这里了，当然还有一些更为高级的特性，由于个人精力有限，这里就不再深入了。
另外由于本人水平有限，文中会有一些错误或不准确的地方，恳请指正！