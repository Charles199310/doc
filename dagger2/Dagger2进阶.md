# Dagger2入门（二）—— Dagger2的高级用法
## 目录
[Dagger2入门（一）—— Dagger2优点](https://www.jianshu.com/p/fa44a88cf27c)  
[Dagger2入门（二）—— Dagger2的简单使用](https://www.jianshu.com/p/46d29e0f0373)  
[Dagger2入门（二）—— Dagger2的高级用法](https://www.jianshu.com/p/146ce3894436)  

## @Qualifier & @Named
> `@Qualifier`: 标记在inject和provides上用于匹配对应关系  
> `@Name`: 一种框架默认实现的`Qualifier`,通过字符串匹配  

当一个Modules中有相同类型返回值的provides时，dagger2会报编译时错误，这时我们需要用Qualifier区分两个provides分别应该注入到什么地方。

## @Scope & @Singleton
> `@Scope`: 标记在Componet和provides上，用于管理生命周期，表示在Componet生命周期里只有一个provides提供的实例
> `@Singleton`: 一种由框架默认实现的`Scope`

一般我们会自定义不同的Scope来表示不同的生命周期类型，比如Activity级别，Application级别

## Component#dependencies & @Subcomponent
> `Component#dependencies`: 静态管理Component的依赖，Component可以将dependencies指定的依赖注入到自己当中。
> `@Subcomponent`: Component的一种，可以动态的管理依赖依赖，当Componet提供Subcomponent的接口时，Subcomponent和Component就建立了依赖关系，Subcomponent依赖于Componenet;

一般我们用Subcomponent管理有生命周期关联的依赖。

## Lazy\<T>
> Lazy\<T>：泛型，Lazy依赖不会马上被注入，而是需要等到get的时候才会被注入
