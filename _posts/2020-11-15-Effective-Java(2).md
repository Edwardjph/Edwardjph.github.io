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

## 第五章 泛型

### 第26条：不要使用原始类型

每个泛型类型定义了一个原始类型，即不带任何类型参数的泛型类型的名称。例如，List<E>对应的原始类型是List，它们的存在主要是为了兼容那些之前在泛型未出现时写的代码；

**如果你使用了原始类型，你将会失去泛型所带来的安全性和可读性**

可以使用参数化类型以允许插入任意对象，例如List<Object>，原始类型List和参数化类型List<Object>之间的区别是什么呢？不严格地说，前者不受泛型类型系统检查，而后者则显示地告诉编译器它可以接受任意类型的对象；

```java
public static void main(String[] args) {
    List<String> strings = new ArrayList<>();
    unsafeAdd(strings, Integer.valueOf(42));
    String s = strings.get(0);
}

private static void unsafeAdd(List list, Object o) {
    list.add(o);
}
---------------------------------------------------------------
Exception in thread "main" java.lang.ClassCastException: java.lang.Integer cannot be cast to java.lang.String
	at com.effectivejava.learn.method.Test.main(Test.java:16)
```

如果你想使用泛型类型，但你不知道或者不关心实际的类型尝试是什么，你可以用一个问好来替代。例如，泛型类型`Set<E>`的无限制通配符类型是`Set<?>`（读作，某些类型的集合）,通配符是安全的而原始类型则不是，但你无法将任意元素（null除外）放入一个`Collection<?>`

对于不应该使用原始类型这条规则，有两个小例外。**你必须在类字面值中使用原始类型**，换句话说，List.class，String[].class以及int.class都是合法的，但List<String>.class和List<?>.class则不是

第二个例外则与instanceof操作符有关。因为泛型类型信息在运行时是被擦除了的，所以在参数化类型而不是无限制通配符类型上用instanceof操作符是非法的。

```java
if (o instanceof Set) {
    Set<?> s = (Set<?>) o;
}
```

### 第27条：消除未检查警告

要尽可能消除每一个非检查警告；

如果你无法消除某个警告，但你能证明引起这个警告的代码是类型安全的话，那么这时候（也只有这时候）可以用@SuppressWarnings("unchecked")注解来禁止这个警告；

应该在尽可能小的作用域上使用SuppressWarnings注解。它通常是个变量声明或者是一个非常短的方法或者是构造器。永远不要在整个类上使用SuppressWarnings注解，这么做会掩盖某些重要的警告。

```java
// Adding local variable to reduce scope of @SuppressWarnings
public <T> T[] toArray(T[] a) {
    if (a.length < size) {
        // This cast is correct because the array we're creating
        // is of the same type as the one passed in, which is T[].
        @SuppressWarnings("unchecked") 
        T[] result = (T[]) Arrays.copyOf(elements, size, a.getClass());
        return result;
    }
    System.arraycopy(elements, 0, a, 0, size); 
    if (a.length > size)
        a[size] = null; 
    return a;
}
```

### 第28条：列表优先于数组

数组与泛型类型的不同体现在两个方面：

**首先，数组是协变的（covariant）**，意味着，如果Sub是Super的一个子类型，那么数组类型Sub[]也是数组类型Super[]的子类型。相反，泛型是有约束的：对于任意的两个不同的类型Type1和Type2，List<Type1>既不是List<Type2>的子类也不是它的父类；

你可能想，这是不是意味着泛型是有缺陷的，但实际上，可以说数组才是有缺陷的；

下面的代码片段是合法的：

```java
// Fails at runtime!
Object[] objectArray = new Long[1];
objectArray[0] = "I don't fit in"; // Throws ArrayStoreException
```

但下面这个则不是：

```java
// Won't compile!
List<Object> ol = new ArrayList<Long>(); // Incompatible types
ol.add("I don't fit in");
```

任意一种方式你都无法将一个String放入一个Long容器里，但如果用数组的话你会在运行时才发现你犯了个错误，而如果用List，你就会在编译时发现它；

**数组和泛型的第二个主要区别是，数组是具化的**，意味着，数组在运行时才知道并检查元素类型；泛型则是通过擦除来实现的，意味着泛型仅在编译时进行类型约束的检查并且在运行是忽略（或擦除）元素类型

当你强转成数组类型时，若得到一个泛型数组创建错误或者未检查强转警告，最好的解决办法是，总是优先采用集合类型List<E>

```java
public class Chooser<T> {
    private final List<T> choiceList;

    public Chooser(Collection<T> choices) {
        choiceList = new ArrayList<>(choices);
    }

    public T choose() {
        Random rnd = ThreadLocalRandom.current();
        return choiceList.get(rnd.nextInt(choiceList.size()));
    }
}
```

### 第29条：优先考虑泛型

泛型类型比需要客户端代码中强制转换的类型更安全、更易于使用。设计新类型时，请确保无需此类强制转换即可使用。这通常意味着使类型泛型

