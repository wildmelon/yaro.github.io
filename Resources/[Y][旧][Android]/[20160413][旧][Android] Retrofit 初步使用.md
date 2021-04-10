## 备注
**原发表于2016.04.13，资料已过时，仅作备份，谨慎参考**

## Retrofit 是什么？
Retrofit is a type-safe HTTP client for Android and java.

互联网上的资料很多很杂，在收集资料初步了解后，我先粗糙地认为：Retrofit 适用于与 Web 服务器提供的 API 接口进行通信。

当你想要做更多的 HTTP 操作时，可以使用 OkHttp，Retrofit的底层也是由 OkHttp 网络加载库来支持的。

关于 Retrofit 的原理，有三个十分重要的概念：『注解』，『动态代理』，『反射』。将会在以后逐步进行分析。

## 初步使用 Retrofit
Retrofit 在使用上与其他网络开源库有些区别，初次使用可能会感到困惑，其使用主要有四个步骤。

在使用前，我们首先假设我们要从某个 API 接口来获取数据，这里我们使用一位博主所提供的接口。接口的 URL 地址如下：
> https://api.github.com/users/Guolei1130

### 添加依赖和权限
在 build.gradle 文件中添加依赖，在 Manifest.xml文件中添加所需的网络权限。
    
    // build.gradle
    compile 'com.squareup.retrofit:retrofit:2.0.1-beta2'
    compile 'com.squareup.retrofit:converter-gson:2.0.0-beta2'
    
    // AndroidManifest.xml
    <uses-permission android:name="android.permission.INTERNET" />

在 Retrofit 2.0 中，如果要将 JSON 数据转化为 Java 实体类对象，需要自己显式指定一个 Gson Converter。

### 定义接口
在这一步，需要将我们的 API 接口地址转化成一个 Java 接口。

我们的 API 接口地址为：
> https://api.github.com/users/Guolei1130

转化写成 Java 接口为：

    public interface APIInterface {
      @GET("/users/{user}")
      Call<TestModel> repo(@Path("user") String user);

在后文构造 Retrofit 对象时会添加一个 baseUrl（https://api.github.com）。

在此处 GET 的意思是 发送一个 GET请求，请求的地址为：baseUrl + "/users/{user}"。

{user} 类似于占位符的作用，具体类型由 repo(@Path("user") String user) 指定，这里表示 {user} 将是一段字符串。

Call<TestModel> 是一个请求对象，<TestModel>表示返回结果是一个 TestModel 类型的实例。

### 定义 Model
请求会将 Json 数据转化为 Java 实体类，所以我们需要自定义一个 Model：

    public class TestModel {
      private String login;

      public String getLogin() {
        return login;
      }

      public void setLogin(String login) {
        this.login = login;
      }
    }

### 进行连接通信
现在我们有了『要连接的 Http 接口』和 『要返回的数据结构』，就可以开始执行请求啦。

首先，构造一个 Retrofit 对象：

    Retrofit retrofit= new Retrofit.Builder()
    .baseUrl("https://api.github.com")
    .addConverterFactory(GsonConverterFactory.create())
    .build();
注意这里添加的 baseUrl 和 GsonConverter，前者表示要访问的网站，后者是添加了一个转换器。

接着，创建我们的 API 接口对象，这里 APIInterface 是我们创建的接口：

    APIInterface service = retrofit.create(APIInterface.class);

使用 APIInterface 创建一个『请求对象』：

    Call<TestModel> model = service.repo("Guolei1130");
注意这里的 .repo("Guolei1130") 取代了前面的 {user}。到这里，我们要访问的地址就成了：

> https://api.github.com/users/Guolei1130

可以看出这样的方式有利于我们使用不同参数访问同一个 Web API 接口，比如你可以随便改成 .repo("ligoudan")

最后，就可以发送请求了！

    model.enqueue(new Callback<TestModel>() {
      @Override
      public void onResponse(Call<TestModel> call, Response<TestModel> response) {
        // Log.e("Test", response.body().getLogin());
        System.out.print(response.body().getLogin());
      }

      @Override
      public void onFailure(Call<TestModel> call, Throwable t) {
        System.out.print(t.getMessage());
      }
      });

至此，我们就利用 Retrofit 完成了一次网络请求。

## Retrofit 的注解
Retrofit 中有许多用到注解的地方，本次文章先了解他们的用法和作用，之后再深入了解其源码特点。

Retrofit 中，有许多的注解：
![](http://upload-images.jianshu.io/upload_images/1903766-676e3b7c1ce7c6c2?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中，包含了与请求方法相关的 @GET、@POST、@HEAD、@PUT、@DELETA、@PATCH，和参数相关的@Path、@Field、@Multipart等。

在之前转化接口时，我们是这样写的：

    public interface APIInterface {
      @GET("/users/{user}")
      Call<TestModel> repo(@Path("user") String user);

可以看到 @GET 很明显就是请求相关的；而 @Path 我们用它来充当一个占位符的功能，它是参数相关的。

### Header 设置
当我们要设置网络请求的 Header 参数时，Retrofit 提供两种方式进行配置。

第一种是静态配置，直接在接口中指定 Header 参数：

    @Headers({
      "User-Agent: Retrofit-Sample-App"
    })

第二种是动态配置：

    @GET("/user")
    Call<TestModel> getUser(@Header("Authorization") String authorization)

在接口中注解但不指定，后面实例化请求体时可通过 .getUser 指定 Header。

### GET 请求参数设置
在我们发送 GET 请求时，如果需要设置 GET 时的参数，Retrofit 注解提供两种方式来进行配置。分别是 @Query（一个键值对）和 @QueryMap（多对键值对）。

    Call<TestModel> one(@Query("username") String username);
    Call<TestModel> many(@QueryMap Map<String, String> params);

### POST 请求参数设置
POST 的请求与 GET 请求不同，POST 请求的参数是放在请求体内的。

所以当我们要为 POST 请求配置一个参数时，需要用到 @Body 注解：

    Call<TestModel> post(@Body User user);

这里的 User 类型是需要我们去自定义的：

    public class User {
      public String username;
      public String password;

      public User(String username,String password){
        this.username = username;
        this.password = password;
    }

最后在获取请求对象时：
    User user = new User("lgd","123456");
    Call<TestModel> model = service.post(user);

就能完成 POST 请求参数的发送，注意该请求参数 user 也会转化成 Json 格式的对象发送到服务器。

## 总结
以上便是对 Retrofit 的初步介绍和使用，可以看到如果 Web 服务器的 API 接口做的足够规范，各个实体类的配置正确，使用 Retrofit 相比其他网络加载库，可以说是十分简洁明了。

另外在搜索资料时，发现对于 Retrofit 的讲解相对不多，Retrofit 2.0 与其之前的版本也有诸多不同，感谢各位博主提供的细致解读与分享。

## 参考资料
[Retrofit 首页](http://square.github.io/retrofit/)

[快速Android开发系列网络篇之Retrofit](http://www.cnblogs.com/angeldevil/p/3757335.html)

[Android 网络开源库之-retrofit](http://blog.csdn.net/qq_21430549/article/details/50113387#t6)

[Retrofit2 源码解析](http://www.jianshu.com/p/c1a3a881a144)

[Unable to create converter for my class in Android Retrofit library](http://stackoverflow.com/questions/32367469/unable-to-create-converter-for-my-class-in-android-retrofit-library)