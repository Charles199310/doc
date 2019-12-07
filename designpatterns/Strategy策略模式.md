# Strategy 策略模式
[设计原则学习笔记](https://www.jianshu.com/p/f7f79adad32b)  
[设计模式学习笔记](https://www.jianshu.com/p/08bf9381697c)  
## 作用
将算法封装起来，使对象可以在不同情况，使用不同算法。
## 类图
![策略模式类图](res/strategy_01.PNG)
## Java实现
```Java
//定义策略接口，并在子类中实现算法
public interface Strategy {
    void algorithmInterface();
}
public class StrategyA implements Strategy {
    @Override
    public void algorithmInterface() {
        //todo do something A
    }
}
public class StrategyB implements Strategy {
    @Override
    public void algorithmInterface() {
        //todo do something B
    }
}
//定义context，在可以使用不同的算法执行操作
public class Context {
    private Strategy strategy;
    public Context(Strategy strategy) {
        this.strategy = strategy;
    }
    public void contextInterface() {
        strategy.algorithmInterface();
    }
}
//定义客户端
public class Client {
    public static void main(String[] args) {
        Context context = new Context(new StrategyA());
        context.contextInterface();
    }
}
```
策略模式和状态模式在代码上几乎和策略模式一模一样。其区别在于状态一般是会切换的。策略模式则不一定会切换。策略模式倾向于算法上的不同，运算结果可能不会有太大差别，状态则可能差别比较大。

## Android源码中的应用
* 动画插值器，估值器
