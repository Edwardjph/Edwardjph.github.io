# 第二章 创建和销毁对象

### 第1条：用静态工厂方法代替构造器

**注：静态工厂方法与设计模式中的工厂方法模式不同**

```java
public static Boolean valueOf(boolean b) {
    return b ? Boolean.TRUE : Boolean.FALSE;
}
```

静态工厂方法与构造器的不同：

优势：

- 有名称
- 不必在每次调用的时候都创建一个新的对象
- 可以返回原返回类型的任何子类型的对象
- 返回的对象的类可以随着每次调用二发生变化，这取决于静态工厂方法的参数值
- 方法返回的对象所属的类，在编写包含该静态工厂方法的类时可以不存在

缺点：

- 类如果不含公有的或者受保护的构造器，就不能被子类化
- 不易被发现

静态工厂方法的一些惯用名称：

- `from`——类型转换方法，它只有单个参数，返回该类型的一个相对应的实例

  ```java
  Date d = Date.from(instant);
  ```

- `of`——聚合方法，带有多个参数，返回该类型的一个实例，把他们合并起来

  ```java
  Set<Rank> faceCards = EnumSet.of(JACK, QUEEN, KING);
  ```

- `valueOf`——比`from`和`of`更加繁琐的一种替代方法

  ```java
  BigInteger prime = BigInteger.valueof(Integer.MAX_VALUE);
  ```

- `instance`或者`getInstance`——返回的实例是通过方法的参数来描述的，但是不能说与参数具有同样的值

  ```java
  StackWalker luke = StackWalker.getInstance(options);
  ```

- `create`或者`newInstance`——像`instance`或者`getInstance`一样，但能够确保每次调用都返回新的实例

  ```java
  Object newArray = Array.newInstance(classObject, arrayLen);
  ```

- `getType`——像`getInstance`一样，但是在工厂方法处于不同的类中的时候使用，`Type`表示工厂方法所返回的对象类型

  ```java
  FileStore fs = Files.getFileStore(path);
  ```

- `nweType`——像`newInstance`一样，但是在工厂方法处于不同的类中的时候使用，`Type`表示工厂方法所返回的对象类型

  ```java
  BufferedReader br = Files.newBufferedReader(path);
  ```

- `type`——`getType`和`newType`的简版

  ```java
  List<Complaint> litany = Collections.list(legaryLitany);
  ```

**静态工厂方法和公有构造器各有用处，静态工厂经常更加合适**

### 第2条：遇到多个构造器参数时要考虑使用构建器

**JavaBean模式(get/set)有很严重的缺点：**

- 因为构造过程被分到了几个调用中，在构造过程中JavaBeans可能处于不一致的状态
- JavaBeans模式使得把类做成不可变的可能性不复存在了

**建造者模式(Builder)**

```java
public class NutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;
    private final int fat;
    private final int sodium;
    private final int carbohydrate;

    public static class Builder {
        //Required parameters
        private final int servingSize;
        private final int servings;

        //Optional parameters - initialized to default values
        private int calories = 0;
        private int fat = 0;
        private int sodium = 0;
        private int carbohydrate = 0;

        public Builder(int servingSize, int servings) {
            this.servingSize = servingSize;
            this.servings = servings;
        }

        public Builder calories(int val) {
            calories = val;
            return this;
        }

        public Builder fat(int val) {
            fat = val;
            return this;
        }

        public Builder sodium(int val) {
            sodium = val;
            return this;
        }

        public Builder carbohydrate(int val) {
            carbohydrate = val;
            return this;
        }

        public NutritionFacts build() {
            return new NutritionFacts(this);
        }
    }

    private NutritionFacts(Builder builder) {
        servingSize = builder.servingSize;
        servings = builder.servings;
        calories = builder.calories;
        fat = builder.fat;
        sodium = builder.sodium;
        carbohydrate = builder.carbohydrate;
    }
}
```

**Builder模式也适用于类层次结构**