```java
public class Stack<E> {
    private E[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    @SuppressWarnings("unchecked")
    public Stack() {
        elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(E e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public E pop() {
        if (size == 0) {
            throw new EmptyStackException();
        }
        E result = elements[--size];
        elements[size] = null;
        return result;
    }

    private void ensureCapacity() {
        if (elements.length == size) {
            elements = Arrays.copyOf(elements, 2 * size + 1);
        }
    }

    public boolean isEmpty() {
        return size == 0;
    }

    public static void main(String[] args) {
        Stack<String> stack = new Stack<>();
        for (String arg : args) {
            stack.push(arg);
        }
        while (!stack.isEmpty()) {
            log.info(stack.pop().toUpperCase());
        }
    }
}
```

无需显式强制转换来调用从堆栈中弹出的元素上的 String 的 toUpperCase 方法，并且自动生成的强制转换保证成功

### 第30条：优先使用泛型方法

```java
public static <E> Set<E> union(Set<E> s1, Set<E> s2) {
    Set<E> result = new HashSet<>(s1);
    result.addAll(s2);
    return result;
}
// Simple program to exercise generic method
public static void main(String[] args) {
    Set<String> guys = Set.of("Tom", "Dick", "Harry");
    Set<String> stooges = Set.of("Larry", "Moe", "Curly");
    Set<String> aflCio = union(guys, stooges);
    System.out.println(aflCio);
}
```

有时，你会需要创建一个不可变的对象，但又希望它适用于不同的类型

定义了一个泛型接口，里面只有一个apply()方法

```java
public interface GenericityIn<T> {
    T apply(T args);
}
```

假如我现在需要一个对String类型数据进行apply操作的单例对象。按惯性思维，我们会这么实现

```java
public class GenericityStr implements GenericityIn<String>{
    private static GenericityStr strInstance = new GenericityStr();

    private GenericityStr(){
    }

    public GenericityStr getInstance() {
        return strInstance;
    }

    @Override
    public String apply(String args) {
        // TODO Auto-generated method stub
        return args;
    }

}
```

这种模式没有考虑到代码的扩展性，当未来需要对Integer对象数据进行apply操作呢？再写一个类吗？这明显是不合理的。这时候就需要再结合工厂的设计模式了

```java
public class GenericityImp {
    private static GenericityIn<Object> IDENTITY_FUNCTION = new GenericityIn<Object>() {
        public Object apply(Object arg) {
            return arg;
        }
    };

    /* 泛型单例工厂 */
    /* 生成不同数据类型参数集合的GenericityIn类 */
    /*
     * 节省GenericityIn对象生成时左右两边都需写类型 如：UnaryFunction<String> function = new UnaryFunction<String>(){};
     */
    @SuppressWarnings("unchecked")
    public static <T> GenericityIn<T> identityFunction() {
        return (GenericityIn<T>) IDENTITY_FUNCTION;
    }

    public static void main(String[] args) {
        String[] strings = { "jute", "hemp", "nylon" };
        GenericityIn<String> sameString = identityFunction();
        GenericityIn<Integer> sameInteger = identityFunction();
        for (String s : strings) {
            System.out.println(sameString.apply(s));
        }

    }
}
```

递归类型限制：通过某个包含类型参数本身的表达式来限制类型参数

```java
<E extends Comparable<E>>
```

### 第31条：利用有限制通配符来提升API的灵活性

假设我们需要添加一个方法，这个方法接受一系列元素并将这些元素推入栈顶。以下是第一个尝试

```java
public void pushAll(Iterable<E> src) {
    for (E e : src) {
        push(e);
    }
}
```

这个方法编译时不会有任何问题，但它并不完全满足我们的需求：

假设你有一个`Stack<Number>`并且调用了`push(intVal)`，`intVal`是`Integer`类型。这也应该能运行，因为`Integer`是`Number`的子类型。所以从逻辑上看，这么做也应该是可行的；

```java
Stack<Number> numberStack = new Stack<>();
Iterable<Integer> integers = ... ;
numberStack.pushAll(integers);
```

假如你真这么做，你将会得到以下的错误信息，因为参数化类型是不可变的（对于任意两个不同的类型`Type1`和`Type2`，`List<Type1>`既不是`List<Type2>`的子类型也不是`List<Type2>`的父类型）

```java
StackTest.java:7: error: incompatible types: Iterable<Integer>
cannot be converted to Iterable<Number>
numberStack.pushAll(integers);
```

Java提供了一种特殊的参数化类型来处理这类情况，它就是有限制通配符类型；

pushAll方法的输入参数类型不应该是“E类型的Iterable”，而应该是“E的子类型的Iterable”，并且有一个通配符类型能恰当地表述这一点：Iterable<? extends E>

```java
 public void pushAll(Iterable<? extends E> src) {
        for (E e : src) {
            push(e);
        }
    }
```

在假设你想给`pushAll`方法写一个对应的`popAll`方法。`popAll`方法将栈里的每一个元素依次弹出，并将弹出的元素添加到指定的集合里去。我们先来试着写`popAll`方法的第一个版本:

```java
public void popAll(Collection<E> dst) {
        while (!isEmpty()) {
            dst.add(pop());
        }
    }
```

