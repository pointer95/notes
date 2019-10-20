## 创建和销毁对象

### 考虑用静态工厂方法代替构造器

```java
public static Boolean valueOf(boolean b) {
    return b ? Boolean.TRUE : Boolean.FALSE;
}
```

优势：

- 有名称
- 不必在每次调用它们时都创建一个新对象
- 可以返回原返回类型的任何子类型的对象
- 在创建参数化类型实例的时候，静态工厂方法使代码变得更简单

缺点：

- 类如果不含 public 或者 protected 修饰的构造器，就不能被子类化
- 与其他的静态方法实际上没有区别

### 遇到多个构造器参数时要考虑用构建器

与构造器相比，builder 模式的微略优势在于，builder 可以有多个可变的参数；不足在于，为了创建对象，必须先创建它的构建器。

### 用私有构造器或者枚举类型强化 Singleton 属性

```java
// Singleton with public final field
public Class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
    
    public void leaveTheBuilding() { ... }
}

// Singleton with static factory
public Class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
    public static Elvis getInstance() { return INSTANCE; }
    
    public void leaveTheBuilding() { ... }
}

// ENUM singleton - the preferred approach
public enum Elvis {
    INSTANCE;
    
    public void leaveTheBuilding() { ... }
}
```

### 通过私有构造器强化不可实例化的能力

### 避免创建不必要的对象

- 重用不可变对象
- 重用那些已知不会被修改的可变对象
- 适配器模式
- 优先使用基本类型而不是包装类型，避免自动装箱

### 消除过期的对象引用

例如一个栈先 push ，然后 pop，从栈中弹出来的对象将不会被当作垃圾回收，最终导致内存泄漏。这是因为栈内部维护着对这些对象的过期引用（obsolete reference）。因此需要将弹出的元素置为 null。

内存泄漏的原因：

- 一般而言，只要是类自己管理内存，就应该警惕内存泄漏问题。
- 缓存
- 监听器和其他回调

### 避免使用终结方法