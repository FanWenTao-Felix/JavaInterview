## 运算符优先级

​	优先级从上到下依次递减，最上面具有最高的优先级，逗号操作符具有最低的优先级。

​	相同优先级中，按结合顺序计算。**大多数运算是从左至右计算，只有三个优先级是从右至左结合的，它们是单目运算符、条件运算符、赋值运算符**。

​	基本的优先级需要记住：

- 指针最优，单目运算优于双目运算。如正负号。
- 先乘除（模），后加减。
- 先算术运算，后移位运算，最后位运算。请特别注意：`1 << 3 + 2 & 7`等价于 `(1 << (3 + 2)) & 7`.
- 逻辑运算最后计算。

## 优先级表

| 运算符                                  | 结合性   |
| --------------------------------------- | -------- |
| `[ ] . ( )` (方法调用)                  | 从左向右 |
| `! ~ ++ -- +`(一元运算) -(一元运算)     | 从右向左 |
| `* / %`                                 | 从左向右 |
| `+ -`                                   | 从左向右 |
| `<< >> >>>`                             | 从左向右 |
| `< <= > >= instanceof`                  | 从左向右 |
| `== !=`                                 | 从左向右 |
| `&`                                     | 从左向右 |
| `^`                                     | 从左向右 |
| `|`                                     | 从左向右 |
| `&&`                                    | 从左向右 |
| `||`                                    | 从左向右 |
| `?:`                                    | 从右向左 |
| `= += -= *= /= %= &= |= ^= <<= >>= >>=` | 从右向左 |
| `,`                                     | 从左到右 |

> ​	无符号右移运算符 `>>>`，无符号右移的规则只记住一点：**忽略了符号位扩展，0 补最高位**。无符号右移规则和右移运算是一样的，只是填充时不管左边的数字是正是负都用 0 来填充，无符号右移运算只针对负数计算，因为对于正数来说这种运算没有意义。无符号右移运算符 `>>>` 只是对 32 位和 64 位的值有意义



## Java异常

​	Java中有Error和Exception，它们都是继承自Throwable类。

![img](../image/error.png)

![img](../image/exception.png)

## 二者的不同之处

​	Exception：

- 可以是可被控制(checked) 或不可控制的(unchecked)。
- 表示一个由程序员导致的错误。
- 应该在应用程序级被处理。

​    Error：

- 总是不可控制的(unchecked)。
- 经常用来用于表示系统错误或低层资源的错误。
- 如何可能的话，应该在系统级被捕捉。

## 异常的分类

- **Checked exception**: 这类异常都是Exception的子类。异常的向上抛出机制进行处理，假如子类可能产生A异常，那么在父类中也必须throws A异常。可能导致的问题：代码效率低，耦合度过高。
- **Unchecked exception**: **这类异常都是RuntimeException的子类，虽然RuntimeException同样也是Exception的子类，但是它们是非凡的，它们不能通过client code来试图解决**，所以称为Unchecked exception 。



## Java泛型

​	开发人员在使用泛型的时候，很容易根据自己的直觉而犯一些错误。比如一个方法如果接收`List`作为形式参数，那么如果尝试将一个`List`的对象作为实际参数传进去，却发现无法通过编译。虽然从直觉上来说，`Object`是`String`的父类，这种类型转换应该是合理的。**但是实际上这会产生隐含的类型转换问题，因此编译器直接就禁止这样的行为**。

## 类型擦除

​	Java中的泛型基本上都是在编译器这个层次来实现的，**在生成的Java字节代码中是不包含泛型中的类型信息的。使用泛型的时候加上的类型参数，会被编译器在编译的时候去掉，这个过程就称为类型擦除**。如在代码中定义的`List`和`List`等类型，在编译之后都会变成`List`。**JVM看到的只是List，而由泛型附加的类型信息对JVM来说是不可见的**。Java编译器会在编译时尽可能的发现可能出错的地方，但是仍然无法避免在运行时刻出现类型转换异常的情况。

​	很多泛型的奇怪特性都与这个类型擦除的存在有关，包括：

