# Chan of Responsibility 责任链模式
[设计原则学习笔记](https://www.jianshu.com/p/f7f79adad32b)  
[设计模式学习笔记](https://www.jianshu.com/p/08bf9381697c)  
## 作用
将请求以其职责拆成，并以链的结构组合，使请求在链中传递并且可以在链中拦截请求。
## 类图
![责任链模式类图](res/chain_of_responsibility_01.PNG)
## Java实现
```Java
//定义handler
public abstract class Handler {
    protected Handler handler;
    public abstract void handleRequest();
}
public class HandlerA extends Handler {
    public HandlerA() {
        handler = new HandlerB();
    }

    @Override
    public void handleRequest() {
        handler.handleRequest();
    }
}
public class HandlerB extends Handler{
    @Override
    public void handleRequest() {

    }
}
//客户端调用责任链
public class Client {
    public static void main(String[] args) {
        Handler handler = new HandlerA();
        handler.handleRequest();
    }
}
```
## Android源码中的应用
* View的点击事件分发
* Okhttp
