# 1.jdk7语法上

### 1.1 二进制变量的表示,支持将整数类型用二进制来表示，用0b开头。

```java
  // 所有整数 int， short，long，byte都可以用二进制表示
  // An 8-bit 'byte' value:
  byte aByte = (byte) 0b00100001;
  // A 16-bit 'short' value:
  short aShort = (short) 0b1010000101000101;
  // Some 32-bit 'int' values:
  intanInt1 = 0b10100001010001011010000101000101;
  intanInt2 = 0b101;
  intanInt3 = 0B101; // The B can be upper or lower case.
  // A 64-bit 'long' value. Note the "L" suffix:
  long aLong = 0b1010000101000101101000010100010110100001010001011010000101000101L;
  // 二进制在数组等的使用
  final int[] phases = { 0b00110001, 0b01100010, 0b11000100, 0b10001001,
  0b00010011, 0b00100110, 0b01001100, 0b10011000 };
```

### 1.2 Switch语句支持string类型

```java
public static String getTypeOfDayWithSwitchStatement(String dayOfWeekArg) {
     String typeOfDay;
     switch (dayOfWeekArg) {
       case "Monday":
         typeOfDay = "Start of work week";
         break;
       case "Tuesday":
       case "Wednesday":
       case "Thursday":
         typeOfDay = "Midweek";
         break;
       case "Friday":
         typeOfDay = "End of work week";
         break;
       case "Saturday":
       case "Sunday":
         typeOfDay = "Weekend";
         break;
       default:
         throw new IllegalArgumentException("Invalid day of the week: " + dayOfWeekArg);
     }
     return typeOfDay;
  } 
```

### ![](https://oscimg.oschina.net/oscnet/up-a12aab2c023a0f1eecba80c78bc756a5600.gif)1.3 Try-with-resource语句

​    注意：实现java.lang.AutoCloseable接口的资源都可以放到try中，跟 final 里面的关闭资源类似；

​    按照声明逆序关闭资源 ;

​    Try块抛出的异常通过Throwable.getSuppressed获取

```java
try (
  java.util.zip.ZipFile zf = new java.util.zip.ZipFile(zipFileName);
  java.io.BufferedWriter writer = java.nio.file.Files.newBufferedWriter(outputFilePath, charset)
) {
  // Enumerate each entry
  for (java.util.Enumeration entries = zf.entries(); entries.hasMoreElements();) {
    // Get the entry name and write it to the output file
    String newLine = System.getProperty("line.separator");
    String zipEntryName = ((java.util.zip.ZipEntry) entries.nextElement()).getName() + newLine;
    writer.write(zipEntryName, 0, zipEntryName.length());
  }
}
```

### ![](https://oscimg.oschina.net/oscnet/up-519d4337d1875975d7da2d71e63f86415c3.gif)![](https://oscimg.oschina.net/oscnet/up-2becdc8d13c442aa9389763a358d23f1f85.gif)1.4 Catch多个异常

​    说明：Catch异常类型为 final ;生成Bytecode 会比多个`catch`小; Rethrow时保持异常类型

```java
public static void main(String[] args) throws Exception {
	try {
		testthrows();
	} catch (IOException | SQLException ex) {
		throw ex;
	}
 }

public static void testthrows() throws IOException, SQLException {

}
```

### ![](https://oscimg.oschina.net/oscnet/up-82355047f16471735c05b785a435e8692ee.gif)1.5 数字类型的下划线表示

​    更友好的表示方式，不过要注意下划线添加的一些标准，可以参考下面的示例:

```java
  long creditCardNumber = 1234_5678_9012_3456L;
  long socialSecurityNumber = 999_99_9999L;
  float pi = 3.14_15F;
  long hexBytes = 0xFF_EC_DE_5E;
  long hexWords = 0xCAFE_BABE;
  long maxLong = 0x7fff_ffff_ffff_ffffL;
  byte nybbles = 0b0010_0101;
  long bytes = 0b11010010_01101001_10010100_10010010; 
  //float pi1 = 3_.1415F;   // Invalid; cannot put underscores adjacent to a decimal point
  //float pi2 = 3._1415F;   // Invalid; cannot put underscores adjacent to a decimal point
  //long socialSecurityNumber1= 999_99_9999_L;     // Invalid; cannot put underscores prior to an L suffix 
  //int x1 = _52;       // This is an identifier, not a numeric literal
  int x2 = 5_2;       // OK (decimal literal)
  //int x3 = 52_;       // Invalid; cannot put underscores at the end of a literal
  int x4 = 5_______2;    // OK (decimal literal) 
  //int x5 = 0_x52;      // Invalid; cannot put underscores in the 0x radix prefix
  //int x6 = 0x_52;      // Invalid; cannot put underscores at the beginning of a number
  int x7 = 0x5_2;      // OK (hexadecimal literal)
  //int x8 = 0x52_;      // Invalid; cannot put underscores at the end of a number 
  int x9 = 0_52;       // OK (octal literal)
  int x10 = 05_2;      // OK (octal literal)
  //int x11 = 052_;      // Invalid; cannot put underscores at the end of a number
```

### ![](https://oscimg.oschina.net/oscnet/up-101b415b1fec5024224da76865a4c293850.gif)1.6 泛型实例的创建可以通过类型推断来简化

​    可以去掉后面`new`部分的泛型类型，只用<>就可以了

```java
    //使用泛型前 
	List strList = new ArrayList(); 
	List strList4 = new ArrayList(); 
	List>> strList5 = new ArrayList>>();

	//编译器使用尖括号 (<>) 推断类型 
	List strList0 = new ArrayList(); 
	List>> strList1 = new ArrayList>>(); 
	List strList2 = new ArrayList<>(); 
	List>> strList3 = new ArrayList<>();
	List list = new ArrayList<>();
	list.add("A");
	// The following statement should fail since addAll expects
	// Collection
	//list.addAll(new ArrayList<>()); 
```

### ![](https://oscimg.oschina.net/oscnet/up-c3bf94aeb31607e953b885b0fc7094c9a9e.gif)1.7 在可变参数方法中传递非具体化参数,改进编译警告和错误.

​    Heap pollution 指一个变量被指向另外一个不是相同类型的变量。

​    JDK7中：

```java
        List l = new ArrayList<Number>();
        List<String> ls = l;       // unchecked warning
        l.add(0, new Integer(42)); // another unchecked warning
        String s = ls.get(0);      // ClassCastException is thrown

        public static <T > void addToList (List < T > listArg, T...elements){
            for (T x : elements) {
                listArg.add(x);
            }
        }
```

