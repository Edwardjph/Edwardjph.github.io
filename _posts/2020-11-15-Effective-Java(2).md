## 第四章 类与接口

### 第15条：使类和成员的可访问性最小化

规则很简单：尽可能地使每个类或者成员不被外界访问；

公有类的实例域决不能是公有的，包含公有可变域的类通常不是线程安全的；

对于静态域，只有是指向基本类型或者不可变对象的引用的常量时，可以通过公有的final域来暴露；

```java
//安全漏洞
public static final Thing[] VALUES = {...};
```

许多IDE产生的方法会返回指向私有数组域的引用，导致安全漏洞，我们可以采用以下两种方式：

```java
//私有化数组
private static final Thing[] PRIVATE_VALUES = {...};
//返回私有化数组的拷贝
public static final Thing[] values() {
	return PRIVATE_VALUES.clone();
}
```

```java
//私有化数组
private static final Thing[] PRIVATE_VALUES = {...};
//公有的不可变的列表
public static final List<Thing> VALUES = 
    Collections.unmodifiableList(Arrays.asList(PRIVATE_VALUES));
```

### 第16条：要在公有类而非公有域中使用访问方法

如果类可以在它所在的包之外进行访问，就提供访问方法（get）；

如果类是包级私有的，或者是私有的嵌套类，直接暴露它的数据域并没有本质的错误；

### 第17条：使可变性最小化

不可变的类比可变化的类更加容易设计、实现和使用，不容易出错且更加安全；

为了使类成为不可变，要遵循下面5条原则：

1. 不要提供任何会修改对象状态的方法
2. 保证类不会被扩展：使用final修饰
3. 声明所有的域都是final
4. 声明所有的域都是私有的
5. 确保对于任何可变组件的互斥访问；

除了使用final修饰以外，还有一种更加灵活的方法可以做到保证类不会被扩展：

```java
public class Complex {
    private final double re;
    private final double im;
    //私有化构造器
    private Complex(double re, double im) {
        this.re = re;
        this.im = im;
    }
    
    public static Complex valueOf(double re, double im) {
        return new Complex(re, im);
    }
}
```

**注：如果要不可变类实现序列化，并且包含一个或者多个指向可变对象的域，就必须提供一个显示的`readObject`或者`readResolve`方法；**

### 第18条：复合优于继承、

继承打破了封装性：子类依赖超类中特定功能的实现细节，超类的实现可能随着发行版本的不同而有所改变，子类可能遭到破坏；

以下类继承了`HashSet<E>`实现一个可以计数的`HashSet`，添加三个值，结果返回6；

```java
public class InstrumentedHashSet<E> extends HashSet<E> {
    private static final long serialVersionUID = 5010872589907607823L;
    private int addCount = 0;

    public InstrumentedHashSet() {
    }

    public InstrumentedHashSet(int initCap, float loadFactor) {
        super(initCap, loadFactor);
    }

    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }

    public int getAddCount() {
        return addCount;
    }

    public static void main(String[] args) {
        InstrumentedHashSet<String> s = new InstrumentedHashSet<>();
        s.addAll(Arrays.asList("Snap", "Crackle", "Pop"));
        log.info(String.valueOf(s.getAddCount()));
    }
}
----------------------------------------------------------------
[23:32:22:623] [INFO] - com.effectivejava.learn.method.InstrumentedHashSet.main(InstrumentedHashSet.java:46) - 6
```

复合示例：

