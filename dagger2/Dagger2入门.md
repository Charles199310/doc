# Dagger2入门（二）—— Dagger2的简单使用
## 环境搭建
Dagger2环境搭建新的Gradle版本上搭建要比原先简单的多。只需要在module目录下的build.gradle文件中添加依赖
``` Groovy
dependencies {
    ...

    implementation 'com.google.dagger:dagger-android:2.X'
    implementation 'com.google.dagger:dagger-android-support:2.X'
    annotationProcessor 'com.google.dagger:dagger-compiler:2.X'
}
```
即可。  
注意，有些教程上教的是一些比较旧的方法:
在project目录下的build.gradle文件中添加
``` Groovy
  dependencies {
      ...
      classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
  }
```
在module目录下的build.gradle文件中添加
``` Groovy
apply plugin: 'com.android.application'
apply plugin: 'com.neenbedankt.android-apt'
dependencies {
    //引入Dagger2以及依赖的一些包，因为用到注释所以还需要引用annotation包
    apt 'com.google.dagger:dagger-compiler:2.0'
    compile 'com.google.dagger:dagger:2.0'
    compile 'javax.inject:javax.inject:1'
    compile 'javax.annotation:javax.annotation-api:1.2'
}
```
在新的Gradle环境下会报出编译时错误：
``` ERROR
ERROR: android-apt plugin is incompatible with the Android Gradle plugin.  Please use 'annotationProcessor' configuration instead.
Affected Modules: app
```
大意是说android-apt以及被包含在Android的工程里了，需要用新的使用方法。
## 代码实现
1. 实现需要被注入的类（Demo）和被注入的类(MainActivity)
``` Java
public class Demo {
    public void work() {
        Log.i("schLog", "Dagger2 正常工作");
    }
}
public class MainActivity extends AppCompatActivity {
    @Inject
    Demo demo;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        demo.work();
    }
}
```
2. 实现Module和Component
``` Java
@Module
public class MainActivityModule {
    @Provides
    public Demo provideWatch() {
        return new Demo();
    }
}
@Component(modules = MainActivityModule.class)
public interface MainActivityComponent {
    void inject(Activity activity);
}
```
3. 按一下Ctrl和F9键（Make Project）或者点击Rebuild Project生成Dagger2工厂文件，然后将依赖注入到需要注入的地方。
``` Java
public class MainActivity extends AppCompatActivity {
    @Inject
    Demo demo;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        //将依赖注入
        DaggerMainActivityComponent.create().inject(this);

        demo.work();
    }
}
```
然后我们就可以看到我们的代码可以正常工作了。
## Dagger2中的概念
> `@Inject`: 作用于字段，构造方法，方法。  
作用于字端是表示被注入的字段。  
作用于方法时，在注入时这个方法会被执行。  
作用于构造方法则表示通过这个方法被注入（不需要provide）
> `@Module`：作用于类，表示该类用于管理依赖
> `@Component`: 作用于接口，生成一个类用于将依赖注入到需要的地方
> `@Provide`: 作用于方法，在module类中，表示生成依赖的地方。