- **泛型类并没有自己独有的Class类对象**。比如并不存在`List.class`或是`List.class`，而只有`List.class`。
- **静态变量是被泛型类的所有实例所共享的**。对于声明为`MyClass`的类，访问其中的静态变量的方法仍然是 `MyClass.myStaticVar`。不管是通过`new MyClass`还是`new MyClass`创建的对象，都是共享一个静态变量。
- **泛型的类型参数不能用在Java异常处理的catch语句中**。因为异常处理是由`JVM`在运行时刻来进行的。由于类型信息被擦除，`JVM`是无法区分两个异常类型`MyException`和`MyException`的。对于`JVM`来说，它们都是 `MyException`类型的。也就无法执行与异常对应的catch语句。

   类型擦除的基本过程也比较简单，首先是找到用来替换类型参数的具体类。这个具体类一般是Object。如果指定了类型参数的上界的话，则使用这个上界。把代码中的类型参数都替换成具体的类。同时去掉出现的类型声明，即去掉<>的内容。比如`T get()`方法声明就变成了`Object get()`；`List`就变成了`List`。接下来就可能需要生成一些桥接方法（bridge method）。这是由于擦除了类型之后的类可能缺少某些必须的方法。比如考虑下面的代码：

```
class MyString implements Comparable<String> {
    public int compareTo(String str) {        
        return 0;    
    }
}
```

​	当类型信息被擦除之后，上述类的声明变成了`class MyString implements Comparable`。但是这样的话，类`MyString`就会有编译错误，因为没有实现接口`Comparable`声明的`int compareTo(Object)`方法。这个时候就由编译器来动态生成这个方法。

## 通配符

​	在使用泛型类的时候，既可以指定一个具体的类型，如`List`就声明了具体的类型是`String`；也可以用通配符`?`来表示未知类型，如`List`就声明了`List`中包含的元素类型是未知的。 通配符所代表的其实是一组类型，但具体的类型是未知的。`List`所声明的就是所有类型都是可以的。**但是`List`并不等同于`List`。`List`实际上确定了`List`中包含的是`Object`及其子类，在使用的时候都可以通过`Object`来进行引用。而`List`则其中所包含的元素类型是不确定**。其中可能包含的是`String`，也可能是 `Integer`。如果它包含了`String`的话，往里面添加`Integer`类型的元素就是错误的。**正因为类型未知，就不能通过new ArrayList中的元素确总是可以用Object来引用的，因为虽然类型未知，但肯定是Object及其子类**。考虑下面的代码：

```
public void wildcard(List<?> list) {
    list.add(1);//编译错误
}  
```

> 如上所示，试图对一个带通配符的泛型类进行操作的时候，总是会出现编译错误。其原因在于通配符所表示的类型是未知的。

​	因为对于`List`中的元素只能用`Object`来引用，在有些情况下不是很方便。在这些情况下，可以使用上下界来限制未知类型的范围。 如 **`List`说明List中可能包含的元素类型是`Number`及其子类。而`List`则说明List中包含的是Number及其父类**。当引入了上界之后，在使用类型的时候就可以使用上界类中定义的方法。

## 类型系统

​	在Java中，大家比较熟悉的是通过继承机制而产生的类型体系结构。比如`String`继承自`Object`。根据`Liskov替换原则`，子类是可以替换父类的。当需要`Object`类的引用的时候，如果传入一个`String`对象是没有任何问题的。但是反过来的话，即用父类的引用替换子类引用的时候，就需要进行强制类型转换。编译器并不能保证运行时刻这种转换一定是合法的。**这种自动的子类替换父类的类型转换机制，对于数组也是适用的。 String[]可以替换Object[]**。但是泛型的引入，对于这个类型系统产生了一定的影响。**正如前面提到的List是不能替换掉List的**。

​	引入泛型之后的类型系统增加了两个维度：**一个是类型参数自身的继承体系结构，另外一个是泛型类或接口自身的继承体系结构**。第一个指的是对于 `List`和`List`这样的情况，类型参数`String`是继承自`Object`的。而第二种指的是 `List`接口继承自`Collection`接口。对于这个类型系统，有如下的一些规则：

- **相同类型参数的泛型类的关系取决于泛型类自身的继承体系结构**。即`List`是`Collection` 的子类型，`List`可以替换`Collection`。这种情况也适用于带有上下界的类型声明。
- **当泛型类的类型声明中使用了通配符的时候，其子类型可以在两个维度上分别展开**。如对`Collection`来说，其子类型可以在`Collection`这个维度上展开，即`List`和`Set`等；也可以在`Number`这个层次上展开，即`Collection`和`Collection`等。如此循环下去，`ArrayList`和 `HashSet`等也都算是`Collection`的子类型。
- 如果泛型类中包含多个类型参数，则对于每个类型参数分别应用上面的规则。



## Object

## 	getClass

返回该对象运行时的 `class` 对象，返回的 `Class` 对象是由所表示的类的静态同步方法锁定的对象。

## 	hashCode

返回该对象的 `hashcode`，该方法对hash表提供支持，例如 `HashMap`。 对于该方法有几点需要注意：