上面的方法编译也没有任何问题，而且在指定集合的元素类型与栈的元素类型完全一致的情况下，这个方法也能正确运行。但它也同样不能完全满足我们的要求。假设你有一个Stack<Number>和一个类型为Object的变量

```java
Stack<Number> numberStack = new Stack<Number>(); 
Collection<Object> objects = ... ; 
numberStack.popAll(objects);
```

如果你尝试编译基于前面所示的`popAll`方法写出的客户端代码，你将会得到一个类似于第一个版本的`pushAll`方法所遇到的错误：`Collection<Object>`不是`Collection<Number>`的子类型。同样，通配符类型解决了这个问题。`popAll`方法的输入参数类型不应该是“E的集合”，而应该是“E的父类型的集合”;有一个通配符类型可以恰当地表示这一点：`Collection<? super E>`

```java
public void popAll(Collection<? super E> dst) {
        while (!isEmpty()) {
            dst.add(pop());
        }
    }
```

**结论很明显。为了最大的灵活性，请在那些表示生产者或者消费者的输入参数上使用通配符类型**；以下是一个助记符，它帮助你记住应该使用哪种通配符类型

**PECS代表for producer-extends, consumer-super。**换句话说，如果一个参数化类型代表一个T类型的生产者，则使用<? extends T>；如果一个参数化类型代表一个T类型的消费者，则使用<? super T>

现在我们来看看条目30的union方法，参数s1和参数s2都是E类型的生产者，所以根据PECS助记符，上述声明应该修改成以下的样子

```
public static <E> Set<E> union(Set<? extends E> s1, Set<? extends E> s2)
```

**但返回类型不要用有限制通配符类型。**因为这将强制客户端用通配符类型，而不是给它们提供额外的灵活性

使用修改后的声明，代码可以完美编译通过

```java
Set<Integer> integers = Set.of(1, 3, 5);
Set<Double> doubles = Set.of(2.0, 4.0, 6.0); 
Set<Number> numbers = union(integers, doubles);
```

**一般来说，如果类型参数只在方法声明中出现一次，就可以用通配符来代替它**

```java
public static <E> void swap(List<E> list, int i, int j);
//在公共API中，这种更好一些，因为它更简单
public static void swap(List<?> list, int i, int j);
```

但第二种声明会有一个问题，以下实现将不能编译：

```java
 public static void swap(List<?> list, int i, int j){
        list.set(i, list.set(j, list.get(i)));
    }
```

因为你不能将null以外的任何值放到List<?>中，我们需要使用辅助方法来实现：

```java
public static void swap(List<?> list, int i, int j) {
    swapHelper(list, i, j);
}
// Private helper method for wildcard capture
private static <E> void swapHelper(List<E> list, int i, int j){
    list.set(i, list.set(j, list.get(i)));
}
```

这样可以隐藏复杂的泛型方法，导出基于通配符的声明

### 第32条：谨慎并用泛型和可变参数

当调用一个可变参数方法时，会创建一个数组用来存放可变参数；

当一个参数化类型的变量指向一个不是该类型的对象时，会产生堆污染（heap pollution）；会导致编译器的自动生成转换失败，破坏泛型系统的基本保证；

```java
public static void dangerous(List<String>... stringLists) {
    List<Integer> integerList = Collections.singletonList(42);
    Object[] objects = stringLists;   //自动生成数组
    objects[0] = integerList;		  //堆污染 stringLists[0]=integerList
    String s = stringLists[0].get(0); //ClassCastException
}
=============================================================
java.lang.ClassCastException: java.lang.Integer cannot be cast to java.lang.String
```

泛型可变参数方法在下列条件下是安全的：

1. 它没有在可变参数数组中保存任何值
2. 它没有对不被信任的代码开放该数组（或者其克隆程序）

SafeVarargs注解，它让带泛型vararg参数的方法设计者能够自动禁止客户端的警告；

SafeVarargs注解只能用在无法不能被重写的方法上，因为它不能保证每个可能的重写方法都是安全的；在 Java 8 中，注解仅在静态方法和 `final` 实例方法上合法; 在 Java 9 中，它在私有实例方法中也变为合法；

可以用一个List参数代替可变参数（这是一个伪装数组）：

```java
static <T> List<T> flatten(List<List<? extends T>> lists) {
    List<T> result = new ArrayList<>();
    for (List<? extends T> list : lists) {
        result.addAll(list);
    }
    return result;
}
```

### 第33条：优先考虑安全的异构容器

考虑以下这样的场景：假设我的一个应用中有很多类都是"无状态"的，并且实例化一个这种类是很费资源的，更糟的是发现这些类没有一个是单例的，这个时候可以考虑在外部编写一个单例的缓存，使用这个缓存来维护这些类的实例;

这个时候就可以考虑不将缓存泛型化，而将缓存的键泛型化:

```java
public class SingletonClassCache {
    private Map<Class<?>, Object> cache = new HashMap<>();

    public <T> void put(Class<T> type, T instance) {
        cache.put(Objects.requireNonNull(type), instance);
    }

    public <T> T get(Class<T> type) {
        return type.cast(cache.get(type));
    }
}
```

`SingletonClassCache`是类型安全的，同时也是异构的（所有的建都有不同的类型）