```java
public abstract class Pizza {
    public enum Topping {HAM, MUSHROOM, ONION, PEPPER, SAUSAGE}

    //set无序唯一的集合
    final Set<Topping> toppings;

    abstract static class Builder<T extends Builder<T>> {
        //EnumSet.noneOf返回一个空的EnumSet
        EnumSet<Topping> toppings = EnumSet.noneOf(Topping.class);

        public T addTopping(Topping topping) {
            //Objects.requireNonNull检查对象是否为空
            toppings.add(Objects.requireNonNull(topping));
            return self();
        }

        abstract Pizza build();

        protected abstract T self();
    }

    Pizza(Builder<?> builder) {
        toppings = builder.toppings.clone();
    }
    
    public static void main(String[] args) {
        NyPizza nyPizza = new NyPizza.Builder(NyPizza.Size.SMALL)
                .addTopping(Topping.ONION)
                .addTopping(Topping.MUSHROOM)
                .build();
        Calzone calzone = new Calzone.Builder()
                .addTopping(Topping.HAM)
                .sauceInsdie()
                .build();
        //NyPizza{size=SMALL, toppings=[MUSHROOM, ONION]}
        System.out.println(nyPizza.toString());
        //Calzone{sauceInsdie=true, toppings=[HAM]}
        System.out.println(calzone.toString());
    }
}
```

```java
public class NyPizza extends Pizza {
    public enum Size {SMALL, MEDIUM, LARGE}

    private final Size size;

    public static class Builder extends Pizza.Builder<Builder> {
        private final Size size;

        public Builder(Size size) {
            this.size = Objects.requireNonNull(size);
        }

        @Override
        public NyPizza build() {
            return new NyPizza(this);
        }

        @Override
        protected Builder self() {
            return this;
        }
    }

    private NyPizza(Builder builder) {
        super(builder);
        size = builder.size;
    }

    @Override
    public String toString() {
        return "NyPizza{" +
                "size=" + size +
                ", toppings=" + toppings +
                '}';
    }
}
```

```java
public class Calzone extends Pizza{
    private final boolean sauceInsdie;

    public static class Builder extends Pizza.Builder<Builder> {
        private boolean sauceInsdie = false;

        public Builder sauceInsdie() {
            sauceInsdie = true;
            return this;
        }

        @Override
        public Calzone build() {
            return new Calzone(this);
        }

        @Override
        protected Builder self() {
            return this;
        }
    }

    Calzone(Builder builder) {
        super(builder);
        sauceInsdie = builder.sauceInsdie;
    }

    @Override
    public String toString() {
        return "Calzone{" +
                "sauceInsdie=" + sauceInsdie +
                ", toppings=" + toppings +
                '}';
    }
}

```

**builder模式的客户端代码比重叠构造器更易于阅读与编写，也比JavaBean模式更加安全；**

**但是builder模式比重叠构造器模式更加冗长，且创建对象时必须先创建他的构建器，虽然开销不大，但在十分注重性能的情况下就有问题**

**所以只在有很多参数的时候使用，比如4个或者更多**

#### 补充：java泛型

泛型提供了编译时类型安全检测机制，该机制允许程序员在编译时检测到非法的类型

**定义泛型方法的规则：**

- 所有泛型方法声明都有一个类型参数声明部分（由尖括号分隔），该类型参数声明部分在方法返回类型之前（在下面例子中的<E>）。
- 每一个类型参数声明部分包含一个或多个类型参数，参数间用逗号隔开。一个泛型参数，也被称为一个类型变量，是用于指定一个泛型类型名称的标识符。
- 类型参数能被用来声明返回值类型，并且能作为泛型方法得到的实际参数类型的占位符。
- 泛型方法体的声明和其他方法一样。注意类型参数只能代表引用型类型，不能是原始类型（像int,double,char的等）。

```java
public static <E> void printArray(E[] inputArray) {
    for (E element : inputArray) {
        System.out.printf("%s", element);
    }
}
```

**有界的类型参数:**

声明一个有界的类型参数，首先列出类型参数的名称，后跟extends关键字，最后紧跟它的上界

```java
public static <T extends Comparable<T>> T maximum(T x, T y, T z) {
    T max = x;
    if (y.compareTo(max) > 0) {
        max = y;
    }
    if (z.compareTo(max) > 0) {
        max = z;
    }
    return max;
}
```

**类型通配符：**

一般是使用?代替具体的类型参数，例如 **`List<?>`** 在逻辑上是**`List<String>`,`List<Integer>`** 等所有List<具体类型实参>的父类。

**<? extends T>和<? super T>的区别**

- <? extends T>表示该通配符所代表的类型是T类型的子类。
- <? super T>表示该通配符所代表的类型是T类型的父类。

#### 补充：Java 中 Comparable 和 Comparator 比较

Comparable 是**排序**接口，若一个类实现了Comparable接口，就意味着“该类支持排序”