- 在运行中的Java应用，如果用在 `equals` 中进行比较的信息没有改变，那么不论何时调用都需要返回一致的int值。这个hash值在应用的两次执行中不需要保持一致。
- 如果两个对象根据 `equals` 方法认为是相等的，那么这两个对象也应该返回相等的 `hashcode`。
- 不要求两个不相等的对象，在调用 `hashCode` 方法返回的结果是必须是不同的。然而，程序员应该了解不同的对象产生不同的 `hashcode` 能够提升哈希表的效率。 Object的`hashcode`对不同的对象，尽可能返回不同的 `hashcode` 。这通常通过将对象的内部地址转换为整数来实现，但Java编程语言不需要此实现技术。

###    Arrays.hashCode

Arrays.hashCode 是一个数组的浅哈希码实现，深哈希可以使用 `deepHashCode`。并且当数组长度为1时，`Arrays.hashCode(object) = object.hashCode` 不一定成立

### 	31

不论是String、Arrays在计算多个元素的哈希值的时候，都会有31这个数字。主要有以下两个原因：

- 31是一个不大不小的质数，是作为 hashCode 乘子的优选质数之一。

  > 另外一些相近的质数，比如37、41、43等等，也都是不错的选择。那么为啥偏偏选中了31呢？请看第二个原因。

- 31可以被 JVM 优化， 31 * i = (i << 5) - i31∗*i*=(*i*<<5)−*i* 。

上面两个原因中，第一个需要解释一下，第二个比较简单，就不说了。一般在设计哈希算法时，会选择一个特殊的质数。至于为啥选择质数，我想应该是可以降低哈希算法的冲突率。

在 Effective Java 中有一段相关的解释：

> 选择数字31是因为它是一个奇质数，如果选择一个偶数会在乘法运算中产生溢出，导致数值信息丢失，因为乘二相当于移位运算。选择质数的优势并不是特别的明显，但这是一个传统。同时，数字31有一个很好的特性，即乘法运算可以被移位和减法运算取代，来获取更好的性能： 31 * i == (i << 5) - i31∗*i*==(*i*<<5)−*i* ，现代的 Java 虚拟机可以自动的完成这个优化。

## 	equals

判定两个对象是否相等。`equals`和`hashCode`需要同时被`overwrite`

## 	clone

创建一个该对象的副本，并且对于对象 x 应当满足以下表达式：

```
x.clone() != x
x.clone().getClass() == x.getClass()
x.clone().equals(x)
```

## 	toString

## 	wait

当前线程等待知道其他线程调用该对象的 `notify` 或者 `notifyAll`方法。当前线程必须拥有该对象的 `monitor`。线程释放该对象`monitor`的拥有权，并且等待到别的线程通知等待在该对象`monitor`上的线程苏醒。然后线程重新拥有`monitor`并继续执行。在某些jdk版本中，中断和虚假唤醒是存在的，所以`wait`方法需要放在循环中。

```
synchronized (obj) {
    while (<condition does not hold>)
        obj.wait();
    ... // Perform action appropriate to condition
}
```

该方法只能被拥有该对象`monitor`的线程调用。

### 	虚假唤醒（spurious wakeup）

虚假唤醒就是一些`obj.wait()`会在除了`obj.notify()`和`obj.notifyAll()`的其他情况被唤醒，而此时是不应该唤醒的。

> 注意 Lock 的 Conditon.await 也有虚假唤醒的问题

解决的办法是基于while来反复判断进入正常操作的临界条件是否满足

> 同时也可以使用同步数据结构：BlokingQueue

#### 	解释

虚假唤醒（`spurious wakeup`）是一个表象，即在多处理器的系统下发出 wait 的程序有可能在没有 notify 唤醒的情形下苏醒继续执行。

以运行在 Linux 的 hotspot 虚拟机上的 java 程序为例， `wait` 方法在 jvm 执行时实质是调用了底层 `pthread_cond_wait/pthread_cond_timedwait` 函数，挂起等待条件变量来达到线程间同步通信的效果，而底层 `wait` 函数在设计之初为了不减慢条件变量操作的效率并没有去保证每次唤醒都是由 `notify` 触发，而是把这个任务交由上层应用去实现，即使用者需要定义一个循环去判断是否条件真能满足程序继续运行的需求，当然这样的实现也可以避免因为设计缺陷导致程序异常唤醒的问题。

## 	notify

唤醒一个等待在该对象`monitor`上的线程。如果有多个线程等待，则会随机选择一个线程唤醒。线程等待是通过调用`wait`方法。

唤醒的线程不会立即执行，直到当前线程放弃对象上的锁。唤醒的线程也会以通常的方式和竞争该对象锁的线程进行竞争。也就是说，唤醒的线程在对该对象的加锁中没有任何优先级。

