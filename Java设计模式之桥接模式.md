## 定义

将事物实现从各维度抽象出来，各维度独立变化，之后通过聚合或依赖的方式组合起来，减少各维度之间的相互耦合，从而更加适合变化。

## 适用于
当一种事物在多个维度都有比较灵活的变化时，如果为每个维度，每个变化都独立一个类的话，假设有N个维度，每个维度有M个变化，那么就会创建MN个类，造成类爆炸。使用桥接模式，将各个维度之间解耦合，不使用继承，使用依赖方式，解决类爆炸问题。
## 例子
### 说明
一辆汽车有多个维度，这里暂时只考虑发动机类型与座位类型，发动机类型有单缸类型、多缸类型；座位类型有五座，七座。
### 代码实现
#### 车的抽象类
```java
public abstract class AbstractCar {

  protected AbstractEngine engine;

  protected AbstractSit sit;

  public AbstractCar(AbstractEngine engine, AbstractSit sit) {
    this.engine = engine;
    this.sit = sit;
  }

  /**
   * 启动
   */
  public abstract void run();

  /**
   * 入座
   */
  public abstract void sit();
}
```
#### 车的实现类
```java
public class BMWCar extends AbstractCar {

  public BMWCar(AbstractEngine engine, AbstractSit sit) {
    super(engine, sit);
  }

  @Override
  public void run() {
    System.out.println("宝马车开始启动了。。。");

    sit();
    engine.start();
  }

  @Override
  public void sit() {
    System.out.println("落座宝马车了。。。");
    sit.sit();
  }
}
```
#### 发动机的抽象类
```java
public abstract class AbstractEngine {

  /**
   * 启动
   */
  public abstract void start();
}
```
#### 发动机的单缸和多缸实现类
```java
public class SingleJarEngine extends AbstractEngine {

  @Override
  public void start() {
    System.out.println("单缸引擎启动了。。。");
  }
}

public class MultiJarEngine extends AbstractEngine {

  @Override
  public void start() {
    System.out.println("多缸引擎启动了。。。");
  }
}

```
#### 座位的抽象类
```java
public abstract class AbstractSit {

  /**
   * 座位
   */
  public abstract void sit();
}
```

#### 座位的实现类
```java
public class FiveSit extends AbstractSit {

  @Override
  public void sit() {
    System.out.println("您乘坐的是五座的车。。。");
  }
}

public class SevenSit extends AbstractSit {

  @Override
  public void sit() {
    System.out.println("您乘坐的是七座的车。。。");

  }
}
```
#### 测试demo

```java
public class BridgeDesignTest {

  public static void main(String[] args) {

    //五座，单缸 宝马车
    BMWCar fiveSitAndSingleJarEngineCar = new BMWCar(new SingleJarEngine(), new FiveSit());
    fiveSitAndSingleJarEngineCar.run();
    //五座，多缸 宝马车
    BMWCar fiveSitAndMultiJarEngineCar = new BMWCar(new MultiJarEngine(), new FiveSit());
    fiveSitAndMultiJarEngineCar.run();
    //七座，单缸 宝马车
    BMWCar sevenSitAndSingleJarEngineCar = new BMWCar(new SingleJarEngine(), new SevenSit());
    sevenSitAndSingleJarEngineCar.run();
    //七座，多缸 宝马车
    BMWCar sevenSitAndMultiJarEngineCar = new BMWCar(new MultiJarEngine(), new SevenSit());
    sevenSitAndMultiJarEngineCar.run();

  }
}
```

### 装饰模式与代理模式的区别

1. 装饰模式是为了增强原有对象的功能，代理模式是为了控制客户端访问被代理对象的权限
2. 装饰模式的对象是在运行时动态指定的，代理模式（静态代理）一般是在编译时期就知道代理对象。
3. 装饰对象是客户端（调用方）手动创建并传给提供方的，而代理对象是提供方创建的，调用方一般不知道被代理对象的存在，只知道代理对象。