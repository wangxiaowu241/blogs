定义

像现有的一个对象添加新的功能，同时又不改变其结构，它是作为现有的一个类的包装。

装饰模式创建了一个装饰类，包装了原有的类，而又不改变其内部结构，同时增加新的功能。

## 适用于

般的，我们为了扩展一个类经常使用继承方式实现，由于继承为类引入静态特征，并且随着扩展功能的增多，子类会很膨胀。在不想增加很多子类的情况下扩展类。

## 好处

**将类的核心职责和装饰功能区分开，而且去除相关类中重复的装饰逻辑。用户可以有选择地、按顺序地使用装饰功能包装对象。**

## 例子

### 接口及具体实现

```java
public interface Subject {

  void show();
}

public class ConcreteSubject implements Subject {

  @Override
  public void show() {
    System.out.println("具体执行者运行。。。");
  }
}
```

### 装饰者

#### 装饰者抽象类

```java
public abstract class AbstractDecorator implements Subject {

  protected Subject subject;

  public AbstractDecorator(Subject subject) {
    this.subject = subject;
  }

  @Override
  public void show() {
    subject.show();
  }
}

```

#### 增强属性装饰类A

```java
public class SubjectDecoratorA extends AbstractDecorator {

  private String addState = "增强属性A";

  public SubjectDecoratorA(Subject subject) {
    super(subject);
  }

  @Override
  public void show() {
    super.show();
    System.out.println("具体装饰类A的增强属性为" + addState);
  }

}
```

#### 增强方法装饰类B

```java
public class SubjectDecoratorB extends AbstractDecorator {

  public SubjectDecoratorB(Subject subject) {
    super(subject);
  }

  @Override
  public void show() {
    super.show();
    addBehavior();
  }

  private void addBehavior() {
    System.out.println("具体装饰类B的增强方法运行。。");
  }

}
```

#### 测试类

```java
public class DecoratorDesignTest {

  public static void main(String[] args) {

    ConcreteSubject concreteSubject = new ConcreteSubject();
    new SubjectDecoratorA(concreteSubject).show();
    new SubjectDecoratorB(concreteSubject).show();
  }
}
```

例子：

咖啡是一种饮品，咖啡种类有拿铁、摩卡、卡布奇诺等。 咖啡可以加调料，糖、牛奶等，假设加调料收费。

那么如何解决咖啡加调料的问题。 可以每种组合方式都建一个类，但是这种方式解决不了加多个相同调料的问题，比如加两包糖

使用装饰模式来解决这个问题。 咖啡的核心抽取出来：拿铁、摩卡、卡布奇诺；装饰点：糖、牛奶。

```java
/**
 * 饮品
 *
 */
public abstract class Drink {

  private String description = "";

  private BigDecimal price = BigDecimal.ZERO;

  public String getDescription() {
    return description;
  }

  public void setDescription(String description) {
    this.description += description;
  }

  public BigDecimal getPrice() {
    return price;
  }

  public void setPrice(BigDecimal price) {
    this.price = price;
  }

  public abstract BigDecimal cost();
}

/**
 * 咖啡基类
 */
public class Coffee extends Drink {

  @Override
  public BigDecimal cost() {

    //直接返回价格
    return super.getPrice();
  }
}

/**
 * 拿铁咖啡
 *
 */
public class LatteCoffee extends Coffee {

  public LatteCoffee() {
    super.setDescription("拿铁咖啡");
    super.setPrice(BigDecimal.ZERO);
  }
}

/**
 * 摩卡咖啡
 *
 */
public class MochaCoffee extends Coffee {

  public MochaCoffee() {
    super.setDescription("摩卡咖啡");
    super.setPrice(BigDecimal.ZERO);
  }
}

/**
 * 卡布奇诺咖啡
 *
 */
public class CappuccinoCoffee extends Coffee {

  public CappuccinoCoffee() {
    super.setDescription("卡布奇诺咖啡");
    super.setPrice(BigDecimal.ZERO);
  }
}

/**
 * 装饰类基类
 */
public abstract class Decorator extends Drink {

  private Drink drink;

  public Decorator(Drink drink) {
    this.drink = drink;
  }

  @Override
  public BigDecimal cost() {
    return super.getPrice().add(drink.cost());
  }
}

/**
 * 调料-牛奶
 *
 */
public class Milk extends Decorator {

  public Milk(Drink drink) {
    super(drink);
    setPrice(BigDecimal.ONE);
    setDescription("牛奶");
  }
}

/**
 * 调料-糖
 */
public class Sugar extends Decorator {

  public Sugar(Drink drink) {
    super(drink);
    setPrice(BigDecimal.ONE);
    setDescription("糖");
  }
}

/**
 * 测试类
 */
public class DecoratorDesignTest {

  public static void main(String[] args) {

    //加两份糖，一份牛奶的摩卡
    Drink firstDrink = new Sugar(new Milk(new Milk(new LatteCoffee())));
    System.out.println(firstDrink.getDescription() + "-" + firstDrink.cost());

    //一份糖，一份牛奶的拿铁
    Drink secondDrink = new Sugar(new Milk(new LatteCoffee()));
    System.out.println(secondDrink.getDescription() + "-" + secondDrink.cost());
  }
}
```