该方法只能被拥有该对象`monitor`的线程调用。线程拥有`monitor`有下面三种方式：

- 执行该对象的 `synchronized` 方法
- 执行以该对象作为同步语句的`synchronized`方法体
- 对于class对象，可以执行该对象的`static synchronized`方法

在同一时间只能有一个线程能够拥有该对象`monitor`

## 	finalize

当 GC 认为该对象已经没有任何引用的时候，该方法被GC收集器调用。子类可以 `overwrite` 该方法来关闭系统资源或者其他清理任务。

`finalize` 的一般契约是，如果 Java 虚拟机确定不再有任何方法可以通过任何尚未死亡的线程访问此对象，除非由于某个操作，它将被调用通过最终确定准备完成的其他一些对象或类来完成。 `finalize` 方法可以采取任何操作，包括使该对象再次可用于其他线程；但是，`finalize` 的通常目的是在对象被不可撤销地丢弃之前执行清理操作。例如，表示输入/输出连接的对象的 `finalize` 方法可能会执行显式 `I/O` 事务，以在永久丢弃对象之前断开连接。

类 Object 的 finalize 方法不执行任何特殊操作;它只是正常返回。 Object 的子类可以覆盖此定义。

Java 编程语言不保证哪个线程将为任何给定对象调用 `finalize` 方法。但是，可以保证，调用 finalize 时，调用 finalize 的线程不会持有任何用户可见的同步锁。如果 `finalize` 方法抛出未捕获的异常，则忽略该异常并终止该对象的终止。在为对象调用 `finalize` 方法之后，在 Java 虚拟机再次确定不再有任何方法可以通过任何尚未死亡的线程访问此对象之前，不会采取进一步操作，包括可能的操作通过准备完成的其他对象或类，此时可以丢弃该对象。

对于任何给定对象，Java 虚拟机永远不会多次调用 `finalize` 方法。 `finalize` 方法抛出的任何异常都会导致暂停此对象的终结，但会被忽略。

### 	缺陷

- 一些与 `finalize` 相关的方法，由于一些致命的缺陷，已经被废弃了，如 `System.runFinalizersOnExit()` 方法、`Runtime.runFinalizersOnExit()`方法。
- `System.gc()` 与 `System.runFinalization()` 方法增加了finalize方法执行的机会，但不可盲目依赖它们。
- Java 语言规范并不保证 `finalize` 方法会被及时地执行、而且根本不会保证它们会被执行。
- `finalize` 方法可能会带来性能问题。因为JVM通常在单独的低优先级线程中完成finalize的执行。
- 对象再生问题： `finalize` 方法中，可将待回收对象赋值给GC Roots可达的对象引用，从而达到对象再生的目的。
- `finalize` 方法至多由GC执行一次(用户当然可以手动调用对象的 `finalize` 方法，但并不影响GC对 `finalize` 的行为)。



## StringBuilder

`StringBuilder`类也封装了一个字符数组，定义如下：

```
    char[] value;
```

与`String`不同，它不是`final`的，可以修改。另外，与`String`不同，字符数组中不一定所有位置都已经被使用，它有一个实例变量，表示数组中已经使用的字符个数，定义如下：

```
    int count;
```

`StringBuilder`继承自`AbstractStringBuilder`，它的默认构造方法是：

```
    public StringBuilder() {
        super(16);
    }
```

调用父类的构造方法，父类对应的构造方法是：

```java
    AbstractStringBuilder(int capacity) {
        value = new char[capacity];
    }
```

也就是说，`new StringBuilder()`这句代码，内部会创建一个长度为16的字符数组，count的默认值为0。

## append的实现

```java
    public AbstractStringBuilder append(String str) {
        if (str == null) str = "null";
        int len = str.length();
        ensureCapacityInternal(count + len);
        str.getChars(0, len, value, count);
        count += len;
        return this;
    }
```

`append`会直接拷贝字符到内部的字符数组中，如果字符数组长度不够，会进行扩展，实际使用的长度用`count`体现。具体来说，`ensureCapacityInternal(count+len)`会确保数组的长度足以容纳新添加的字符，`str.getChars`会拷贝新添加的字符到字符数组中，`count+=len`会增加实际使用的长度。

`ensureCapacityInternal`的代码如下：

```java
    private void ensureCapacityInternal(int minimumCapacity) {

        if (minimumCapacity - value.length > 0)
            expandCapacity(minimumCapacity);
    }
```

如果字符数组的长度小于需要的长度，则调用`expandCapacity`进行扩展，`expandCapacity`的代码是：

