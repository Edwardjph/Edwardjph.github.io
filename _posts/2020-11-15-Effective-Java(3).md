## 第6章 枚举和注解

### 第34条：使用enum替代int

在将枚举类型添加到该语言之前，表示枚举类型的常见模式是声明一组名为 `int` 的常量，每个类型的成员都有一个常量：

```java
// The int enum pattern - severely deficient!
public static final int APPLE_FUJI         = 0;
public static final int APPLE_PIPPIN       = 1;
public static final int APPLE_GRANNY_SMITH = 2;

public static final int ORANGE_NAVEL  = 0;
public static final int ORANGE_TEMPLE = 1;
public static final int ORANGE_BLOOD  = 2;
```

这种被称为 `int` 枚举模式的技术有许多缺点：**它没有提供类型安全的方式，也没有提供任何表达力**

使用 `int` 枚举的程序很脆弱。 因为 `int` 枚举是编译时常量，所以它们的 `int` 值被编译到使用它们的客户端中。 如果与 `int` 枚举关联的值发生更改，则必须重新编译其客户端；

Java 提供了一种避免 `int` 和 `String` 枚举模式的所有缺点的替代方法，并提供了许多额外的好处：

```java
public enum Apple  { FUJI, PIPPIN, GRANNY_SMITH }
public enum Orange { NAVEL, TEMPLE, BLOOD }
```

Java 枚举类型背后的基本思想很简单：**它们是通过公共静态 `final` 属性为每个枚举常量导出一个实例的类。**

由于没有可访问的构造方法，枚举类型实际上是 `final` 的。 由于客户既不能创建枚举类型的实例也不能继承它，除了声明的枚举常量外，不能有任何实例。 换句话说，枚举类型是实例控制的；

枚举提供了编译时类型的安全性。如果声明一个参数为 `Apple` 类型，则可以保证传递给该参数的任何非空对象引用是三个有效 `Apple` 值中的一个；

具有相同名称常量的枚举类型可以和平共存，因为每种类型都有其自己的名称空间。 可以在枚举类型中添加或重新排序常量，而无需重新编译其客户端，因为导出常量的属性在枚举类型与其客户端之间提供了一层隔离：常量值不会编译到客户端，因为它们位于 `int` 枚举模式中。 最后，可以通过调用其 `toString` 方法将枚举转换为可打印的字符串；

**枚举类型还允许添加任意方法和属性并实现任意接口。**它们提供了所有 `Object` 方法的高质量实现（第 3 章），它们实现了 `Comparable`和 `Serializable`，并针对枚举类型的可任意改变性设计了序列化方式；

```java
public enum Planet {
    MERCURY(3.302e+23, 2.439e6),
    VENUS(4.869e+24, 6.052e6),
    EARTH(5.975e+24, 6.378e6),
    MARS(6.419e+23, 3.393e6),
    JUPITER(1.899e+27, 7.149e7),
    SATURN(5.685e+26, 6.027e7),
    URANUS(8.683e+25, 2.556e7),
    NEPTUNE(1.024e+26, 2.477e7);
    
    private final double mass;

    private final double radius;
    /**
     * In m / s^2
     */
    private final double surfaceGravity;
    /**
     * Universal gravitational constant in m^3 / kg s^2
     */
    private static final double G = 6.67300E-11;

    /**
     * @param mass 质量
     * @param radius 半径
     */
    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
        surfaceGravity = G * mass / (radius * radius);
    }

    public double mass() {
        return mass;
    }

    public double radius() {
        return radius;
    }

    public double surfaceGravity() {
        return surfaceGravity;
    }

    public double surfaceWeight(double mass) {
        return mass * surfaceGravity;
    }
    public static void main(String[] args) {
        double earthWeight = Double.parseDouble("185");
        double mass = earthWeight / Planet.EARTH.surfaceGravity();
        for (Planet p : Planet.values()) {
            System.out.printf("Weight on %s is %f%n", p, p.surfaceWeight(mass));
        }
    }
}
===================================================================
Weight on MERCURY is 69.912739
Weight on VENUS is 167.434436
Weight on EARTH is 185.000000
Weight on MARS is 70.226739
Weight on JUPITER is 467.990696
Weight on SATURN is 197.120111
Weight on URANUS is 167.398264
Weight on NEPTUNE is 210.208751
```

