# 编写自定义Android Gradle Plugin  
## 什么是Gradle
Gradle是我们在Android工程中的 __自动构建工具__ 。__自动化构建__ 就是将我们的代码打包成jar、aar、apk的过程。  
我们知道，我们编写的Java、xml并不能直接运行在ART、Dalvik或者JVM上。需要将其编译为dex或jar、class，如可可能的话还要直接运行。Gradle将这些操作打包成一个命令。
在Java历史上有过Ant、Maven。发展到现在Gradle被认为集合了Ant和Maven的灵活和易用等优点，在这里不做论述。

## 自定义Gradle插件能做什么
利用Gradle，我们可以：  
1. 自动生成代码（Android R.java 以及GreedDao，Dagger等众多开元库就是这么做的）  
2. 代码或文件检查（lint就是这么做的）和抛出自定义编译时错误（有时候不方便用lint等工具检查）
3. 在打包的过程中做一些我们希望自动执行的事情，比如讲打包好的文件发到群里@相关人员等  

## Gradle插件如何编写
Gradle插件有三种编写，并应用的方法：  
1. 编写Gradle文件，在应用模块上用`apply from: ...`应用该文件。
2. 在project目录下，新建名为buildSrc的groovy模块，用`apply plugin: ...`应用该模块。
3. 新建groovy模块，打包成jar包，发布，用`classpath ...`引入jar包，用`apply plugin: ...`应用该jar包