```java
    void expandCapacity(int minimumCapacity) {
        int newCapacity = value.length * 2 + 2;
        if (newCapacity - minimumCapacity < 0)
            newCapacity = minimumCapacity;
        if (newCapacity < 0) {
            if (minimumCapacity < 0)
                throw new OutOfMemoryError();
            newCapacity = Integer.MAX_VALUE;
        }
        value = Arrays.copyOf(value, newCapacity);
    }
```

扩展的逻辑是，分配一个足够长度的新数组，然后将原内容拷贝到这个新数组中，最后让内部的字符数组指向这个新数组，这个逻辑主要靠下面这句代码实现：

```
    value = Arrays.copyOf(value, newCapacity);
```

## toString实现

字符串构建完后，我们来看toString代码：

```java
    public String toString() {
        return new String(value, 0, count);
    }
```



## 注解

​	注解(`Annotation`)是 Java1.5 中引入的一个重大修改之一，为我们在代码中添加信息提供了一种形式化的方法，使我们可以在稍后某个时刻非常方便的使用这些数据。注解在一定程度上是把元数据与源代码结合在一起，而不是保存在外部文档中。注解的含义可以理解为 java 中的元数据。元数据是描述数据的数据。

​	注解是一个继承自`java.lang.annotation.Annotation`的接口

## 可见性

根据注解在程序不同时期的可见性，可以把注解区分为：

- `source`：注解会在编译期间被丢弃，不会编译到 class 文件
- `class`：注解会被编译到 class 文件中，但是在运行时不能获取
- `runtime`：注解会被编译到 class 文件中，并且能够在运行时通过反射获取

## 继承

|                                        | 有@Inherited | 没有@Inherited |
| -------------------------------------- | ------------ | -------------- |
| 子类的类上能否继承到父类的类上的注解？ | 否           | 能             |
| 子类实现了父类上的抽象方法             | 否           | 否             |
| 子类继承了父类上的方法                 | 能           | 能             |
| 子类覆盖了父类上的方法                 | 否           | 否             |

`@Inherited` 只是可控制对类名上注解是否可以被继承。不能控制方法上的注解是否可以被继承。

## 注解的实现机制

1. 注解是继承自：`java.lang.annotation.Annotation` 的接口

```
...
  Compiled from "TestAnnotation.java"
public interface TestAnnotation extends java.lang.annotation.Annotation
...
```

1. 注解内部的属性是在编译期间确定的

```
...
SourceFile: "SimpleTest.java"
RuntimeVisibleAnnotations:
  0: #43(#44=s#45)
...
```

1. 注解在运行时会生成 `Proxy` 代理类，并使用 `AnnotationInvocationHandler.memberValues` 来进行数据读取

```java
...
default:
    //从 Map 中获取数据
    Object var6 = this.memberValues.get(var4);
    if (var6 == null) {
        throw new IncompleteAnnotationException(this.type, var4);
    } else if (var6 instanceof ExceptionProxy) {
        throw ((ExceptionProxy)var6).generateException();
    } else {
        if (var6.getClass().isArray() && Array.getLength(var6) != 0) {
            var6 = this.cloneArray(var6);
        }

        return var6;
    }
}
...
```





### 如何用数组实现队列？

用数组实现队列时要注意 **溢出** 现象，这时我们可以采用循环数组的方式来解决，即将数组收尾相接。使用front指针指向队列首位，tail指针指向队列末位。

------

### 内部类访问局部变量的时候，为什么变量必须加上final修饰？

因为生命周期不同。局部变量在方法结束后就会被销毁，但内部类对象并不一定，这样就会导致内部类引用了一个不存在的变量。

所以编译器会在内部类中生成一个局部变量的拷贝，这个拷贝的生命周期和内部类对象相同，就不会出现上述问题。

但这样就导致了其中一个变量被修改，两个变量值可能不同的问题。为了解决这个问题，编译器就要求局部变量需要被final修饰，以保证两个变量值相同。

在JDK8之后，编译器不要求内部类访问的局部变量必须被final修饰，但局部变量值不能被修改（无论是方法中还是内部类中），否则会报编译错误。利用javap查看编译后的字节码可以发现，编译器已经加上了final。

------

### long s = 499999999 * 499999999 在上面的代码中，s的值是多少？

根据代码的计算结果，`s`的值应该是`-1371654655`，**这是由于Java中右侧值的计算默认是**`int`类型。

------

### NIO相关，Channels、Buffers、Selectors

`NIO(Non-blocking IO)`为所有的原始类型提供(Buffer)缓存支持，字符集编码解码解决方案。 `Channel` ：一个新的原始I/O 抽象。 支持锁和内存映射文件的文件访问接口。提供多路(non-bloking) 非阻塞式的高伸缩性网络I/O 。

