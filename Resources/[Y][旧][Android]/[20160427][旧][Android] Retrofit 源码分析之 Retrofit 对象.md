## 备注
**原发表于2016.04.27，资料已过时，仅作备份，谨慎参考**

## 前言
在上一周学习了一下 [Retrofit 的执行流程](http://blog.qiji.tech/archives/9354)。接下来的文章要更为深入地学习 Retrofit 的各个类，这次我们先学习一下 Retrofit 框架里的 Retrofit 对象，有没有十分的拗口。。

![](http://upload-images.jianshu.io/upload_images/1903766-4275d9987f2d7ddf.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

本文主要讲 Retrofit 对象的创建及其 .create 方法。基本包括了这个类的全部内容。

## Retrofit 对象
### 简介
Retrofit 通过使用方法上的『注解』来定义请求的构成，将我们声明的 Http 接口转化成一个 Call 对象。

这个 Call 对象呢，我们上周提到过，可以调用同步或非同步方法来发送请求，之后就交给 OkHttp 去执行啦。
### 使用
Retrofit 类用到了创建者模式，我们需要使用 Retrofit.Builder 来创建它的实例，接着调用 Retrofit.create(Class<T>) 方法就能够生成我们的接口实现类了。

这里回顾一下 Retrofit 相关的使用：
    
    Retrofit retrofit = new Retrofit.Builder()
     .baseUrl("https://api.example.com/")
     .addConverterFactory(GsonConverterFactory.create())
     .build();

     MyApi api = retrofit.create(MyApi.class);

## Retrofit.Builder
Retrofit.Builder 是 Retrofit 对象的一个嵌套类，负责用来创建 Retrofit 实例对象，使用『建造者模式』的好处是清晰明了可定制化。

在执行 .build() 方法前，只有 .baseUrl() 是必须调用来设置访问地址的，其余方法则是可选的。

首先看一下 Builder.build() 最后的返回语句：

    return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
          callbackExecutor, validateEagerly);

这里的参数包括了 Call 工厂，地址，转换器，CallAdapter 工厂， 执行 Callback 的线程池以及 validateEagerly 标识。

下面我们挑选其中几个参数来进行分析：

### baseUrl
baseUrl 其实是 okHttp3 的 HttpUrl 类实例，一个 http 或者 https 协议的 URL。

为 Retrofit.Builder 添加 baseUrl，有两个重载的方法 baseUrl(String baseUrl) 和 baseUrl(HttpUrl baseUrl)，但实际最后调用的都是后者。

    public Builder baseUrl(HttpUrl baseUrl) {
      checkNotNull(baseUrl, "baseUrl == null");
      List<String> pathSegments = baseUrl.pathSegments();
      if (!"".equals(pathSegments.get(pathSegments.size() - 1))) {
        throw new IllegalArgumentException("baseUrl must end in /: " + baseUrl);
      }
      this.baseUrl = baseUrl;
      return this;
    }

可以看到，检查验证后就设置了 Retrofit 对象的 URL。

### callbackExecutor
callbackExecutor 是 Callback 调用中用于执行 Callback 的 线程池。

如果不自行设置的话，会根据平台设置一个默认的 Executor。

    if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }

这里的 .defaultCallbackExecutor() 是 Platform 抽象类的一个方法。包含了 Converter，Client 等属性。他有三个实现类：Android，Java8，IOS。分别设置了各个平台下的一些默认参数。

在创建 Retrofit.Buidler 时会获取并设置当前环境的 Platform：

    public Builder() {
      this(Platform.get());
    }

最后我们找到 Platform 的安卓实现类看一下：

    static class Android extends Platform {
      @Override public Executor defaultCallbackExecutor() {
      return new MainThreadExecutor();
    }

      @Override CallAdapter.Factory defaultCallAdapterFactory(Executor callbackExecutor) {
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }

      static class MainThreadExecutor implements Executor {
        private final Handler handler = new Handler(Looper.getMainLooper());

        @Override public void execute(Runnable r) {
        handler.post(r);
        }
      }
    }

了解过 Handler 机制的同学肯定十分眼熟，这里获取了主线程的 Looper 并构造了一个 主线程的 Handler，于是在 Android 平台，调用 Callback 时会将该请求 post 到主线程上去执行。

### validateEagerly 标识
validateEagerly 是一个布尔类型的参数

我们知道当我们调用接口方法时，代理类会为方法创建一个 ServiceMethod。

    public <T> T create(final Class<T> service) {
      Utils.validateServiceInterface(service);
      if (validateEagerly) {
      eagerlyValidateMethods(service);
      }
      ...
    }

如果将 validateEagerly 标识设置为 True，那么在我们调用 .eagerlyValidateMethods(service) 方法之前就提前验证并创建好啦。