```java
public class InstrumentedSet<E> extends ForwardingSet<E> {
    private int addCount = 0;

    public InstrumentedSet(Set<E> s) {
        super(s);
    }

    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }

    public int getAddCount() {
        return addCount;
    }

    public static void main(String[] args) {
        InstrumentedSet<String> s = new InstrumentedSet<>(new HashSet<>());
        s.addAll(Arrays.asList("Snap", "Crackle", "Pop"));
        log.info(String.valueOf(s.getAddCount()));
    }
}
public class ForwardingSet<E> implements Set<E> {
    private final Set<E> s;

    public ForwardingSet(Set<E> s) {
        this.s = s;
    }

    @Override
    public boolean contains(Object o) {
        return s.contains(o);
    }

    @Override
    public int size() {
        return s.size();
    }

    @Override
    public boolean isEmpty() {
        return s.isEmpty();
    }

    @Override
    public Iterator<E> iterator() {
        return s.iterator();
    }

    @Override
    public Object[] toArray() {
        return s.toArray();
    }

    @Override
    public <T> T[] toArray(T[] a) {
        return s.toArray(a);
    }

    @Override
    public boolean add(E e) {
        return s.add(e);
    }

    @Override
    public boolean remove(Object o) {
        return s.remove(o);
    }

    @Override
    public boolean containsAll(Collection<?> c) {
        return s.containsAll(c);
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        return s.addAll(c);
    }

    @Override
    public boolean retainAll(Collection<?> c) {
        return s.retainAll(c);
    }

    @Override
    public boolean removeAll(Collection<?> c) {
        return s.removeAll(c);
    }

    @Override
    public void clear() {
        s.clear();
    }

    @Override
    public boolean equals(Object o) {
        return s.equals(o);
    }

    @Override
    public int hashCode() {
        return s.hashCode();
    }

    @Override
    public String toString() {
        return s.toString();
    }
}
```

继承的功能非常强大，但是它违背了封装原则。只有当子类和超类之间确实存在子类型关系时，使用继承才是恰当的；

### 第19条：要么设计继承并提供文档说明，要么禁止继承

为了允许继承，**构造器绝不能调用可被覆盖的方法**，由于父类的构造器在子类之前运行，如果调用了子类覆盖的方法，将会先于子类构造器调用

```java
public class Super {
    public Super() {
        overrideMe();
    }

    public void overrideMe() {
    }

    public static final class Sub extends Super {
        private final Instant instant;

        Sub() {
            instant = Instant.now();
        }

        @Override
        public void overrideMe() {
            log.info(String.valueOf(instant));
        }

        public static void main(String[] args) {
            Sub sub = new Sub();
            sub.overrideMe();
        }
    }
}
---------------------------------------------------
[23:54:00:402] [INFO] - com.effectivejava.learn.method.Super$Sub.overrideMe(Super.java:31) - null
[23:54:00:411] [INFO] - com.effectivejava.learn.method.Super$Sub.overrideMe(Super.java:31) - 2020-11-18T15:54:00.404Z
```

为了继承而设计类中实现了`Cloneable`和`Serializable`接口时，无论是`clone`还是`readObject`方法，都不可以调用可覆盖的方法，不管是直接还是间接

为了继承而设计类中实现了`Serializable`接口，且有一个`readResolve`和`writeReplace`方法时，必须使这两个方法成为受保护的方法；

由此可以看出，为了继承而设计的类，对这个类会有一些实质性的限制；

**对于那些并非为了安全的进行子类化而设计和编写文档的类要禁止子类化；**

1. **如果类实现了某个可以反映其本质的接口，如Set、List或者Map，可以使用第18条中的方法；**
2. **如果类没有实现标准化的接口，一种合理的办法是确保这个类永远不会调用它的任何可覆盖的方法，即完全消除这个类可覆盖方法的自用特性；**

除非真正需要子类，否则最后将类声明为final，或者确保没有可访问的构造器来禁止类被继承

### 第20条：接口优于抽象类

**骨架实现：集接口多继承的优势与抽象类可以减少重复代码的优势于一体**

接口与抽象类的优劣：

假设，我们要做一个苹果自动贩卖机 (自动贩卖机简称贩卖机) 和葡萄贩卖机。

首先，定义一个贩卖机的协议 ：

```java
public interface IVending {
    /**
     * 启动机器.
     */
    void start();

    /**
     * 贩卖水果.
     */
    void process();

    /**
     * 暂停机器.
     */
    void stop();
}
```

下一步，我们先来创建，苹果贩卖机