| IO     | NIO      |
| ------ | -------- |
| 面向流 | 面向缓冲 |
| 阻塞IO | 非阻塞IO |
| 无     | 选择器   |

##### 流与缓冲

Java NIO和IO之间第一个最大的区别是，**IO是面向流的，NIO是面向缓冲区的**。 Java IO面向流意味着每次从流中读一个或多个字节，直至读取所有字节，它们没有被缓存在任何地方。此外，它不能前后移动流中的数据。如果需要前后移动从流中读取的数据，需要先将它缓存到一个缓冲区。

Java NIO的缓冲导向方法略有不同。**数据读取到一个它稍后处理的缓冲区，需要时可在缓冲区中前后移动。这就增加了处理过程中的灵活性**。但是，还需要检查是否该缓冲区中包含所有您需要处理的数据。而且，需确保当更多的数据读入缓冲区时，不要覆盖缓冲区里尚未处理的数据。

##### 阻塞与非阻塞IO

Java IO的各种流是阻塞的。这意味着，当一个线程调用`read()` 或 `write()`时，该线程被阻塞，直到有一些数据被读取，或数据完全写入。该线程在此期间不能再干任何事情了。 **Java NIO的非阻塞模式，是线程向某通道发送请求读取数据，仅能得到目前可用的数据，如果目前没有数据可用时，就什么都不会获取，当然它不会保持线程阻塞。所以直至数据变的可以读取之前，该线程可以继续做其他的事情**。 非阻塞写也是如此。所以一个单独的线程现在可以管理多个输入和输出通道。

##### 选择器（Selectors）

Java NIO 的 **选择器允许一个单独的线程来监视多个输入通道**，你可以注册多个通道使用一个选择器，然后使用一个单独的线程来“选择”通道：这些通道里已经有可以处理的输入，或者选择已准备写入的通道。这种选择机制，使得一个单独的线程很容易来管理多个通道。

------

### 反射的用途

Java反射机制可以让我们在编译期(Compile Time)之外的运行期(Runtime)检查类，接口，变量以及方法的信息。反射还可以让我们在运行期实例化对象，调用方法，通过调用`get/set`方法获取变量的值。同时我们也可以通过反射来获取泛型信息，以及注解。还有更高级的应用–动态代理和动态类加载（`ClassLoader.loadclass()`）。

下面列举一些比较重要的方法：

- getFields：获取所有 `public` 的变量。
- getDeclaredFields：获取所有包括 `private` , `protected` 权限的变量。
- setAccessible：设置为 true 可以跳过Java权限检查，从而访问`private`权限的变量。
- getAnnotations：获取注解，可以用在类和方法上。

获取方法的泛型参数：

```
method = Myclass.class.getMethod("setStringList", List.class);

Type[] genericParameterTypes = method.getGenericParameterTypes();

for(Type genericParameterType : genericParameterTypes){
    if(genericParameterType instanceof ParameterizedType){
        ParameterizedType aType = (ParameterizedType) genericParameterType;
        Type[] parameterArgTypes = aType.getActualTypeArguments();
        for(Type parameterArgType : parameterArgTypes){
            Class parameterArgClass = (Class) parameterArgType;
            System.out.println("parameterArgClass = " + parameterArgClass);
        }
    }
}
```

动态代理：

```
//Main.java
public static void main(String[] args) {
    HelloWorld helloWorld=new HelloWorldImpl();
    InvocationHandler handler=new HelloWorldHandler(helloWorld);

    //创建动态代理对象
    HelloWorld proxy=(HelloWorld)Proxy.newProxyInstance(
            helloWorld.getClass().getClassLoader(),
            helloWorld.getClass().getInterfaces(),
            handler);
    proxy.sayHelloWorld();
}

//HelloWorldHandler.java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    Object result = null;
    //调用之前
    doBefore();
    //调用原始对象的方法
    result=method.invoke(obj, args);
    //调用之后
    doAfter();
    return result;
}
```

通过反射获取方法注解的参数：

```
Class aClass = TheClass.class;
Annotation[] annotations = aClass.getAnnotations();

for(Annotation annotation : annotations){
   if(annotation instanceof MyAnnotation){
       MyAnnotation myAnnotation = (MyAnnotation) annotation;
       System.out.println("name: " + myAnnotation.name());
       System.out.println("value: " + myAnnotation.value());
   }
}
```

------

------

### 非静态内部类能定义静态方法吗？

```
public class OuterClass{
    private static float f = 1.0f;

    class InnerClass{
        public static float func(){return f;}
    }
}
```

以上代码会出现编译错误，因为只有静态内部类才能定义静态方法。