Comparator 是比较器接口，我们若需要控制某个类的**次序**，而该类本身不支持排序(即没有实现Comparable接口)；那么，我们可以建立一个“该类的比较器”来进行排序。这个“比较器”只需要实现Comparator接口即可

```java
public class Person implements Comparable<Person>{
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public int compareTo(Person o) {
        return name.compareTo(o.name);
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }

    private static class AscAgeComparator implements Comparator<Person> {

        @Override
        public int compare(Person o1, Person o2) {
            return o1.age - o2.age;
        }
    }

    private static class DescAgeComparator  implements Comparator<Person> {

        @Override
        public int compare(Person o1, Person o2) {
            return o2.age - o1.age;
        }
    }

    public static void main(String[] args) {
        List<Person> personList = new ArrayList<>();
        personList.add(new Person("ccc", 20));
        personList.add(new Person("AAA", 30));
        personList.add(new Person("bbb", 10));
        personList.add(new Person("ddd", 40));
		//c,a,b,d
        System.out.printf("%s\n", personList);
		//a,b,c,d
        Collections.sort(personList);
        System.out.printf("%s\n", personList);
		//b,c,a,d
        Collections.sort(personList, new AscAgeComparator());
        System.out.printf("%s\n", personList);
		//d,a,c,b
        Collections.sort(personList, new DescAgeComparator());
        System.out.printf("%s\n", personList);
    }
}
```

### 第3条：用私用构造器或者枚举类型强化Singleton属性

三种单例实现方式：

最简单的方式

```java
//Singleton with public final field
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() {}
}
```

更加灵活

```java
//Singleton with static factory
public class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() {} 
    public static Elvis getInstance() {
        return INSTANCE;
    }
}
```

这两种方式都可以通过反射调用私有的构造器，要抵御这种攻击，可以修改构造器，让它在被要求创建第二个实例的时候抛出异常；

且这两种方式实现可序列化时，不止需要`implements Serializable`，还需要提供一个`readResolve`方法；否则，每次反序列化一个序列化的实例时，都会创建一个新的实例

```java
	//readResolve method to preserve singleton property
    private Object readResolve() {
        //Return the one true Elvis and let the garbage collector
        //take care of the Elvis impersonator
        return INSTANCE;
    }
```

最后一种方法，还没有广泛采用，但是实现单例最好的方法，更加的简洁，且提供序列化机制

```java
//Enum singleton - the preferred approach
public enum Elvis {
    INSTANCE;
}
```

### 第4条：通过私有构造器强化不可实例化能力

如果一个类不希望被实例化，应显示的包含一个私有构造器，且为了防止内部调用，抛出异常

```java
//Noninstantiable utility class
public class UtilityClass {
    //Suppress default constructor for noninstantiability
    private UtilityClass() {
        throw new AssertionError();
    }
}
```

### 第6条：避免创建不必要的对象

```java
String s = new String("bikini");//Don`t do this
```

要优先使用静态工厂方法而不是构造器，避免创建不必要的对象；例如：`Boolean.valueOf(String)`优先于`Boolean(String)`

要优先使用基础类型而不是装箱类型，要当心无意识的自动装箱

### 第7条：消除过期对象的引用

```java
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;
    
    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }
    
    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }
    
    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        //此处返回值后，elements[--size]将再也不会被使用，但不会被垃圾回收器回收
        return elements[--size];
    }

    private void ensureCapacity() {
        if (elements.length == size) {
            elements = Arrays.copyOf(elements, 2 * size + 1);
        }
    }
}
```

应修改为：

```java
public Object pop() {
    if (size == 0)
        throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null;
    return result;
}
```

清空对象引用应该是一种例外，而不是一种规范行为；

只要类是自己管理内存，就应该警惕内存泄漏问题；

### 第9条：try-with-resources优先于try-finally

```java
public static String firstLineOfFile(String path) throws IOException {
    BufferedReader br = new BufferedReader(new FileReader(path));
    try {
        return br.readLine();
    } finally {
        br.close();
    }
}
```

`try-finally`语句存在着些许不足，因为在`try`和`finally`中都有可能会抛出异常，当同时抛出异常时，`close`的异常将抹除`readline`的异常，这将使得调试变得非常复杂；

使用`try-with-resources`解决这些问题，将自动关闭资源，且当同时抛出异常时，`close`的异常将会被禁止，但还是可以通过编程调用`getSuppressed`方法访问它们

```java
public static String firstLineOfFile(String path) throws IOException {
    try (BufferedReader br = new BufferedReader(
        new FileReader(path))) {
        return br.readLine();
    }
}
```

# 第三章 对于所有对象都通用的方法

### 第10条：覆盖equals时请遵守通用规定

什么时候应该覆盖`equals`方法：类有自己特有的“逻辑相等”概念（不同于对象等同的概念），且超类还没有覆盖`equals`；

覆盖`equals`方法时的通用约定：

- 自反性：`x.equals(x)`必须返回`true`
- 对称性：当`x.equals(y)`返回`true`时，`y.equals(x)`也必须返回`true`
- 传递性：当`x.equals(y)`返回`true`，且`y.equals(z)`也返回`true`时，`x.equals(z)`也必须返回`true`
- 一致性：当对象信息没有被修改时，多次调用应返回同一结果
- `x.equals(null)`必须返回`false`

**以下实例违反了对称性：**

```java
public class CaseInsensitiveString {
    private final String s;

