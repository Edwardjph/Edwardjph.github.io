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

同时这种方法还可能导致无限递归问题：假设Point有两个子类，如`ColorPoint`和`SmellPoint`都用这种方式，则`ColorPoint.equals(SmellPoint)` 的调用将抛出`StackOverflowError`；

这是一个在面向对象语言中关于等价关系的一个基本问题：

**我们没法在扩展可实例化的类的同时，既增加新的值组件，同时又保留equals约定**