------

### Lock 和 Synchronized 有什么区别？

1. 使用方法的区别

```
- **Synchronized**：在需要同步的对象中加入此控制，`synchronized`可以加在方法上，也可以加在特定代码块中，括号中表示需要锁的对象。

- **Lock**：需要显示指定起始位置和终止位置。一般使用`ReentrantLock`类做为锁，多个线程中必须要使用一个`ReentrantLock`类做为对象才能保证锁的生效。且在加锁和解锁处需要通过`lock()`和`unlock()`显示指出。所以一般会在`finally`块中写`unlock()`以防死锁。
```

1. 性能的区别

```
`synchronized`是托管给JVM执行的，而`lock`是java写的控制锁的代码。在Java1.5中，`synchronize`是性能低效的。因为这是一个重量级操作，需要调用操作接口，导致有可能加锁消耗的系统时间比加锁以外的操作还多。相比之下使用Java提供的Lock对象，性能更高一些。但是到了Java1.6，发生了变化。`synchronize`在语义上很清晰，可以进行很多优化，有适应自旋，锁消除，锁粗化，轻量级锁，偏向锁等等。导致在Java1.6上`synchronize`的性能并不比Lock差。

  - **Synchronized**：采用的是CPU悲观锁机制，即线程获得的是独占锁。独占锁意味着 **其他线程只能依靠阻塞来等待线程释放锁**。而在CPU转换线程阻塞时会引起线程上下文切换，当有很多线程竞争锁的时候，会引起CPU频繁的上下文切换导致效率很低。

  - **Lock**：用的是乐观锁方式。所谓乐观锁就是，**每次不加锁而是假设没有冲突而去完成某项操作，如果因为冲突失败就重试，直到成功为止**。乐观锁实现的机制就是`CAS`操作。我们可以进一步研究`ReentrantLock`的源代码，会发现其中比较重要的获得锁的一个方法是`compareAndSetState`。这里其实就是调用的CPU提供的特殊指令。
```

1. `ReentrantLock`：具有更好的可伸缩性：比如时间锁等候、可中断锁等候、无块结构锁、多个条件变量或者锁投票。

------

### float 变量如何与 0 比较？

folat类型的还有double类型的，**这些小数类型在趋近于0的时候直接等于0的可能性很小，一般都是无限趋近于0，因此不能用==来判断**。应该用`|x-0|来判断，这里`|x-0|`表示绝对值，`err`表示限定误差。

```
//用程序表示就是