    public CaseInsensitiveString(String s) {
        this.s = Objects.requireNonNull(s);
    }

    @Override
    public boolean equals(Object o) {
        if (o instanceof CaseInsensitiveString)
            return s.equalsIgnoreCase(((CaseInsensitiveString) o).s);
        if (o instanceof String)
            return s.equalsIgnoreCase((String) o);
        return false;
    }
    
    public static void main(String[] args) {
        CaseInsensitiveString hello = new CaseInsensitiveString("Hello");
        String h = "hello";
        //返回true
        System.out.println(hello.equals(h));
        //返回False
        System.out.println(h.equals(hello));
        List<CaseInsensitiveString> list = new ArrayList<>();
        list.add(hello);
        //返回true，此值只是在当前JDK中碰巧返回true，在其他实例中可能返回false也可能抛错
        System.out.println(list.contains(hello));
    }
}

```

解决办法：

```java
@Override
public boolean equals(Object o) {
    return o instanceof CaseInsensitiveString &&
        ((CaseInsensitiveString) o).s.equalsIgnoreCase(s);
}
```

**以下实例违反了传递性：**

```java
@Slf4j
public class ColorPoint extends Point {
    private final Color color;

    public ColorPoint(int x, int y, Color color) {
        super(x, y);
        this.color = color;
    }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Point)) {
            return false;
        }
        //If o is a normal Point; do a color-blind comparison
        if (!(o instanceof ColorPoint)) {
            return o.equals(this);
        }
        //If o is a ColorPoint; do a full comparison
        return super.equals(o) && ((ColorPoint) o).color == color;
    }

    public static void main(String[] args) {
        ColorPoint c1 = new ColorPoint(1, 2, Color.RED);
        Point p = new Point(1, 2);
        ColorPoint c2 = new ColorPoint(1, 2, Color.BLACK);
        log.info(String.valueOf(c1.equals(p)));
        log.info(String.valueOf(p.equals(c2)));
        log.info(String.valueOf(c1.equals(c2)));
    }
}
-------------------------------------------------------------------------------
[00:46:51:816] [INFO] - com.effectivejava.learn.method.ColorPoint.main(ColorPoint.java:39) - true
[00:46:51:817] [INFO] - com.effectivejava.learn.method.ColorPoint.main(ColorPoint.java:40) - true
[00:46:51:817] [INFO] - com.effectivejava.learn.method.ColorPoint.main(ColorPoint.java:41) - false
```

同时这种方法还可能导致无限递归问题：假设`Point`有两个子类，如`ColorPoint`和`SmellPoint`都用这种方式，则`ColorPoint.equals(SmellPoint)` 的调用将抛出`StackOverflowError`；

这是一个在面向对象语言中关于等价关系的一个基本问题：

**我们没法在扩展可实例化的类的同时，既增加新的值组件，同时又保留equals约定**

根据“复合优先于继承”，我们可以在`ColorPoint`中加入一个私有的`Point`域

```java
@Slf4j
public class ColorPoint{
    private final Point point;
    private final Color color;

    public ColorPoint(int x, int y, Color color) {
        point = new Point(x, y);
        this.color = color;
    }
    
