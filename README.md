# RxCache

  这是一个基于  Retrofit 的 Reactive 缓存库，可用于 Android  和 Java 。能够将你的缓存成需求转成一个接口
  

## 一、功能介绍

   实现对数据的缓存


## 二、使用

### 配置到 Android 项目

添加 JitPack 仓库在你的  build.gradle  文件 (项目根目录下):

```
repositories {
    google()
    jcenter()
    maven { url "https://jitpack.io" }
}

```

添加依赖库,在项目模块中

```
dependencies {
    compile "com.github.VictorAlbertos.RxCache:runtime:1.8.3-2.x"
    compile "io.reactivex.rxjava2:rxjava:2.1.6"

  compile 'com.squareup.retrofit2:adapter-rxjava2:2.4.0'
//Caused by: java.lang.IllegalArgumentException: Could not locate call adapter for rx.Observable<>

 compile 'com.squareup.retrofit2:retrofit:2.4.0'
 compile 'com.squareup.retrofit2:converter-gson:2.4.0'

 compile 'com.github.VictorAlbertos.Jolyglot:gson:0.0.3'
 compile 'io.reactivex.rxjava2:rxandroid:2.0.0'
}

```

###   基本的结构


Students               :     和Retrofit 中存放的数据类一样  就是一个Bena类

RestApi            :      Retrofit的接口类

CacheProvider :    用来创建一个RxCaChe的接口

Repository       :    Retrofit和RxCaChe的具体结合操作，以及是一个接口类

MainActivity    :    执行网络请求操作
 


###  1. 使用 GsonFormat 创建一个数据  Bean 类，既 Student  类

###  2. RestApi 类：   跟普通的Retrofit接口没有什么区别
   
   ```
  public interface RestApi {


    @POST("s6/weather/now?parameters")
    Observable<News> getUsers(@Query("key")String key, @Query("location")String location);

}      

```

### 3  CacheProvider类

```
interface Providers {        
    Observable<List<Mock>> getMocksPaginateWithFiltersEvictingPerFilter(Observable<List<Mock>> oMocks, DynamicKeyGroup filterPage, EvictDynamicKey evictFilter);
}

```
请注意：

 1： @LifeCache设置缓存过期时间. 如果没有设置@LifeCache, 数据将被永久缓存除非你使用了其他的清除缓存的标注，这个一会我们再说。

2： 此接口写的里面的第一个参数要写成你Retrofit接口中开头的一段，可以从上边的代码和下边的代码仔细看一下

当然，这里我们只设置了一个参数，也就是必需的参数，如果只设置这个所必需的参数和没有设置@LifeCache那么只要本地缓存没有消失，就会每次都进行本地读取数 据。其他参数是一些 清除缓存的标注，我们先进行一次简单的数据请求与缓存，再写关于其他标注的事情。 

### 4：Repository 类
```
//调用此处，用来生成一个缓存文件
public static Repository init(File cacheDir) {
    return new Repository(cacheDir);
}

private final CacheProvider cacheProvider;
private final RestApi restApi;

public Repository(File cacheDir) {

    cacheProvider = new RxCache.Builder()
            .persistence(cacheDir, new GsonSpeaker())
            .using(CacheProvider.class);


    restApi = new Retrofit.Builder()
            .baseUrl("https://free-api.heweather.com/")
            .addConverterFactory(GsonConverterFactory.create())
            .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
            .build()
            .create(RestApi.class);

}


//真正进行数据请求和缓存的接口
public Observable<Reply<News>> getRepos(String key, String location) {
    return cacheProvider.getRepos(restApi.getUsers(key, location));
}


```
###  5:MainActivity  进行网络请求和缓存操作

```
private Repository repository;

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    repository = Repository.init(getFilesDir());
    initData();
}

private void initData() {

    repository.getRepos("fad6d6b2cda14ef69d260ea9a4415e31", "北京")
            .subscribeOn(Schedulers.newThread())
            .subscribe(new Consumer<Reply<News>>() {
                @Override
                public void accept(Reply<News> newsReply) throws Exception {
                    News.HeWeather6Bean bean = newsReply.getData().getHeWeather6().get(0);
                    Log.e("TagSuccess", bean.getNow().getCond_txt());
                }
            }, new Consumer<Throwable>() {
                @Override
                public void accept(Throwable throwable) throws Exception {
                    Log.e("TagErr", throwable.getMessage());
                }
            });

}

```