```java
public class AppleVending implements IVending {
    
	@Override
    public void start() {
        log.info("启动，贩卖机。");
    }
    
    @Override
    public void process() {
        log.info("正在卖苹果....");
    }
    
    @Override
    public void stop() {
        log.info("关闭，贩卖机。");
    }
}
```

下一步，我们再来创建，葡萄贩卖机，代码如下：

```java
public class GrapeVending implements IVending {
 
   @Override
    public void start() {
        log.info("启动，贩卖机。");
    }
    
    @Override
    public void process() {
        log.info("正在卖葡萄....");
    }
    
    @Override
    public void stop() {
        log.info("关闭，贩卖机。");
    }
}
```

**两种贩卖机的，start() 和 stop() 方法的代码，重复了。这就是使用接口，带来的问题。共同的协议行为有了，但是，相同的动作，却要重复的写，可以采用抽象类的方法，来解决这一问题，因为抽象类可以包含实现，而接口不行**

此时我们可以使用骨架实现，即先使用一个抽象类实现接口中重复部分的代码

```java
public abstract class AbstractVending implements IVending {
    /**
     * 启动机器.
     */
    @Override
    public void start() {
        log.info("启动，贩卖机。");
    }

    /**
     * 暂停机器.
     */
    @Override
    public void stop() {
        log.info("关闭，贩卖机。");
    }
}
```

再继承这个抽象类

```java
public class AppleVending extends AbstractVending {
    
    @Override
    public void process() {
        log.info("正在卖苹果....");
    }

    public static void main(String[] args) {
        AppleVending appleVending = new AppleVending();
        appleVending.start();
        appleVending.process();
        appleVending.stop();
    }
}
-------------------------------------------------------------------
[21:14:09:953] [INFO] - com.effectivejava.learn.method.AbstractVending.start(AbstractVending.java:18) - 启动，贩卖机。
[21:14:09:955] [INFO] - com.effectivejava.learn.method.AppleVending.process(AppleVending.java:15) - 正在卖苹果....
[21:14:09:955] [INFO] - com.effectivejava.learn.method.AbstractVending.stop(AbstractVending.java:26) - 关闭，贩卖机。
```

### 第21条：为后代设计接口

**java8中的缺省方法：其允许在接口中增加新的方法，并自动在接口实现中可用，因此无需修改实现类。通过这种方式，无需重构之前的实现，优雅地实现了向后兼容**

```java
public interface IVending {
    /**
     * 启动机器.
     */
    default void start() {
        System.out.println("启动，贩卖机。");
    }

    /**
     * 贩卖水果.
     */
    void process();

    /**
     * 暂停机器.
     */
    default void stop() {
        System.out.println("关闭，贩卖机。");
    }
}
```

```java
public class AppleVending implements IVending {

    @Override
    public void process() {
        log.info("正在卖苹果....");
    }

    public static void main(String[] args) {
        AppleVending appleVending = new AppleVending();
        appleVending.start();
        appleVending.process();
        appleVending.stop();
    }
}
```

有了缺省方法，接口的现有实现就不会出现编译时没有报错或警告，运行时却失败的情况，但谨慎设计接口仍然是至关重要的；**尽量避免利用缺省方法再现有接口上添加新的方法，除非有特殊的需要，就算在那样的情况下也应该慎重的考虑：缺省方法实现是否会破坏现有的接口实现；然而在创建接口的时候，用缺省方法提供标准的方法实现是非常方便的；**

**java8中的静态接口方法：static方法不属于特定对象，不属于实现接口的类API的一部分，只能通过接口名称直接调用**

```java
public interface Vehicle {
     
    // regular / default interface methods
     
    static int getHorsePower(int rpm, int torque) {
        return (rpm * torque) / 5252;
    }
}
```

```java
public static void main(String[] args) { 
    Vehicle.getHorsePower(2500, 480));
}
```

### 第22条：接口只用于定义类型

接口不应该用来导出常量，例如：

