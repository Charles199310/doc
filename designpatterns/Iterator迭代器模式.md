# Interator 迭代器模式
[设计原则学习笔记](https://www.jianshu.com/p/f7f79adad32b)  
[设计模式学习笔记](https://www.jianshu.com/p/08bf9381697c)  
## 作用
针对数据集合，提供一种（若干）忽略集合内部细节以及访问细节的访问方式，从而使访问者与集合解耦。
## 类图
![迭代器模式类图](res/iterator_01.PNG)  
## Java实现
```Java
//定义数据结构
public class Item {
}
//定义迭代器接口
public interface Iterator {
    void moveToFirst();
    void next();
    boolean hasNext();
    Item getItem();
}
//定义创建集合以及迭代器的接口
public interface Aggregate {
    Iterator createIterator();
}
public class ConcreteAggregate implements Aggregate {
    private Item[] items = new Item[]{new Item(), new Item()};
    @Override
    public Iterator createIterator() {
        return new ConcreteIterator(items);
    }
}
//实现具体的迭代器
public class ConcreteIterator implements Iterator {
    private Item[] items;
    private int index;

    ConcreteIterator(Item[] items) {
        this.items = items;
    }

    @Override
    public void moveToFirst() {
        index = 0;
    }

    @Override
    public void next() {
        index++;
    }

    @Override
    public boolean hasNext() {
        return items != null && index > 0 && index < items.length;
    }

    @Override
    public Item getItem() {
        return items[index];
    }
}
// 客户端通过迭代器访问Item
public class Client {
    public static void main(String[] args) {
        Aggregate aggregate = new ConcreteAggregate();
        Iterator iterator = aggregate.createIterator();
        iterator.moveToFirst();
        do {
            Item item = iterator.getItem();
            if (!iterator.hasNext()) {
                break;
            }
            iterator.next();
        } while (true);
    }
}
```
