AOP 的原理：

涉及到了 设计模式的 代理模式。

代理模式的实现呢，我们可以分为 静态代理 和 动态代理。

**静态代理**

静态代理，主要是 一个接口，一个接口实现类 和 实现该接口的 代理类。 这些类都是在编译时 就确定好了，所以叫做静态代理，缺点 在于 每次都要写 一个实现接口的 代理类。  因此 出现了 动态代理。

```java
public interface PersonDao { void savePerson();
}

public class PersonDaoImpl implements PersonDao {
  @Override public void savePerson() {
     System.out.println("save person");
  }
}

public class Transaction { void beginTransaction(){
        System.out.println("begin Transaction");
    } void commit(){
        System.out.println("commit");
    }
}
```

接下来编写静态代理类---实现PersonDao接口

```java
/** 静态代理类 */
public class PersonDaoProxy implements PersonDao{
    PersonDao personDao;
    Transaction transaction; 
    public PersonDaoProxy(PersonDao personDao, Transaction transaction) {
       this.personDao = personDao;
       this.transaction = transaction;
    }

   @Override
   public void savePerson() {
     this.transaction.beginTransaction();
     this.personDao.savePerson();
     this.transaction.commit();
  }
}
```

测试

```java
/** 测试静态代理  */
public class TestPersonProxy {
    @Test
    public void testSave(){
        PersonDao personDao = new PersonDaoImpl();
        Transaction transaction = new Transaction();
        PersonDaoProxy proxy = new PersonDaoProxy(personDao, transaction);
        proxy.savePerson();
    }
}
```



**动态代理**

动态代理 有 很多种框架。AspectJ、JBoss Aop、Spring Aop.  我们主要讨论 spring Aop。

AOP 实现有 JDK动态代理 和 Cglib 动态代理。spring AOP 这两种都是支持的。

JDK 代理

> 单纯不使用spring AOP 去实现 JDK 动态代理，那么就需要 存在一个被代理的接口 和 被代理接口的实现类， 再创建一个实现 InvocationHandler 接口的类，在 invoke 方法中定义 代理类需要做的行为。
>
> 通过调用 Proxy.newProxyInstance 就可以为 被代理接口 动态生成一个 class 代理类。之后 只要调用方法的时候，只使用 代理类去调用，不使用 实现类。就能完成动态代理。
>
> JDK 动态代理 生成的类 实现了 被代理接口，继承了 Proxy。由于java 只支持单继承，因此在 继承Proxy 之后，只能选择 实现 被代理接口，这也是为什么 JDK代理必须要求提供 被代理接口 的原因。

 代码实现：

```java
import java.lang.reflect.InvocationHandler; 
import java.lang.reflect.Method; 
 /** 拦截器 
  *  1、目标类导入进来 
  *  2、事务导入进来 
  *  3、invoke完成：开启事务、调用目标对象的方法、事务提交
  */
public class Interceptor implements InvocationHandler {
    private Object target; // 目标类
    private Transaction transaction;
    public Interceptor(Object target, Transaction transaction) {
       this.target = target; this.transaction = transaction;
    }
    /*** @param proxy 目标对象的代理类实例
      * @param method 对应于在代理实例上调用接口方法的Method实例
      * @param args 传入到代理实例上方法参数值的对象数组
      * @return 方法的返回值，没有返回值是null
      * @throws Throwable
      */
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String methodName = method.getName(); 
        if ("savePerson".equals(methodName) || "deletePerson".equals(methodName) || "updatePerson".equals(methodName)) {
            this.transaction.beginTransaction(); // 开启事务
            method.invoke(target); // 调用目标方法
            this.transaction.commit(); // 提交事务
        } else {
            method.invoke(target);
        }
        return null;
    }
}
```

**测试jdk动态代理  **

```java
public class TestJDKProxy {
  @Test
  public void testSave(){ 
     /** 1、创建一个目标对象
      * 2、创建一个事务
      * 3、创建一个拦截器
      * 4、动态产生一个代理对象 */
    Object target = new PersonDaoImpl();
    Transaction transaction = new Transaction();
    Interceptor interceptor = new Interceptor(target, transaction); 
    /** 参数一：设置代码使用的类加载器，一般采用跟目标类相同的类加载器
      * 参数二：设置代理类实现的接口，跟目标类使用相同的接口
      * 参数三：设置回调对象，当代理对象的方法被调用时，会调用该参数指定对象的invoke方法 */
    PersonDao personDao = (PersonDao) Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            interceptor);
    personDao.savePerson();
   }
}
```



Cglib代理

> 单纯不使用spring AOP 去实现 Cglib 动态代理，那么就需要 存在一个被代理的类， 再创建一个实现 MethodInterceptor 接口的类，在 intercept 方法中定义 代理类需要做的行为。
>
> 通过调用 Enhancer类 进行代码增强，就可以为 被代理类 动态生成一个 class 代理类。之后 只要调用方法的时候，只使用 代理类去调用，不使用 实现类。就能完成动态代理。

