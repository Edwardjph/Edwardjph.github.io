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