    public Point asPoint() {
        return point;
    }

    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof ColorPoint)) {
            return false;
        }
        ColorPoint cp = (ColorPoint) obj;
        return cp.point.equals(point) && cp.color.equals(color);
    }
}
```

在java平台库中，有一些类扩展了可实例化的类，并添加了新的组件。例如：`java.sql.Timestamp`对`java.util.Date`进行了扩展，并添加了`nanoseconds`域。当它们被混在一起的时候，会引起不正确的行为。

**equals方法的诀窍：**

1. 使用==操作符检查“参数是否为这个对象的引用”
2. 使用`instanceof`检查“参数是否为正确的类型“
3. 把参数转换成正确的类型
4. 对于该类中的每个关键域，检查参数中的域是否与该对象中对应的域相匹配

```java
public class PhoneNumber {
    private final short areaCode, prefix, lineNum;

    public PhoneNumber(short areaCode, short prefix, short lineNum) {
        this.areaCode = areaCode;
        this.prefix = prefix;
        this.lineNum = lineNum;
    }

    @Override
    public boolean equals(Object obj) {
        if (obj == this) {
            return true;
        }
        if (!(obj instanceof PhoneNumber)) {
            return false;
        }
        PhoneNumber pn = (PhoneNumber) obj;
        return pn.lineNum == lineNum && pn.areaCode == areaCode
                && pn.prefix == prefix;
    }
}
```

**注意：**

- 覆盖equals时总是要覆盖hashCode
- 不要企图让equals过于智能
- 不要将equals声明中的Object对象替换为其他的类型

#### 补充：springboot2集成log4j2日志框架

**Spring Boot 2.x**默认使用**Logback**日志框架，要使用 **Log4j2**必须先排除 **Logback**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
    <exclusions>
         <!--排除logback-->
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<!--log4j2 依赖-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

在`resources`目录里创建`log4j2.xml`文件

```xml
 <?xml version="1.0" encoding="UTF-8"?>
  <!--日志级别以及优先级排序: OFF > FATAL > ERROR > WARN > INFO > DEBUG > TRACE > ALL -->
  <!--Configuration后面的status，这个用于设置log4j2自身内部的信息输出，可以不设置，当设置成trace时，你会看到log4j2内部各种详细输出-->
  <!--monitorInterval：Log4j能够自动检测修改配置 文件和重新配置本身，设置间隔秒数-->
  <configuration status="WARN" monitorInterval="30">
      <!--先定义所有的appender-->
      <appenders>
      <!--这个输出控制台的配置-->
          <console name="Console" target="SYSTEM_OUT">
          <!--输出日志的格式-->
              <PatternLayout pattern="[%d{HH:mm:ss:SSS}] [%p] - %l - %m%n"/>
          </console>
      <!--文件会打印出所有信息，这个log每次运行程序会自动清空，由append属性决定，这个也挺有用的，适合临时测试用-->
      <File name="log" fileName="log/test.log" append="false">
         <PatternLayout pattern="%d{HH:mm:ss.SSS} %-5level %class{36} %L %M - %msg%xEx%n"/>
      </File>
      <!-- 这个会打印出所有的info及以下级别的信息，每次大小超过size，则这size大小的日志会自动存入按年份-月份建立的文件夹下面并进行压缩，作为存档-->
          <RollingFile name="RollingFileInfo" fileName="${sys:user.home}/logs/info.log"
                       filePattern="${sys:user.home}/logs/?{date:yyyy-MM}/info-%d{yyyy-MM-dd}-%i.log">
              <!--控制台只输出level及以上级别的信息（onMatch），其他的直接拒绝（onMismatch）-->        
              <ThresholdFilter level="info" onMatch="ACCEPT" onMismatch="DENY"/>
              <PatternLayout pattern="[%d{HH:mm:ss:SSS}] [%p] - %l - %m%n"/>
              <Policies>
                  <TimeBasedTriggeringPolicy/>
                  <SizeBasedTriggeringPolicy size="100 MB"/>
              </Policies>
          </RollingFile>
          <RollingFile name="RollingFileWarn" fileName="${sys:user.home}/logs/warn.log"
                       filePattern="${sys:user.home}/logs/?{date:yyyy-MM}/warn-%d{yyyy-MM-dd}-%i.log">
              <ThresholdFilter level="warn" onMatch="ACCEPT" onMismatch="DENY"/>
              <PatternLayout pattern="[%d{HH:mm:ss:SSS}] [%p] - %l - %m%n"/>
              <Policies>
                  <TimeBasedTriggeringPolicy/>
                  <SizeBasedTriggeringPolicy size="100 MB"/>
              </Policies>
          <!-- DefaultRolloverStrategy属性如不设置，则默认为最多同一文件夹下7个文件，这里设置了20 -->
              <DefaultRolloverStrategy max="20"/>
          </RollingFile>
          <RollingFile name="RollingFileError" fileName="${sys:user.home}/logs/error.log"
                       filePattern="${sys:user.home}/logs/?{date:yyyy-MM}/error-%d{yyyy-MM-dd}-%i.log">
              <ThresholdFilter level="error" onMatch="ACCEPT" onMismatch="DENY"/>
              <PatternLayout pattern="[%d{HH:mm:ss:SSS}] [%p] - %l - %m%n"/>
              <Policies>
                  <TimeBasedTriggeringPolicy/>
                  <SizeBasedTriggeringPolicy size="100 MB"/>
              </Policies>
          </RollingFile>
      </appenders>
      <!--然后定义logger，只有定义了logger并引入的appender，appender才会生效-->
      <loggers>
          <!--过滤掉spring和mybatis的一些无用的DEBUG信息-->
          <logger name="org.springframework" level="INFO"/> 
          <logger name="org.mybatis" level="INFO"/>
          <root level="all">
              <appender-ref ref="Console"/>
              <appender-ref ref="RollingFileInfo"/>
              <appender-ref ref="RollingFileWarn"/>
              <appender-ref ref="RollingFileError"/>
          </root>
      </loggers>
  </configuration>