以上便是 Retrofit.Builder 的一些参数和方法，更具体的大家可以参照官方文档来学习。

## .create 方法
现在我们通过嵌套类 build 了一个 Retrofit 对象，就可以开始执行下一步了。

    // 将 Http 接口 转化为 Call 对象
    MyApi api = retrofit.create(MyApi.class);

我们先直接把 create 方法的代码丢上来，代码并不是很多：

    public <T> T create(final Class<T> service) {
      Utils.validateServiceInterface(service);
      if (validateEagerly) {
      eagerlyValidateMethods(service);
      }
      return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod serviceMethod = loadServiceMethod(method);
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
    }

下面一步步进行分析：

    Utils.validateServiceInterface(service);

validateServiceInterface(service) 会验证我们的 Http 接口是否是 Interface，是否未包含了其他的接口。若为否则会抛出错误。

    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }

validateEagerly 的标签的作用则在之前已经说过了，算是一个提前验证标识。

接下来便返回了一个动态代理，其实仔细看会发现这里只是返回了动态代理的实例方法而已：

    return (T) Proxy.newProxyInstance(...);

代理类首先获取了当前的平台 Platform，然后当你调用接口方法时，会调用到代理类的 invoke 方法。

我们看看 invoke 方法里到底做了什么：

    if (method.getDeclaringClass() == Object.class) {
      return method.invoke(this, args);
    }
    if (platform.isDefaultMethod(method)) {
        return platform.invokeDefaultMethod(method, service, proxy, args);
    }

如果我们调用的是来自 Object 类或者平台默认的方法，则会交给方法执行或者平台执行，但从代码上看 isDefaultMethod(method) 直接返回的是 false，可能是为了方便开发者扩展设置各个平台的不同方法调用。

    ServiceMethod serviceMethod = loadServiceMethod(method);

经过两个判断后，会将我们的方法转换成一个 ServiceMethod 对象，我们可以来看看 loadServiceMethod 方法内发生了什么：

    ServiceMethod loadServiceMethod(Method method) {
      ServiceMethod result;
      synchronized (serviceMethodCache) {
        result = serviceMethodCache.get(method);
        if (result == null) {
        result = new ServiceMethod.Builder(this, method).build();
        serviceMethodCache.put(method, result);
        }
      }
      return result;
    }

代码很简单，每个 Method 对应一个 ServiceMethod，如果缓存里没有，则新建一个。至于这个 ServiceMethod 是什么呢？我们具体可能要以后再详细分析。

这里简单的了解一下：之前我们说 Retrofit 对象的作用是将我们声明的 Http 接口转化成一个 Call 对象。实际上真正的工作是由 ServiceMethod 的来完成的，在其内部分析并转换了我们自定义的注解，并生成了一个 Call 对象。

    OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
    return serviceMethod.callAdapter.adapt(okHttpCall);

接下来创建了一个 OkHttpCall。并使用 serviceMethod.CallAdapter 对 OkHttpCall 进行了转化。

我们在创建 serviceMethod 时，传入了 Retrofit 对象作为参数，这个 CallAdapter 就是从我们最开始构建 Retrofit 时所添加的 CallAdapterFatory所生成的。如果你没有设置的话，在 Android 平台，系统会为你设置一个 ExecutorCallAdapterFactory。

ExecutorCallAdapter会先返回一个 CallAdapter 实现类，.adapt(okHttpCall) 就是这个类的方法。

终于，callAdapter.adapt 把 okHttpCall 转化成了 ExecutorCallbackCall：

    @Override public <R> Call<R> adapt(Call<R> call) {
        return new ExecutorCallbackCall<>(callbackExecutor, call);
    }

于是我们就完成了 .create() 方法的调用，实际上 Retrofit 的使用我们也几乎掌握了，因为之后的事情是交给 okHttp 去做的。

我们可以看看这个 ExecutorCallbackCall<>(callbackExecutor, call)，参数里的 callbackExecutor，有没有很眼熟，之前 Retrofit.Builder 我们提到的默认添加的 Executor，这里其实就是我们 APP 应用的主线程。

也就是我们的网络请求完成后 Callback 回调的 onResponse 和 onFailure 方法，都会 post 到主线程上的 Handler 来执行。

## 总结
似乎这次文章的内容有点长？总结一句话就是：Retrofit 如何将 Http 接口方法调用转换成一个 Call 请求类。

这次我在学习代码和写文章到最后时，确实发现了之前的许多错误。目前难免会有许多理解不到位的地方，文章也写的比较散乱，希望各位能多多提出意见。

## 参考资料
[Retrofit 2.0.0 API](http://square.github.io/retrofit/2.x/retrofit/)

[快速Android开发系列网络篇之Retrofit](http://www.cnblogs.com/angeldevil/p/3757335.html)