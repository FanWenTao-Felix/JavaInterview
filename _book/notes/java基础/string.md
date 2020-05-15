### **String，Stringbuffer，StringBuilder的区别？**

> String类是不可变的，所谓不可变：创建一个String类之后，任何对String的改变都会引发新的String对象的生成 
>
> StringBuffer不同于String的是StringBuffer是可变的 
>
> StringBuffer和StringBuilder类的原理和操作基本相同，区别在于StringBuffer支持并发操作，线性安全的，适合多线程中使用。StringBuilder不支持并发操作，线性不安全的，不适合多线程中使用。
>
> ​	在class文件中有一部分来存储编译期间生成的字面常量以及符号引用，这部分叫做class文件常量池，在运行期间对应着方法区的运行时常量池。 
>
> ​	String str1 = "hello world";和String str3 = "hello world"; 都在编译期间生成了 字面常量和符号引用，运行期间字面常量"hello world"被存储在运行时常量池（当然只保存了一份）。通过这种方式来将String对象跟引用绑定的话，JVM执行引擎会先在运行时常量池查找是否存在相同的字面常量，如果存在，则直接将引用指向已经存在的字面常量；否则在运行时常量池开辟一个空间来存储该字面常量，并将引用指向该字面常量。 
>
> ​	通过new关键字来生成对象是在堆区进行的，而在堆区进行对象生成的过程是不会去检测该对象是否已经存在的。因此通过new来创建对象，创建出的一定是不同的对象，即使字符串的内容是相同的。 

​	下面这段代码的输出结果是什么？

> > String a = "hello2"; 　　String b = "hello" + 2; 　　System.out.println((a == b));
>
> 输出结果为：true。
>
> 原因很简单，"hello"+2在编译期间就已经被优化成"hello2"，因此在运行期间，变量a和变量b指向的是同一个对象。

​	下面这段代码的输出结果是什么？

> > String a = "hello2"; 　 String b = "hello";    String c = b + 2;    System.out.println((a == c));
>
> 输出结果为:false。
>
> 由于有符号引用的存在，所以 String c = b + 2;不会在编译期间被优化，不会把b+2当做字面常量来处理的，因此这种方式生成的对象事实上是保存在堆上的。因此a和c指向的并不是同一个对象。

​	下面这段代码的输出结果是什么？

> > String a = "hello2";  　 final String b = "hello";    String c = b + 2;    System.out.println((a == c));
>
> 输出结果为：true。对于被final修饰的变量，会在class文件常量池中保存一个副本，也就是说不会通过连接而进行访问，对final变量的访问在编译期间都会直接被替代为真实的值。那么String c = b + 2;在编译期间就会被优化成：String c = "hello" + 2;

​	下面这段代码输出结果为：

> ```java
> public class Main {
> 
>  public static void main(String[] args) {
>      String a = "hello2";
>      final String b = getHello();
>      String c = b + 2;
>      System.out.println((a == c));
>  }
> 
>  public static String getHello() {
>      return "hello";
>  }
> 
> }
> ```
>
> ​	输出结果为false。这里面虽然将b用final修饰了，但是由于其赋值是通过方法调用返回的，那么它的值只能在运行期间确定，因此a和c指向的不是同一个对象。

​	下面这段代码的输出结果是什么？

> ```java
> public class Main {
> 
>  public static void main(String[] args) {
>      String a = "hello";//存放到常量池
>      String b =  new String("hello");//在堆分配空间,常量池已存在该常量,不再进行分配
>      String c =  new String("hello");//在堆分配空间,常量池已存在该常量,不再进行分配
>      String d = b.intern();//直接取常量池的引用
>       
>      System.out.println(a==b);//false
>      System.out.println(b==c);//fasle
>      System.out.println(b==d);//fasle
>      System.out.println(a==d);//true
>  }
> }
> ```
>
> 这里面涉及到的是String.intern方法的使用。在String类中，intern方法是一个本地方法，在JAVA SE6之前，intern方法会在运行时常量池中查找是否存在内容相同的字符串，如果存在则返回指向该字符串的引用，如果不存在，则会将该字符串入池，并返回一个指向该字符串的引用。因此，a和d指向的是同一个对象。