```

使用了`lombok`中的`@Slf4j` 注解调用 **`org.slf4j.Logger`** 对象

### 第11条：覆盖`equals`时总是要覆盖`hashCode`

如果不这样做将导致该类无法结合所有基于散列的集合一起正常工作，例如使用第10条中的`PhoneNumber`类的实例作为建：

```java
public static void main(String[] args) {
    Map<PhoneNumber, String> map = new HashMap<>();
    map.put(new PhoneNumber(707, 867, 5309), "Jenny");
    log.info(map.get(new PhoneNumber(707, 867, 5309)));
}
---------------------------------------------------------
[20:37:05:398] [INFO] - com.effectivejava.learn.method.PhoneNumber.main(PhoneNumber.java:47) - null
```

put与get中的key在equals时总是相等的，但是具有不同的散列码，所以结果为空；

一个简单的实现方法

```java
@Override
    public int hashCode() {
        int result = Short.hashCode(areaCode);
        result = 31 * result + Short.hashCode(prefix);
        result = 31 * result + Short.hashCode(lineNum);
        return result;
    }
```

也可以使用

```java
@Override
    public int hashCode() {
        return Objects.hash(areaCode, prefix, lineNum);
    }
```

但运行速度会更慢一些，因为这会引发数组的创建，以便传入数目可变的参数，如果参数中有基本类型，还需要装箱和拆箱；

### 第12条：始终覆盖`tostring`

好的tostring实现可以使类用起来更加舒适，使用这个类的系统也更易于调试

```java
@Override
    public String toString() {
        return "PhoneNumber{" +
                "areaCode=" + areaCode +
                ", prefix=" + prefix +
                ", lineNum=" + lineNum +
                '}';
    }