```java
import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy; 
/** CGLIB代理 拦截器 */
public class Interceptor  implements MethodInterceptor {
    private Object target; // 代理的目标类
    private Transaction transaction;
    public Interceptor(Object target, Transaction transaction) {
       this.target = target;
       this.transaction = transaction;
    } 
    
    /** 创建目标对象的代理对象
      * 
      * @return
     */
    public Object createProxy() { // 代码增强
        Enhancer enhancer = new Enhancer(); // 该类用于生成代理对象
        enhancer.setCallback(this); // 参数为拦截器
        enhancer.setSuperclass(target.getClass());// 设置父类
        return enhancer.create(); // 创建代理对象
    }
    
    /** @param obj 目标对象代理类的实例
      * @param method 代理实例上 调用父类方法的Method实例
      * @param args 传入到代理实例上方法参数值的对象数组
      * @param methodProxy 使用它调用父类的方法
      * @return
      * @throws Throwable
      */
    public Object intercept(Object obj, Method method, Object[] args,
            MethodProxy methodProxy) throws Throwable {
            this.transaction.beginTransaction();
            method.invoke(target);
            this.transaction.commit();
            return null;
    }
}
```

测试

**测试cglib动态代理,通过cglib产生的代理对象，代理类是目标类的子类**

```java
public class TestCglibProxy {
    @Test
    public void testSave(){
        Object target = new PersonDaoImpl();
        Transaction transaction = new Transaction();
        Interceptor interceptor = new Interceptor(target, transaction);
        PersonDaoImpl personDaoImpl = (PersonDaoImpl) interceptor.createProxy();
        personDaoImpl.savePerson();
   }
}
```

两者的不同：

​    1\. 通过反射类`Proxy`和`InvocationHandler`回调接口实现的`JDK`动态代理，要求被代理类必须实现一个接口，但事实上并不是所有类都有接口，对于没有实现接口的类，便无法使用JDK动态代理。

​        CGLIB会让生成的代理类继承被代理类，并在代理类中对代理方法进行强化处理(前置处理、后置处理等)。

​    2.JDK动态代理，生成的代理类 是 实现了 被代理接口 的所有方法；

​       CGLIB 动态代理，生成的代理类是 对指定目标类产生一个子类，通过重写 所有父类方法的调用。

​    3\. Java 动态代理使用 Java 原生的反射 API 进行操作，在生成类上比较高效；

​       CGLIB 使用 ASM 框架直接对字节码进行操作，在类的执行过程中比较高效。

 Spring AOP：

​    1、若目标对象实现了若干接口，spring使用JDK的java.lang.reflect.Proxy类代理。

​    2、若目标对象没有实现任何接口，spring使用CGLIB库生成目标对象的子类。

​    只需要一个定义代理类所需要做的行为的 类。然后 可以选择 使用 xml 或者注解 两种形式 去定义 切面 和 切点。

​    代码1： 注解 + 定义一个 代理类所作行为的类

```java
@Aspect
public class Audience {

    
    /**
     * 目标方法执行之前调用
     */
    @Before("execution(** com.spring.aop.service.perform(..))")
    public void silenceCellPhone() {
        System.out.println("Silencing cell phones");
    }

    /**
     * 目标方法执行之前调用
     */
    @Before("execution(** com.spring.aop.service.perform(..))")
    public void takeSeats() {
        System.out.println("Taking seats");
    }

    /**
     * 目标方法执行完后调用
     */
    @AfterReturning("execution(** com.spring.aop.service.perform(..))")
    public void applause() {
        System.out.println("CLAP CLAP CLAP");
    }

    /**
     * 目标方法发生异常时调用
     */
    @AfterThrowing("execution(** com.spring.aop.service.perform(..))")
    public void demandRefund() {
        System.out.println("Demanding a refund");
    }
}
```

在spring 配置中，开启  自动代理功能：

方法1： 在Java Config 类上，添加 @EnableAspectJAutoProxy

方法2： 在 spring xml配置中，添加`<aop:aspectj-autoproxy/>`

代码2：

定义切面和切点

```xml
<aop:config>
   <!-- 切面 -->
   <aop:aspect id="myAspect" ref="aBean">
   <!-- 切点 -->
   <aop:pointcut id="businessService"
      expression="execution(* com.xyz.myapp.service.*.*(..))"/>  
   <!-- 环绕通知,在切点上执行切面bean的watchPerformance方法 -->
   <aop:around pointcut-ref="businessService" 
         method="watchPerformance"/>

   ...
   </aop:aspect>
</aop:config>

<!-- 切面: 在切点上执行相关代理行为的类 的bean -->
<bean id="aBean" class="..."> 
```

定义一个abean，实现 watchPerformance方法

```java
public void watchPerformance(ProceedingJoinPoint jp) {
    try {
        System.out.println("Silencing cell phones");
        System.out.println("Taking seats");
        jp.proceed();
        System.out.println("CLAP CLAP CLAP!!!");
    } catch (Throwable e) {
        System.out.println("Demanding a refund");
    }
}
```

**spring AOP原理**

　　1、当spring容器启动的时候，加载bean 进行实例化；  
　　2、当spring容器对配置文件解析到<aop:config>的时候，把切入点表达式解析出来，按照切入点表达式匹配spring容器内容的bean；  
　　3、如果bean 与 表达式 匹配成功，则为该bean创建代理对象；  
　　4、当客户端利用context.getBean获取一个对象时，如果该对象有代理对象，则返回代理对象，如果没有代理对象，则返回对象本身。