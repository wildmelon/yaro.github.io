## 备注
**原发表于2016.05.03，资料已过时，仅作备份，谨慎参考**

## 前言
大家好，我又来学习 Retrofit 了，可能这是最后一篇关于 Retrofit 框架的文章了。我发现源码分析这回事，当时看明白了，过些时候再看就想这写的啥玩意。所以大家还是多看多学多分析。

另外跟我自己文章结构组织也有很大关系，我尽量在以后加强这点，做到简洁清晰有层次。

![](http://upload-images.jianshu.io/upload_images/1903766-8eee8f999951558c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这周我们要分析一下 ServiceMethod。

## ServiceMethod 是什么
ServiceMethod 会将你的接口方法调用转化为一个 Call 对象。也就说对于每一个接口方法，他都会创建一个与之对应的 ServiceMethod，实际上上篇文章也有提到。

那么 ServiceMethod 是从哪开始的呢？

    MyApi api = retrofit.create(MyApi.class);

这句代码总没有忘记吧？

create 方法根据你的『Http 接口』返回了一个『动态代理』，当我们调用接口方法时，会转发到『动态代理』的 invoke 方法。内有三句重点：

    ServiceMethod serviceMethod = loadServiceMethod(method);
    OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
    return serviceMethod.callAdapter.adapt(okHttpCall);

最后一句就不说了，把你的 OkHttpCall 转化适配成指定类型的 Call 对象。我们主要分析其余两句。

## loadServiceMethod(method)

### 流程概要
实际上，这个方法我们在上节简单学习过了：

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

当接口方法调用时，会先从缓存（一个 Map）中找，如果没有的话就新建一个，我们看到 ServiceMethod 也是用到建造者模式：

    result = new ServiceMethod.Builder(this, method).build();

传入了 retrofit 对象和我们调用的方法 method 作为参数创建了一个 ServiceMethod：

    public Builder(Retrofit retrofit, Method method) {
      this.retrofit = retrofit;
      this.method = method;
      // 获取 method 中的所有注解
      this.methodAnnotations = method.getAnnotations();
      // 获取 method 中方法的参数类型
      this.parameterTypes = method.getGenericParameterTypes();
      // 获得参数的值
      this.parameterAnnotationsArray = method.getParameterAnnotations();
    }

### build() 方法
然后调用 build() 方法。

build() 方法内容比较多，主要是一些判断并根据 retrofit 对象的实例变量状态创建 ServiceMethod 的变量。关键是一下几行代码：
    
    // 根据 retrofit 对象的 CallAdapterFactory 为 ServiceMethod 创建一个 callAdapter
    callAdapter = createCallAdapter();
    // 根据 retrofit 对象创建一个 responseConverter，默认是一个 BuildInConveter
    responseConverter = createResponseConverter();
    // 解析 method 的所有注解
    for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
    }

前两句代码都很好理解，这里讲一下 parseMethodAnnotation 方法，该方法内容是许多个判断语句，这里我们看其中一个判断：

    if (annotation instanceof DELETE) {
      parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
    } else if (annotation instanceof GET) {
      parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
    } else if ...

可以看出就是对注解的类型进行判断，如果是 DELETE、POST、PATCH 等这样的 Http 请求方式，则调用 parseHttpMethodAndPath 设置 ServiceMethod 实例的 httpMethod 变量。

例如我们的接口是这样的：

    @GET("/users/{user}")
    Call<TestModel> repo(@Path("user") String user);

那么当我们调用 repo 方法时，会将 GET 设置到 httpMethod 中，将其值 "/users/{user}" 进一步解析，设置到 relativeUrl 中。

那么 repo(@Path("user") String user) 里的参数呢？我们之前创建 method 时获取了方法的『参数类型』和『参数值』，所以在解析过外层的想 @GET，@Headers 这样的注解后，就会进行方法参数注解的解析：

    parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);

具体逻辑就不多说了，这里通过 Java 的 『反射机制』和『自定义注解』特性，使得我们能够通过一种简单的方式，把我们编写在 Http 接口的信息，都转化为了 ServiceMethod 所必要的变量值。

## OkHttpCall
现在 ServiceMethod 创建好了，我们学习到他包含了许多东西，包括了我们要请求的地址，请求的方式以及 CallAdapter、Converter 等等。

现在我们要用他来构造一个 okHttpCall：

    OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);

当你调用 okHttpCall 的 enqueue 方法或 execute 方法时，如果还没生成一个 okhttp3.Call 对象则会调用以下方法：

    private okhttp3.Call createRawCall() throws IOException {
    Request request = serviceMethod.toRequest(args);
    okhttp3.Call call = serviceMethod.callFactory.newCall(request);
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
    }

这里创建了调用了 ServiceMethod.toRequest(args) 方法构建了一个 HTTP Request 对象：

    RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl, headers,
        contentType, hasBody, isFormEncoded, isMultipart);

可以看到就是把我们编写的信息作为参数去构建的。

这里的 Request 对象和 Call 对象实际上都属于 okHttp 的范畴了，也就是说至此我们的『Http 接口』最终转化为了一个『okhttp3.Call』并将工作交给了 okHttp 去执行。

那么 Retrofit 框架的工作也就大体完成了。

## 结语
实际上这节的内容并不复杂，只是细节的东西太多了，有许多 Retrofit 框架以外的内容，我自己也并不熟悉。所以其实内容比较杂乱，但是 Retrofit 的主要思想就是利用 Java 的特性，来减少了编写代码时的工作量。

这确实是一个框架设计的经典作品，值得大家去深入学习。