```

### 第13条：谨慎的覆盖clone

`Cloneable`接口的目的是作为对象的一个`minin`接口，表明这个对象是运行克隆的；但遗憾的是它并没有达到这个目的，因为它缺少一个clone方法，而Object的clone方法是受保护的；

当每一个属性都是一个基本类型的值，或者是一个指向不可变对象的引用，比如`PhoneNumber`类，则可以这样实现clone方法：

```java
@Override
    public PhoneNumber clone() {
        try {
            return (PhoneNumber) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
```

**注意：不可变的类永远不应该提供clone方法**

如果对象中属性包含可变对象的引用，则不能使用上述方法，因为修改原始数组将影响到克隆对象，我们的这样：

```java
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;
    
    public Stack() {
        this.elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }
    
    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }
    
    public Object pop() {
        if (size == 0){
            throw new EmptyStackException();
        }
        Object result = elements[--size];
        elements[size] = null;
        return result;
    }
    
    private void ensureCapacity() {
        if (elements.length == size) {
            elements = Arrays.copyOf(elements, 2*size +1);
        }
    }

    @Override
    protected Stack clone() {
        try {
            Stack result = (Stack) super.clone();
            result.elements = elements.clone();
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}
```

**注意：如果elements属性是final的，则上述方案将不能正常使用**

递归调用clone有时还不够，例如，你正在为一个散列表编写clone方法，如果使用上述方案，则克隆对象与原始对象引用的对象是一样的，从而引起不正确的行为，我们可以使用以下方式：

```java
public class HashTable implements Cloneable {
    private Entry[] buckets = ...;

    private static class Entry {
        final Object key;
        Object value;
        Entry next;

        Entry(Object key, Object value, Entry next) {
            this.key = key;
            this.value = value;
            this.next = next;
        }

        Entry deepCopy() {
            Entry result = new Entry(key, value, next);
            for (Entry p = result; p.next != null; p = p.next) {
                p.next = new Entry(p.next.key, p.next.value, p.next.next);
            }
            return result;
        }
    }

    @Override
    public HashTable clone() {
        try {
            HashTable result = (HashTable) super.clone();
            result.buckets = new Entry[buckets.length];
            for (int i = 0; i < buckets.length; i++) {
                if (buckets[i] != null) {
                    result.buckets[i] = buckets[i].deepCopy();
                }
            }
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}
```

鉴于 clone() 方法存在这么多限制:

**除了拷贝数组，其他任何情况都不应该去覆盖 clone() 方法，也不该去调用它**

**对象拷贝的更好的办法是提供一个拷贝构造器或者拷贝工厂：**

```java
public final class PolicyCopyConstructor {

    private String code;
    private int applicantAge;
    private Liability liability;
    private List<String> specialDescriptions;

    public PolicyCopyConstructor(PolicyCopyConstructor policy) {
        this.code = policy.code;
        this.applicantAge = policy.applicantAge;
        this.liability = policy.liability;
        this.specialDescriptions = policy.specialDescriptions;
    }
}
```

```java
public final class PolicyCopyFactory {

    private String code;
    private int applicantAge;
    private Liability liability;
    private List<String> specialDescriptions;

    public static PolicyCopyFactory newInstance(PolicyCopyFactory policy) {
        PolicyCopyFactory copyPolicy = new PolicyCopyFactory();
        copyPolicy.setCode(policy.getCode());
        copyPolicy.setApplicantAge(policy.getApplicantAge());
        copyPolicy.setLiability(policy.getLiability());
        copyPolicy.setSpecialDescriptions(policy.getSpecialDescriptions());
        return copyPolicy;
    }
}
```

- Copy constructor & Copy factory 的优势：

1. 不依赖于某一种带有风险的，语言之外的对象创建机制（clone 是 native 方法）。
2. 不会与 final 域的正常使用发生冲突（clone 架构与引用可变对象的 final 域的正常使用是不兼容的）。
3. 不会抛出受检异常。
4. 不需要类型转换。

### 第14条：考虑实现Comparable接口

如果你正在编写一个值类，它具有非常明显的内在排序关系，比如按字母顺序、按数值顺序或者按年代顺序，那么你就应该坚决考虑实现Comparable接口；

`CompareTo`方法也必须遵守：自反性、对称性与传递性；**因此与equals相同：无法在用新的值组件扩展可实例化的类时，同时保持`CompareTo`约定；**

```java
public class CaseInsensitiveString implements Comparable<CaseInsensitiveString> {
    private final String s;

    public CaseInsensitiveString(String s) {
        this.s = Objects.requireNonNull(s);
    }

    @Override
    public int compareTo(CaseInsensitiveString o) {
        return String.CASE_INSENSITIVE_ORDER.compare(s, o.s);
    }
}
```

如果一个类有多个关键属性：

```java
@Override
    public int compareTo(PhoneNumber o) {
        int result = Short.compare(areaCode, o.areaCode);
        if (result == 0) {
            result = Short.compare(prefix, o.prefix);
            if (result == 0) {
                result = Short.compare(lineNum, o.lineNum);
            }
        }
        return result;
    }
```

在java8中，Comparator接口配置了一组比较器构造方法，是的比较器的构造工作变得非常顺畅：

```java
private static final Comparator<PhoneNumber> COMPARATOR =
            Comparator.comparingInt((PhoneNumber pn) -> pn.areaCode)
                    .thenComparingInt(pn -> pn.prefix)
                    .thenComparingInt(pn -> pn.lineNum);

    @Override
    public int compareTo(PhoneNumber o) {
        return COMPARATOR.compare(this, o);
    }
```

