## 备注
**原发表于2016.04.23，资料已过时，仅作备份，谨慎参考**

## 前言
由于是第一次自己翻看源代码进行学习，加上基础不好，在看源代码的过程中简直痛苦不堪，但同时也暴露出了自己的许多问题。我觉得学习源代码是一件耗时但也收益颇多的学习方式，哪怕你暂时没有足够的时间自己去分析学习，也要擅于学习别人的经验总结。

## Java 基础知识点
Retrofit 的功能涉及到了 Java 的『反射』、『注解』和『动态代理』。

[公共技术点之 Java 反射 Reflection](http://a.codekk.com/detail/Android/Mr.Simple/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%20Java%20%E5%8F%8D%E5%B0%84%20Reflection)

[公共技术点之 Java 注解 Annotation](http://a.codekk.com/detail/Android/Trinea/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%20Java%20%E6%B3%A8%E8%A7%A3%20Annotation)

[公共技术点之 Java 动态代理](http://a.codekk.com/detail/Android/Caij/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%20Java%20%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86)

## Retrofit 的主要接口和类
Retrofit 的代码并不是很多，其底层网络通信时交由 OkHttp 来完成的。其包结构如下图所示：
![](http://upload-images.jianshu.io/upload_images/1903766-bb8eadd9c73c063f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/1903766-af92652930ef6203.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中 retrofit2.http 包里面全部是用来定义 HTTP 请求的自定义注解。
### 接口
#### Call<T>
Call 接口的主要作用是发送一个 Http 请求，在 Retrofit 中的默认实现是 OkHttpCall<T>，也可以根据实际情况实现自己的 Call 类。Call 实现类需要实现两个请求发送方法：
    
    // 同步请求方法，返回请求的结果
    Response<T> execute() throws IOException;

    // 异步请求方法，在 CallBack 中处理返回的结果
    void enqueue(Callback<T> callback);

#### Callback
Call 的回调，该接口是 Retrofit 异步请求数据返回的接口，包含两个方法：
    
    void onResponse(Call<T> call, Response<T> response);
    void onFailure(Call<T> call, Throwable t);
    
#### Converter
数据转换器，该接口将 Http 请求返回的数据解析成 Java 对象，我们之前创建 Retrofit实例时有一句：

    .addConverterFactory(GsonConverterFactory.create())

就是添加了一个 GsonConverter 来使用 Gson 将我们的结果转换成 Model 类。

#### CallAdapter
Call 的适配器，负责将 Call 对象转化成另一个对象，同样可在创建 Retrofit 实例时调用 .addCallAdapterFactory(Factory) 来添加。

## Retrofit 执行步骤
在上一篇介绍 Retrofit 初步使用的文章里，已经知道 Retrofit 使用的基本步骤，这里再重新简单地介绍一遍：

    // 创建接口
    public interface APIInterface {
        @GET("/users/{user}")
        Call<TestModel> repo(@Path("user") String user);
      }

    // 创建 Retrofit 实例
    Retrofit retrofit= new Retrofit.Builder()
    .baseUrl("https://api.github.com")
    .addConverterFactory(GsonConverterFactory.create())
    .build();

    // 生成接口实现类
    APIInterface service = retrofit.create(APIInterface.class);

    // 调用接口实现类的请求方法，获取 Call 对象
    Call<TestModel> model = service.repo("Guolei1130");

    // 调用 Call 对象的异步执行方法
    model.enqueue(Callback callback)

## Retrofit 步骤分析
这里从代码层面来对上述步骤中的关键进行简要的分析：

### 创建 Retrofit 实例，生成接口的实现类
生成接口实现类时，编写了以下语句：

    APIInterface service = retrofit.create(APIInterface.class);

.create()的代码如下所示：

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

这里使用了『动态代理』，返回了一个 Proxy 代理类，调用接口中的任何方法都会调用 proxy 里的 invoke 方法。

### 调用接口实现类的请求方法，获取 Call 对象
接着上一步分析，我们知道当我们调用请求方法时：

    Call<TestModel> model = service.repo("Guolei1130");

实际上会调用到 Proxy 的 invoke 方法。在该方法内，下面的三行代码是最为主要和重要的：

    ServiceMethod serviceMethod = loadServiceMethod(method);
    OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
    return serviceMethod.callAdapter.adapt(okHttpCall);

第一行 loadServiceMethod(method)：

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

该方法会根据 接口请求方法 method 来建造一个 ServiceMethod，put 到缓存里并返回。在建造 ServiceMethod时，会调用 createCallAdapter() 来为 ServiceMethod 添加一个 CallAdapter：

    // ServiceMethod.Builder(this, method).build();
    public ServiceMethod build() {
      callAdapter = createCallAdapter();
      ...
    }

    // ServiceMethod.createCallAdapter()
    private CallAdapter<?> createCallAdapter() {
      Type returnType = method.getGenericReturnType();
      if (Utils.hasUnresolvableType(returnType)) {
        throw methodError(
            "Method return type must not include a type variable or wildcard: %s", returnType);
      }
      if (returnType == void.class) {
        throw methodError("Service methods cannot return void.");
      }
      Annotation[] annotations = method.getAnnotations();
      try {
        return retrofit.callAdapter(returnType, annotations);
      } catch (RuntimeException e) { // Wide exception range because factories are user code.
        throw methodError(e, "Unable to create call adapter for %s", returnType);
      }
    }

这里会根据 method 的返回类型来创建相应的 CallAdapter，由于 Retrofit 的默认实现是 OkHttpCall，所以在这里会创建一个默认的 DefaultCallAdapter。至此，我们再回到 invoke() 的最后一行返回语句：

    return serviceMethod.callAdapter.adapt(okHttpCall);

调用了 DefaultCallAdapter 不对传进来的 Call 对象做任何处理，所以我们通过调用接口实现类的方法，实际上最终获得了一个 OkHttpCall 对象，其具有两个请求执行方法，一个同步，一个异步。

调用同步方法时，会使用应用线程来发送请求；调用异步方法时会通过 OkHttp 的 Dispatcher 提供的线程来执行请求。

## 总结
通过本文，较为粗糙地从表层了解了 Retrofit 执行步骤中一步步在代码中传递的过程，尽管暂时没能深入代码内部透彻解析各个类和方法，但是通过这次分析，我自己对 Retrofit 的基本步骤已经有了更为深入的了解。

另外这种对框架，不能说了如指掌，但对他的实现多了一份认识的感觉，真的只有自己去研究过才能体会得到。

也希望各位大神能指出本文中结构或理解上的一些错误，同时希望这篇文章能帮助您粗略了解 Retrofit 的运行。

## 参考资料
[Retrofit2 源码解析](http://www.jianshu.com/p/c1a3a881a144)

[Retrofit分析-漂亮的解耦套路](http://www.jianshu.com/p/45cb536be2f4)

[Retrofit2 源代码初步解读](http://www.jianshu.com/p/0c6837686898)

[Retrofit2 源码解析](http://www.tuicool.com/articles/UryUnyF)