```java
public interface PhysicalConstants {
    static final double AVOGADROS_NUMBER = 6.022_140_857e23;
    
    static final double BOLTZMANN_CONSTANT = 1.380_648_52e-23;
    
    static final double ELECTRON_MASS = 9.109_383_56e-31;
}
```

**注意，有时会在数字的字面量中使用下划线(_)，从java7开始，下划线的使用已经合法了，它对数字字面量的值没有影响，使用得当可以极大的提升可读性：**

- **如果包含五个或者五个以上连续数字，无论浮点还是定点，都要考虑在数字的字面量中添加下划线；**
- **对于基数为10的字面量，无论是整形还是浮点，都应该使用下划线把数字隔成每三位一组，表示一千的正负倍数；**

正确的常量工具类：

```java
public class PhysicalConstants {
    private PhysicalConstants() {}
    
    public static final double AVOGADROS_NUMBER = 6.022_140_857e23;

    public static final double BOLTZMANN_CONSTANT = 1.380_648_52e-23;

    public static final double ELECTRON_MASS = 9.109_383_56e-31;
}
```

使用时，可以利用`静态导入`机制，避免用类名修饰常量名：

```java
import static com.effectivejava.learn.method.PhysicalConstants.ELECTRON_MASS;

public static void main(String[] args) {
    log.info(String.valueOf(ELECTRON_MASS));
}
-----------------------------------------------------------------------------
[22:10:22:940] [INFO] - com.effectivejava.learn.method.AppleVending.main(AppleVending.java:26) - 9.10938356E-31
```

### 第24条：静态成员类优于非静态成员类

嵌套类是指定义在另一个类的内部的类；嵌套类存在的目的应该只是为它的外围类提供服务，如果嵌套类可能会用于其他的某个环境中，它就应该是顶层类；

嵌套类有四种：**静态成员类**、**非静态成员类**、**匿名类**和**局部类**；除了第一种外，其他的都称为内部类；

**静态成员类**通常的一个用法是作为一个公有的辅助类，只有与它的外围类一起用时才有意义；

**非静态成员类**的一个通常的用法是，定义一个适配器

从语法上来说，静态成员类和非静态成员类之间的区别是，静态成员类在声明上有static标识符，非静态成员类的每个实例都是隐式地和它的外围类的实例关联在一起，在非静态成员类的实例方法内部，你可以调用外围实例的方法，或者通过标识了this的构造来获取外围实例的引用；

非静态成员类实例和它的外围实例的关联可以手动使用表达式`enclosingInstance.new MemberClass(args)`来建立，但是很少这么做。正如你所料，这个关联需要消耗非静态成员类实例的空间，而且增加了它的构建时间。

**如果你声明了一个不需要访问外围实例的成员类，那你总是应该static修饰符加到声明里去**，使得这个成员类是静态的；如果你不加这个修饰符，那么每个实例都将包含一个隐藏的外围实例的引用。就如前面说的那样，存储这个引用将耗费时间和空间。更严重的是，当这个外围实例已经满足垃圾回收的条件时，非静态成员类实例会导致外围实例被保留。因此而导致的内存泄露是灾难性的。由于这个引用是不可见的，所以这个问题也通常很难被检测到

在lambda表达式加入Java（第6章）之前，**匿名类**是创建小的函数对象和处理对象的首选方式，但现在lambda表达式是首选了（条目42）。匿名类的另一个常见用法是实现静态工厂方法（见条目20的`intArrayAsList`）

在四种嵌套类里，**局部类**是最不常用的

**如果一个嵌套类必须在方法外部可见，或者放在方法内部会显得太长时，就使用成员类。如果成员类的实例需要拥有该类的外围类的引用，就将其做成非静态；不然，就将其做成静态。假设一个类应当在方法内部，若你需要只从一个地方创建实例而且已经存在一个类型能说明这个类的特征，那么将其做成匿名类；否则，就将其做成局部类。**

### 第25条：限制源文件为单个顶级类

**永远不要将多个顶级类或接口放到一个源文件里**