fabs(x) < 0.00001f
```

------

### 如何新建非静态内部类？

内部类在声明的时候必须是 `Outer.Inner a`，就像`int a` 一样，至于静态内部类和非静态内部类new的时候有点区别：

- `Outer.Inner a = new Outer().new Inner()`（非静态，先有Outer对象才能 new 内部类）
- `Outer.Inner a = new Outer.Inner()`（静态内部类）

------

### Java标识符命名规则

可以包含：字母、数字、$、`_`(下划线)，不可用数字开头，不能是 Java 的关键字和保留字。

------

### 你知道哪些JDK中用到的设计模式？

- 装饰模式：java.io
- 单例模式：Runtime类
- 简单工厂模式：Integer.valueOf方法
- 享元模式：String常量池、Integer.valueOf(int i)、Character.valueOf(char c)
- 迭代器模式：Iterator
- 职责链模式：ClassLoader的双亲委派模型
- 解释器模式：正则表达式java.util.regex.Pattern

------

### ConcurrentHashMap如何保证线程安全

JDK 1.7及以前：

ConcurrentHashMap允许多个修改操作并发进行，其关键在于使用了锁分离技术。它使用了多个锁来控制对hash表的不同部分进行的修改。ConcurrentHashMap内部使用段(Segment)来表示这些不同的部分，每个段其实就是一个小的hash table，它们有自己的锁。只要多个修改操作发生在不同的段上，它们就可以并发进行。

JDK 1.8：

Segment虽保留，但已经简化属性，仅仅是为了兼容旧版本。

插入时使用CAS算法：unsafe.compareAndSwapInt(this, valueOffset, expect, update)。 CAS(Compare And Swap)意思是如果valueOffset位置包含的值与expect值相同，则更新valueOffset位置的值为update，并返回true，否则不更新，返回false。插入时不允许key或value为null

与Java8的HashMap有相通之处，底层依然由“数组”+链表+红黑树；

底层结构存放的是TreeBin对象，而不是TreeNode对象；

CAS作为知名无锁算法，那ConcurrentHashMap就没用锁了么？当然不是，当hash值与链表的头结点相同还是会synchronized上锁，锁链表。

------

### i++在多线程环境下是否存在问题，怎么解决？

虽然递增操作++i是一种紧凑的语法，使其看上去只是一个操作，但这个操作并非原子的，因而它并不会作为一个不可分割的操作来执行。实际上，它包含了三个独立的操作：读取count的值，将值加1，然后将计算结果写入count。这是一个“读取 - 修改 - 写入”的操作序列，并且其结果状态依赖于之前的状态。所以在多线程环境下存在问题。

要解决自增操作在多线程环境下线程不安全的问题，可以选择使用Java提供的原子类，如AtomicInteger或者使用synchronized同步方法。

------

### new与newInstance()的区别

- new是一个关键字，它是调用new指令创建一个对象，然后调用构造方法来初始化这个对象，可以使用带参数的构造器
- newInstance()是Class的一个方法，在这个过程中，是先取了这个类的不带参数的构造器Constructor，然后调用构造器的newInstance方法来创建对象。

> Class.newInstance不能带参数，如果要带参数需要取得对应的构造器，然后调用该构造器的Constructor.newInstance(Object … initargs)方法

------

### 你了解哪些JDK1.8的新特性？

- 接口的默认方法和静态方法，JDK8允许我们给接口添加一个非抽象的方法实现，只需要使用default关键字即可。也可以定义被static修饰的静态方法。
- 对HashMap进行了改进，当单个桶的元素个数大于6时就会将实现改为红黑树实现，以避免构造重复的hashCode的攻击
- 多并发进行了优化。如ConcurrentHashMap实现由分段加锁、锁分离改为CAS实现。
- JDK8拓宽了注解的应用场景，注解几乎可以使用在任何元素上，并且允许在同一个地方多次使用同一个注解
- Lambda表达式

------

### 你用过哪些JVM参数？

- Xms 堆最小值
- Xmx 堆最大值
- Xmn: 新生代容量
- XX:SurvivorRatio 新生代中Eden与Surivor空间比例
- Xss 栈容量
- XX:PermSize 方法区初始容量
- XX:MaxPermSize 方法区最大容量
- XX:+PrintGCDetails 收集器日志参数

------

### 如何打破 ClassLoader 双亲委托？

重写`loadClass()`方法。

------

### hashCode() && equals()

`hashcode()` 返回该对象的哈希码值，支持该方法是为哈希表提供一些优点，例如，`java.util.Hashtable` 提供的哈希表。

在 Java 应用程序执行期间，在同一对象上多次调用 `hashCode` 方法时，必须一致地返回相同的整数，前提是对象上 `equals` 比较中所用的信息没有被修改（`equals`默认返回对象地址是否相等）。如果根据 `equals(Object) `方法，两个对象是相等的，那么在两个对象中的每个对象上调用 `hashCode` 方法都必须生成相同的整数结果。

以下情况不是必需的：如果根据 `equals(java.lang.Object)` 方法，两个对象不相等，那么在两个对象中的任一对象上调用 `hashCode` 方法必定会生成不同的整数结果。但是，**程序员应该知道，为不相等的对象生成不同整数结果可以提高哈希表的性能**。

实际上，由 `Object` 类定义的 `hashCode` 方法确实会针对不同的对象返回不同的整数。（**这一般是通过将该对象的内部地址转换成一个整数来实现的，但是 JavaTM 编程语言不需要这种实现技巧I**。）

- **hashCode的存在主要是用于查找的快捷性**，如 Hashtable，HashMap等，hashCode 是用来在散列存储结构中确定对象的存储地址的；
- 如果两个对象相同，就是适用于 `equals(java.lang.Object)` 方法，那么这两个对象的 `hashCode` 一定要相同；
- 如果对象的 `equals` 方法被重写，那么对象的 `hashCode` 也尽量重写，并且产生 `hashCode` 使用的对象，一定要和 `equals` 方法中使用的一致，否则就会违反上面提到的第2点；
- **两个对象的hashCode相同，并不一定表示两个对象就相同，也就是不一定适用于equals(java.lang.Object) 方法，只能够说明这两个对象在散列存储结构中，如Hashtable，他们“存放在同一个篮子里”**。

------

### Thread.sleep() & Thread.yield()

sleep()和yield()都会释放CPU。

sleep()使当前线程进入停滞状态，所以执行sleep()的线程在指定的时间内肯定不会执行；yield()只是使当前线程重新回到可执行状态，所以执行yield()的线程有可能在进入到可执行状态后马上又被执行。

sleep()可使优先级低的线程得到执行的机会，当然也可以让同优先级和高优先级的线程有执行的机会；yield()只能使同优先级的线程有执行的机会。