编写一个丰富的枚举类型:

1. **要将数据与枚举常量相关联，请声明实例属性并编写一个构造方法，构造方法带有数据并将数据保存在属性中;**
2. 枚举本质上是不变的，所以所有的属性都应该是 `final` 的。 属性可以是公开的，但最好将它们设置为私有并提供公共访问方法；

当你想编写一个枚举类型，来表示计算器的四个基本操作：

```java
public enum Operation {
    PLUS, MINUS, TIMES, DIVIDE;

    public double apply(double x, double y) {
        switch (this) {
            case PLUS:
                return x + y;
            case MINUS:
                return x - y;
            case TIMES:
                return x * y;
            case DIVIDE:
                return x / y;
        }
        throw new AssertionError("Unknown op: " + this);
    }
}
```

此代码有效，但不是很漂亮。 如果没有 `throw` 语句，就不能编译，更糟的是，代码很脆弱。 如果添加新的枚举常量，但忘记向 `switch` 语句添加相应的条件，枚举仍然会编译，但在尝试应用新操作时，它将在运行时失败；

有一种更好的方法可以将不同的行为与每个枚举常量关联起来：**在枚举类型中声明一个抽象的 apply 方法，并用常量特定的类主体中的每个常量的具体方法重写它。 这种方法被称为特定于常量（constant-specific）的方法实现**：

```java
public enum Operation {
    PLUS {
        @Override
        public double apply(double x, double y) {
            return x + y;
        }
    },
    MINUS {
        @Override
        public double apply(double x, double y) {
            return x - y;
        }
    },
    TIMES {
        @Override
        public double apply(double x, double y) {
            return x * y;
        }
    },
    DIVIDE {
        @Override
        public double apply(double x, double y) {
            return x / y;
        }
    };

    public abstract double apply(double x, double y);
}
```

如果向第二个版本的操作添加新的常量，则不太可能会忘记提供 `apply` 方法，因为该方法紧跟在每个常量声明之后。 万一忘记了，编译器会提醒你，因为枚举类型中的抽象方法必须被所有常量中的具体方法重写

**特定于常量的方法实现可以与特定于常量的数据结合使用。** 例如，以下是 `Operation` 的一个版本，它重写 `toString` 方法以返回通常与该操作关联的符号：

```java
public enum Operation {
    PLUS("+") {
        @Override
        public double apply(double x, double y) {
            return x + y;
        }
    },
    MINUS("-") {
        @Override
        public double apply(double x, double y) {
            return x - y;
        }
    },
    TIMES("*") {
        @Override
        public double apply(double x, double y) {
            return x * y;
        }
    },
    DIVIDE("/") {
        @Override
        public double apply(double x, double y) {
            return x / y;
        }
    };

    public abstract double apply(double x, double y);

    private final String symbol;

    Operation(String symbol) {
        this.symbol = symbol;
    }

    @Override
    public String toString() {
        return symbol;
    }

    public static void main(String[] args) {
        double x = Double.parseDouble("2");
        double y = Double.parseDouble("1");
        for (Operation op : Operation.values()) {
            System.out.printf("%f %s %f = %f%n",
                    x, op, y, op.apply(x, y));
        }
    }
}
=====================================================================
2.000000 + 4.000000 = 6.000000
2.000000 - 4.000000 = -2.000000
2.000000 * 4.000000 = 8.000000
2.000000 / 4.000000 = 0.500000
```

枚举类型具有自动生成的 `valueOf(String)` 方法，该方法将常量名称转换为常量本身。 如**果在枚举类型中重写 `toString` 方法，请考虑编写 `fromString` 方法将自定义字符串表示法转换回相应的枚举类型**:

```java
private static final Map<String, Operation> stringToEnum = Stream.of(values()).collect(toMap(Object::toString, e -> e));
public static Optional<Operation> fromString(String symbol) {
    return Optional.ofNullable(stringToEnum.get(symbol));
}
```

**特定于常量的方法实现的一个缺点是它们使得难以在枚举常量之间共享代码。**例如，考虑一个代表工资包中的工作天数的枚举。 该枚举有一个方法，根据工人的基本工资（每小时）和当天工作的分钟数计算当天工人的工资。 在五个工作日内，任何超过正常工作时间的工作都会产生加班费; 在两个周末的日子里，所有工作都会产生加班费。

```java
public enum PayrollDay {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY,
    SATURDAY, SUNDAY;

    private static final int MINS_PER_SHIFT = 8 * 60;

    int pay(int minutesWorked, int payRate) {
        int basePay = minutesWorked * payRate;

        int overtimePay;
        switch (this) {
            case SATURDAY:
            case SUNDAY:
                overtimePay = basePay / 2;
                break;
            default:
                overtimePay = minutesWorked <= MINS_PER_SHIFT ?
                        0 : (minutesWorked - MINS_PER_SHIFT) * payRate / 2;
        }
        return basePay + overtimePay;
    }
}
```

这段代码无可否认是简洁的，但从维护的角度来看是危险的。 假设你给枚举添加了一个元素，可能是一个特殊的值来表示一个假期，但忘记在 `switch` 语句中添加一个相应的 `case` 条件。 该程序仍然会编译，但付费方法会默默地为工作日支付相同数量的休假日，与普通工作日相同

幸运的是，有一个很好的方法来实现这一点。 这个想法是将加班费计算移入私有嵌套枚举中，并将此**策略枚举**的实例传递给 `PayrollDay` 枚举的构造方法

```java
public enum PayrollDay {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY,
    SATURDAY(PayType.WEEKEND), SUNDAY(PayType.WEEKEND);

    private final PayType payType;

    PayrollDay(PayType payType) {
        this.payType = payType;
    }

    PayrollDay() {
        this(PayType.WEEKDAY);
    }

    int pay(int minutesWorked, int payRate) {
        return payType.pay(minutesWorked, payRate);
    }

    // The strategy enum type
    private enum PayType {
        WEEKDAY {
            @Override
            int overtimePay(int minsWorked, int payRate) {
                return minsWorked <= MINS_PER_SHIFT ? 0 :
                        (minsWorked - MINS_PER_SHIFT) * payRate / 2;
            }
        },
        WEEKEND {
            @Override
            int overtimePay(int minsWorked, int payRate) {
                return minsWorked * payRate / 2;
            }
        };

        abstract int overtimePay(int minsWorked, int payRate);

        private static final int MINS_PER_SHIFT = 8 * 60;

        int pay(int minsWorked, int payRate) {
            int basePay = minsWorked * payRate;
            return basePay + overtimePay(minsWorked, payRate);
        }
    }
}
```

每当需要一组固定的常量，并且在编译的时候就知道其成员的时候，就应该使用枚举；

### 第35条：使用实例域替代序数

所有枚举都有一个 `ordinal` 方法，它返回每个枚举常量类型的数值位置。

```java
public enum Ensemble {
    SOLO,   DUET,   TRIO, QUARTET, QUINTET,
    SEXTET, SEPTET, OCTET, NONET,  DECTET;
    public int numberOfMusicians() { return ordinal() + 1; }
}
```

**永远不要从枚举的序号中得出与它相关的值; 请将其保存在实例属性中**

```java
public enum Ensemble {
    SOLO(1), DUET(2), TRIO(3), QUARTET(4), QUINTET(5),
    SEXTET(6), SEPTET(7), OCTET(8), DOUBLE_QUARTET(8),
    NONET(9), DECTET(10), TRIPLE_QUARTET(12);
    private final int numberOfMusicians;
    Ensemble(int size) { this.numberOfMusicians = size; }
    public int numberOfMusicians() { return numberOfMusicians; }
}
```

