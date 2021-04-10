## 备注
**原发表于2016.05.08，资料已过时，仅作备份，谨慎参考**

## 前言
自上星期写 Retrofit 写吐之后
...
我问大队长能不能换个其他什么东西写，大队长就说了个单词 ButterKnife，这个我知道，是黄油刀的意思，然后看到是减轻工作量的框架我就开心了，还在为 findViewById() 烦恼吗？

## ButterKnife 概要
### 简介
ButterKnife 是一个 Android 系统的 View 注入框架，能够通过『注解』的方式来绑定 View 的属性或方法。

比如使用它能够减少 findViewById() 的书写，使代码更为简洁明了，同时不消耗额外的性能。

当然这样也有个缺点，就是可读性会差一些，好在 ButterKnife 比较简单，学习难度也不大。

### 添加依赖
这里以 Android Studio Gradle 为例，为项目添加 ButterKnife，注意两个步骤都要完成：

**1. Project 的 build.gradle 添加：**

    
    dependencies {
      classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
    }

**2. App 的 build.gradle 添加：**

    apply plugin: 'com.neenbedankt.android-apt'

    dependencies {
      compile 'com.jakewharton:butterknife:8.0.1'
      apt 'com.jakewharton:butterknife-compiler:8.0.1'
    }

这里顺便提一个名为 Android Butterknife Zelezny 的插件，可在 Settings - plugins 中下载，提供一键生成注解的功能，这里不多赘述，可以看以下文章：

[ButterKnife 注解框架的偷懒插件](http://www.tuicool.com/articles/Q3mmay/)

## ButterKnife 的使用
ButterKnife 的使用是比较简单的，具体的使用可以参考官方文档或者搜索其他博文资料，在这里只简单介绍一下它的用途：

1. 通过使用 @BindView 来消除 findViewById
2. 将多个 View 组织到一个列表中，一次性操作它们
3. 通过使用 @onClick 为 View 绑定监听，消除 listener 的匿名内部类
4. 通过使用资源注解如 @BindColor，来消除资源的查找

我们看到呢，它好像消除了一堆东西，其实就是简化的意思啦。

### 举个栗子
    class ExampleActivity extends Activity {
      // 声明注解
      @BindView(R.id.title) TextView title;
      @BindView(R.id.subtitle) TextView subtitle;
      @BindView(R.id.footer) TextView footer;
    
      @Override public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.simple_activity);
        // 进行绑定
        ButterKnife.bind(this);
        // TODO Use fields...
      }
    }

先仔细看前面三个 @BindView，机智的同学一眼就能明白了。

其次，是在 onCreate 方法中的，ButterKnife.bind(this)，它其实相当于以下代码：

    public void bind(ExampleActivity activity) {
      activity.subtitle = (android.widget.TextView) activity.findViewById(2130968578);
      activity.footer = (android.widget.TextView) activity.findViewById(2130968579);
      activity.title = (android.widget.TextView) activity.findViewById(2130968577);
    }

可以看到 .bind 方法的内容，其实就是我们平时写的 findViewById，它只是自己帮我们完成了这些代码的输入。

所以开头说 ButterKnife 并不消耗额外的性能，因为这跟我们自己写是一样的嘛。

### 使用方式整合
总的来说，ButterKnife 有以下使用：

1. View 绑定
2. Resource 绑定
3. 非 Activity 绑定
4. View List 绑定
5. Listener 绑定

具体的使用，参照文档吧！

[Butter Knife](https://jakewharton.github.io/butterknife/)

## 原理简单分析
### Java Annotation Processing
注解处理器，在 Java5 中叫 APT，有没有很眼熟：

    dependencies {
      classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
    }
    
    // 引入 ButterKnifeProcessor 等 
    apt 'com.jakewharton:butterknife-compiler:8.0.1'

这是一个用于编译时扫描和解析 Java 注解的工具，通过它我们可以自己定义注解，并定义解析器来处理它们。它的原理是读入 Java 源代码，解析注解，然后生成新的 Java 代码，新生成的代码最后被编译成 Java 字节码。

当然这里我们不是要深究 APT 的原理，而是要知道在 ButterKnife 中运用到了这个工具。

### ButterKnife 流程
这里以一个栗子来一步步观察，ButterKnife 的工作流程。

在此之前看一眼 @BindView 注解的定义：

    @Retention(CLASS) @Target(FIELD)
    public @interface BindView {
      /** View ID to which the field will be bound. */
      @IdRes int value();
    }

重点在于 @Retention(CLASS)，它表示该注解在编译时被保留，但在运行时 JVM 会忽略它。因此使用 ButterKnife 的注解，不会对运行时的性能造成消耗。

#### 扫描 ButterKnife 注解
首先我们使用注解来声明我们的一个 View：

    @BindView(R.id.text1) TextView text1;

于是在我们编译的时候，ButterKnifeProcessor 类的 process() 方法便会执行，搜索到所有的 ButterKnife 注解（@BindView），然后生成一个 Java 类。

#### 根据注解，生成 Java 类
我们在 app\build\generated\source\apt\ 中找到生成的 MainActivity$$ViewBinder 文件，该类中包含了一个 bind 方法：

    public Unbinder bind(final Finder finder, final T target, Object source) {
        InnerUnbinder unbinder = createUnbinder(target);
        View view;
        view = finder.findRequiredView(source, 2131492944, "field 'text1'");
        target.text1 = finder.castView(view, 2131492944, "field 'text1'");
        return unbinder;
      }

到这里就越来越像我们手写 findViewById() 了。

#### 动态注入
最后当我们执行 ButterKnife.bind(this) 时，ButterKnife 会加载上面生成的类，然后调用其 bind 方法。

- 这里首先调用了 findRequiredView 去寻找 R.id.text1 所对应的控件，其实相当于我们的 findViewById()
- 其次调用 castView，相当于类型转换，把找到的 View 转化为 TextView 类型

至此，我们就完成了一次 ButterKnife 的工作流程。

## 总结
实际上我们发现，ButterKnife 的原理还是相对简单的，因为其主要难点在于 APT 的使用，那属于其他方面的内容，在此我们并不深究。

在这个框架中我们看到了『反射』和『Annotation Processing』间的取舍，ButterKnife 使用了后者，避免了额外性能的消耗。

## 参考资料
[最新ButterKnife框架原理](http://www.trinea.cn/android/java-annotation-android-open-source-analysis/)

[Java Annotation 及几个常用开源项目注解原理简析](http://www.tuicool.com/articles/QB73eyf)

[【Android】ButterKnife 简单原理实现](http://www.itnose.net/detail/6291539.html)