![](https://oscimg.oschina.net/oscnet/up-d116f33c89bae48c39dfac6a8a669b70a34.gif)![](https://oscimg.oschina.net/oscnet/up-c89d218c15ffd2fb507b62c202b76265ded.gif)    你会得到一个warning:

​    \[varargs\] Possible heap pollution from parameterized vararg type

​    要消除警告，可以有三种方式:

>         1.加 annotation @SafeVarargs
> 
>         2.加 annotation @SuppressWarnings({"unchecked", "varargs"})
> 
>         3.使用编译器参数 –Xlint:varargs;

### 1.8 信息更丰富的回溯追踪

​    上面try中try语句和里面的语句同时抛出异常时，异常栈的信息

![](https://oscimg.oschina.net/oscnet/up-ad83e973b09f6f19e3afac4fbfae16a2856.png)

# 2\. NIO2的一些新特性

### 1.java.nio.file 和java.nio.file.attribute包 支持更详细属性，比如权限，所有者

### 2.symbolic and hard links支持

### 3.Path访问文件系统，Files支持各种文件操作

### 4.高效的访问metadata信息

### 5.递归查找文件树，文件扩展搜索

### 6.文件系统修改通知机制

### 7.File类操作API兼容

### 8.文件随机访问增强 mapping a region,locl a region,绝对位置读取

### 9.AIO Reactor（基于事件）和Proactor

下面列一些示例：

### 2.1 IO and New IO 监听文件系统变化通知

​    通过FileSystems.getDefault().newWatchService()获取watchService，然后将需要监听的path目录注册到这个watchservice中，对于这个目录的文件修改，新增，删除等实践可以配置，然后就自动能监听到响应的事件。

```java
    private WatchService watcher;

    public TestWatcherService(Path path) throws IOException {
        watcher = FileSystems.getDefault().newWatchService();
        path.register(watcher, ENTRY_CREATE, ENTRY_DELETE, ENTRY_MODIFY);
    }

    public void handleEvents() throws InterruptedException {
        while (true) {
            WatchKey key = watcher.take();
            for (WatchEvent event : key.pollEvents()) {
                WatchEvent.Kind kind = event.kind();
                if (kind == OVERFLOW) {// 事件可能lost or discarded
                    continue;
                }
                WatchEvent e = (WatchEvent) event;
                Path fileName = e.context();
                System.out.printf("Event %s has happened,which fileName is %s%n", kind.name(), fileName);
            }
            if (!key.reset()) {
                break;
            }
        }
    }
```

### 2.2 IO and New IO遍历文件树 ，通过继承SimpleFileVisitor类，实现事件遍历目录树的操作，然后通过Files.walkFileTree(listDir, opts, Integer.MAX_VALUE, walk);这个API来遍历目录树

```java
   private void workFilePath() {
        Path listDir = Paths.get("/tmp"); // define the starting file 
        ListTree walk = new ListTree();
        Files.walkFileTree(listDir, walk);
        // 遍历的时候跟踪链接
        EnumSet opts = EnumSet.of(FileVisitOption.FOLLOW_LINKS);
        try {
            Files.walkFileTree(listDir, opts, Integer.MAX_VALUE, walk);
        } catch (IOException e) {
            System.err.println(e);
        }
        class ListTree extends SimpleFileVisitor {// NIO2 递归遍历文件目录的接口 

            @Override
            public FileVisitResult postVisitDirectory(Path dir, IOException exc) {
                System.out.println("Visited directory: " + dir.toString());
                return FileVisitResult.CONTINUE;
            }

            @Override
            public FileVisitResult visitFileFailed(Path file, IOException exc) {
                System.out.println(exc);
                return FileVisitResult.CONTINUE;
            }
        }
    }
```

### 2.3 AIO异步IO 文件和网络 异步IO在java NIO2实现了，都是用AsynchronousFileChannel，AsynchronousSocketChanne等实现。

​    Java NIO2中就实现了操作系统的异步非阻塞IO。

```java
    // 使用AsynchronousFileChannel.open(path, withOptions(), 
    // taskExecutor))这个API对异步文件IO的处理 
    public static void asyFileChannel2() {
        final int THREADS = 5;
        ExecutorService taskExecutor = Executors.newFixedThreadPool(THREADS);
        String encoding = System.getProperty("file.encoding");
        List> list = new ArrayList<>();
        int sheeps = 0;
        Path path = Paths.get("/tmp",
                "store.txt");
        try (AsynchronousFileChannel asynchronousFileChannel = AsynchronousFileChannel
                .open(path, withOptions(), taskExecutor)) {
            for (int i = 0; i < 50; i++) {
                Callable worker = new Callable() {
                    @Override
                    public ByteBuffer call() throws Exception {
                        ByteBuffer buffer = ByteBuffer
                                .allocateDirect(ThreadLocalRandom.current()
                                        .nextInt(100, 200));
                        asynchronousFileChannel.read(buffer, ThreadLocalRandom);
                    }
                }
            }
        }
    }
```

# 3\. JDBC 4.1

### 3.1 try-with-resources自动关闭Connection, ResultSet, 和 Statement资源对象

### 3.2 RowSet 1.1：引入RowSetFactory接口和RowSetProvider类，可以创建JDBC driver支持的各种 row sets，这里的rowset实现其实就是将sql语句上的一些操作转为方法的操作，封装了一些功能。

### 3.3. JDBC-ODBC驱动会在jdk8中删除

```java
try (Statement stmt = con.createStatement()) { 
	RowSetFactory aFactory = RowSetProvider.newFactory();
	CachedRowSet crs = aFactory.createCachedRowSet();
	RowSetFactory rsf = RowSetProvider.newFactory("com.sun.rowset.RowSetFactoryImpl", null);
	WebRowSet wrs = rsf.createWebRowSet();

	createCachedRowSet 
	createFilteredRowSet 
	createJdbcRowSet 
	createJoinRowSet 
	createWebRowSet 
```

# 4\. 并发工具增强

### 4.1 fork-join最大的增强，充分利用多核特性，将大问题分解成各个子问题，由多个cpu可以同时解决多个子问题，最后合并结果，继承RecursiveTask，实现compute方法，然后调用fork计算，最后用join合并结果。

```java
class Fibonacci extends RecursiveTask {
    final int n;

    Fibonacci(int n) {
        this.n = n;
    }

    private int compute(int small) {
        final int[] results = {1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89};
        return results[small];
    }

    public Integer compute() {
        if (n <= 10) {
            return compute(n);
        }
        Fibonacci f1 = new Fibonacci(n - 1);
        Fibonacci f2 = new Fibonacci(n - 2);
        System.out.println("fork new thread for " + (n - 1));
        f1.fork();
        System.out.println("fork new thread for " + (n - 2));
        f2.fork();
        return f1.join() + f2.join();
    }
} 
```

### 4.2 ThreadLocalRandon 并发下随机数生成类，保证并发下的随机数生成的线程安全，实际上就是使用threadlocal

```java
        final int MAX = 100000;
        ThreadLocalRandom threadLocalRandom = ThreadLocalRandom.current();
        long start = System.nanoTime();
        for (int i = 0; i < MAX; i++) {
            threadLocalRandom.nextDouble();
        }
        long end = System.nanoTime() - start;
        System.out.println("use time1 : " + end);
        long start2 = System.nanoTime();
        for (int i = 0; i < MAX; i++) {
            Math.random();
        }
        long end2 = System.nanoTime() - start2;
        System.out.println("use time2 : " + end2);
```

### 4.3 phaser 类似cyclebarrier和countdownlatch，不过可以动态添加资源减少资源

```java
    void runTasks(List tasks) {
        final Phaser phaser = new Phaser(1); // "1" to register self
        // create and start threads
        for (final Runnable task : tasks) {
            phaser.register();
            new Thread() {
                public void run() {
                    phaser.arriveAndAwaitAdvance(); // await all creation
                    task.run();
                }
            }.start();
        }
        // allow threads to start and deregister self
        phaser.arriveAndDeregister();
    }
```

# 5\. Networking增强

​    新增URLClassLoader close方法，可以及时关闭资源，后续重新加载`class`文件时不会导致资源被占用或者无法释放问题 URLClassLoader.newInstance(`new` URL\[\]{}).close(); 新增Sockets Direct Protocol 绕过操作系统的数据拷贝，将数据从一台机器的内存数据通过网络直接传输到另外一台机器的内存中

# 6\. Multithreaded Custom Class Loaders

​    解决并发下加载class可能导致的死锁问题，这个是jdk1.6``的一些新版本就解决了，jdk7也做了一些优化

​    jdk7前：

>     类层次结构:        
>         class A extends B  
>         class C extends D
> 
>     ClassLoader委托层次结构：
> 
>     自定义ClassLoader CL1:  
>        直接加载A类   
>        委托给类B的自定义ClassLoader CL2
> 
>     自定义ClassLoader CL2:  
>         直接加载C类  
>         委托给类D的自定义ClassLoader CL1
> 
>     Thread 1:  
>         使用CL1加载A类（锁定CL1）  
>         defineClass A触发器  
>         loadClass B（尝试锁定CL2）
> 
>     Thread 2:  
>         使用CL2加载C类（锁定CL2）  
>         defineClass C触发器  
>         loadClass D（尝试锁定CL1）

​    jdk7

>   Thread 1:  
>     使用CL1加载A类（锁定CL1 + A）  
>     defineClass A触发器  
>     loadClass B（锁CL2 + B）
> 
>   Thread 2:  
>     使用CL2加载C类（锁定CL2 + C）  
>     defineClass C触发器  
>     loadClass D（锁CL1 + D）

# 7\. Security 增强

​    7.1 提供几种 ECC-based algorithms (ECDSA/ECDH) Elliptic Curve Cryptography (ECC)

​    7.2 禁用CertPath Algorithm Disabling

​    7.3 JSSE (SSL/TLS)的一些增强

# 8\. Internationalization 增强 增加了对一些编码的支持和增加了一些显示方面的编码设置等

​    1.New Scripts and Characters from Unicode 6.0.0

​    2.Extensible Support for ISO 4217 Currency Codes

​    3.Currency类添加：     
​      getAvailableCurrencies   
​      getNumericCode   
​      getDisplayName   
​      getDisplayName(Locale)

​    4.Category Locale Support  getDefault(Locale.Category)FORMAT DISPLAY

​    5.Locale Class Supports BCP47 and UTR35  UNICODE\_LOCALE\_EXTENSION  PRIVATE\_USE\_EXTENSION  Locale.Builder getExtensionKeys()  getExtension(char)  getUnicodeLocaleType(String  ……

​    6.New NumericShaper Methods

​        NumericShaper.Range   
​        getShaper(NumericShaper.Range)   
​        getContextualShaper(Set)…… 

# 9.jvm方面的一些特性增强，下面这些特性有些在jdk6中已经存在，这里做了一些优化和增强。

​    1.Jvm支持非java的语言 invokedynamic 指令

​    2\. Garbage-First Collector 适合server端，多处理器下大内存，将heap分成大小相等的多个区域，mark阶段检测每个区域的存活对象，compress阶段将存活对象最小的先做回收，这样会腾出很多空闲区域，这样并发回收其他区域就能减少停止时间，提高吞吐量。

​    3\. HotSpot性能增强 Tiered Compilation -XX:+UseTieredCompilation 多层编译，对于经常调用的代码会直接编译程本地代码，提高效率 Compressed Oops 压缩对象指针，减少空间使用Zero-Based Compressed Ordinary Object Pointers (oops)进一步优化零基压缩对象指针，进一步压缩空间

​    4\. Escape Analysis 逃逸分析，对于只是在一个方法使用的一些变量，可以直接将对象分配到栈上，方法执行完自动释放内存，而不用通过栈的对象引用引用堆中的对象，那么对于对象的回收可能不是那么及时。

​    5\. NUMA Collector Enhancements

​        NUMA(Non Uniform Memory Access),NUMA在多种计算机系统中都得到实现,简而言之,就是将内存分段访问,类似于硬盘的RAID,Oracle中的分簇

# 10\. Java 2D Enhancements

> 1.  XRender-Based Rendering Pipeline -Dsun.java2d.xrender=True
>    
> 2.  Support for OpenType/CFF Fonts GraphicsEnvironment.getAvailableFontFamilyNames
>    
> 3.  TextLayout Support for Tibetan Script
>    
> 4.  Support for Linux Fonts
>    

# 11\. Swing Enhancements

> 1.  JLayer
>    
> 2.  Nimbus Look & Feel
>    
> 3.  Heavyweight and Lightweight Components
>    
> 4.  Shaped and Translucent Windows
>    
> 5.  Hue-Saturation-Luminance (HSL) Color Selection in JColorChooser Class
>    

# 12\. Jdk8 lambda表达式 最大的新增的特性，不过在很多动态语言中都已经原生支持。

​    原来这么写：

```java
        btn.setOnAction(new EventHandler() {
            @Override
            public void handle(ActionEvent event) {
                System.out.println("Hello World!");
            }
        });
```

​    jdk8直接可以这么写：

```java
btn.setOnAction(event -> System.out.println("Hello World!"));
```

​    更多示例：

```java
public class Utils {
        public static int compareByLength(String in, String out){
            return in.length() - out.length();
        }
    }

    public class MyClass {
        public void doSomething() {
            String[] args = new String[] {"microsoft","apple","linux","oracle"}
            Arrays.sort(args, Utils::compareByLength);
        }
    }
```

# 13.jdk8的一些其他特性，当然jdk8的增强功能还有很多，大家可以参考 [https://www.oracle.com/technetwork/java/javase/8-whats-new-2157071.html](https://www.oracle.com/technetwork/java/javase/8-whats-new-2157071.html)

​    用Metaspace代替PermGen 动态扩展，可以设置最大值，限制于本地内存的大小

​    Parallel array sorting 新APIArrays#parallelSort.

​    New Date & Time API

```java
Clock clock = Clock.systemUTC(); //return the current time based on your system clock and set to UTC.
Clock clock = Clock.systemDefaultZone(); //return time based on system clock zone
long time = clock.millis(); //time in milliseconds from January 1st, 1970
​```xxxxxxxxxx Clock clock = Clock.systemUTC(); //return the current time based on your system clock and set to UTC.Clock clock = Clock.systemDefaultZone(); //return time based on system clock zonelong time = clock.millis(); //time in milliseconds from January 1st, 1970
```