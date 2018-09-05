# RxCache

  这是一个基于  Retrofit 的 Reactive 缓存库，可用于 Android  和 Java 。能够将你的缓存成需求转成一个接口
  

## 一、功能介绍

   实现对数据的缓存


## 二、使用

### 配置到 Android 项目

添加 JitPack 仓库在你的  build.gradle  文件 (项目根目录下):

```
allprojects {
    repositories {
        jcenter()
        maven { url "https://jitpack.io" }
    }
}

```

添加依赖库,在项目模块中

```
dependencies {
    compile "com.github.VictorAlbertos.RxCache:core:1.4.6"
    compile "io.reactivex:rxjava:1.1.5"
}

```

###  定义一个接口  CacheProvider

 

```
public interface CacheProviders {
    //这里设置缓存失效时间为2分钟。
    @LifeCache(duration = 2, timeUnit = TimeUnit.MINUTES)
    Observable<Reply<List<Repo>>> getRepos(Observable<List<Repo>> oRepos, DynamicKey userName, EvictDynamicKey evictDynamicKey);

    @LifeCache(duration = 2, timeUnit = TimeUnit.MINUTES)
    Observable<Reply<List<User>>> getUsers(Observable<List<User>> oUsers, DynamicKey idLastUserQueried, EvictProvider evictProvider);

    Observable<Reply<User>> getCurrentUser(Observable<User> oUser, EvictProvider evictProvider);
}
public interface RestApi {
    String URL_BASE = "https://api.github.com";
    String HEADER_API_VERSION = "Accept: application/vnd.github.v3+json";

    @Headers({HEADER_API_VERSION})
    @GET("/users")
    Observable<List<User>> getUsers(@Query("since") int lastIdQueried, @Query("per_page") int perPage);

    @Headers({HEADER_API_VERSION})
    @GET("/users/{username}/repos")
    Observable<List<Repo>> getRepos(@Path("username") String userName);

    @Headers({HEADER_API_VERSION})
    @GET("/users/{username}") Observable<Response<User>> getUser(@Path("username") String username);
}

```

将 RestApi 中需要缓存的接口方法在 CacheProviders 写相应的方法,如：
```
Observable<List<Repo>> getRepos(@Path("username") String userName);

```

对应

```
Observable<Reply<List<Repo>>> getRepos(Observable<List<Repo>> oRepos, DynamicKey userName, EvictDynamicKey evictDynamicKey);

```

###  创建一个RxCache实例并使用它

   最后,使用  RxCache.Builder 实例化提供者 interface，提供一个有效的文件系统路径允许 RxCache 写磁盘上。
   
   ```
    //cacheDir缓存文件路径
 ort rx.Observable;
import rx.functions.Func1;
import sample_data.cache.CacheProviders;
import sample_data.entities.Repo;
import sample_data.entities.User;
import sample_data.net.RestApi;

public class Repository {
    public static final int USERS_PER_PAGE = 25;

    public static Repository init(File cacheDir) {
        return new Repository(cacheDir);
    }

    private final CacheProviders cacheProviders;
    private final RestApi restApi;

    public Repository(File cacheDir) {
        //persistence设置为缓存文件路径cacheDir,using设置成你所定义的接口类class
        cacheProviders = new RxCache.Builder()
                .persistence(cacheDir)
                .using(CacheProviders.class);

        restApi = new Retrofit.Builder()
                .baseUrl(RestApi.URL_BASE)
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                .addConverterFactory(GsonConverterFactory.create())
                .build().create(RestApi.class);
    }
    /**
     *
     * @param update 是否更新,如果设置为 true，缓存数据将被清理，并且向服务器请求数据
     * @return
     */
    public Observable<Reply<List<User>>> getUsers(int idLastUserQueried, final boolean update) {
        //这里设置idLastUserQueried为DynamicKey,
        return cacheProviders.getUsers(restApi.getUsers(idLastUserQueried, USERS_PER_PAGE), new DynamicKey(idLastUserQueried), new EvictDynamicKey(update));
    }

    //对应每个不同的userName，配置缓存
    public Observable<Reply<List<Repo>>> getRepos(final String userName, final boolean update) {
        //以userName为DynamicKey,如果update为true,将会重新获取数据并清理缓存。
        return cacheProviders.getRepos(restApi.getRepos(userName), new DynamicKey(userName), new EvictDynamicKey(update));
    }

    public Observable<Reply<User>> loginUser(final String userName) {
        return restApi.getUser(userName).map(new Func1<Response<User>, Observable<Reply<User>>>() {
            @Override public Observable<Reply<User>> call(Response<User> userResponse) {

                if (!userResponse.isSuccess()) {
                    try {
                        ResponseError responseError = new Gson().fromJson(userResponse.errorBody().string(), ResponseError.class);
                        throw new RuntimeException(responseError.getMessage());
                    } catch (JsonParseException | IOException exception) {
                        throw new RuntimeException(exception.getMessage());
                    }
                }
                //用户登陆，这里设置 new EvictProvider(true),表示登陆不缓存，为实时登陆
                return cacheProviders.getCurrentUser(Observable.just(userResponse.body()), new EvictProvider(true));
            }
        }).flatMap(new Func1<Observable<Reply<User>>, Observable<Reply<User>>>() {
            @Override public Observable<Reply<User>> call(Observable<Reply<User>> replyObservable) {
                return replyObservable;
            }
        }).map(new Func1<Reply<User>, Reply<User>>() {
            @Override public Reply<User> call(Reply<User> userReply) {
                return userReply;
            }
        });
    }

    public Observable<String> logoutUser() {
        return cacheProviders.getCurrentUser(Observable.<User>just(null), new EvictProvider(true))
                .map(new Func1<Reply<User>, String>() {
                    @Override
                    public String call(Reply<User> user) {
                        return "Logout";
                    }
                })
                .onErrorReturn(new Func1<Throwable, String>() {
                    @Override
                    public String call(Throwable throwable) {
                        return "Logout";
                    }
                });
    }

    public Observable<Reply<User>> getLoggedUser(boolean invalidate) {
        Observable<Reply<User>> cachedUser = cacheProviders.getCurrentUser(Observable.<User>just(null), new EvictProvider(false));

        Observable<Reply<User>> freshUser = cachedUser.flatMap(new Func1<Reply<User>, Observable<Reply<User>>>() {
            @Override public Observable<Reply<User>> call(Reply<User> userReply) {
                return loginUser(userReply.getData().getLogin());
            }
        });

        if (invalidate) return freshUser;
        else return cachedUser;
    }

    private static class ResponseError {
        private final String message;

        public ResponseError(String message) {
            this.message = message;
        }

        public String getMessage() {
            return message;
        }
    }
}         

```

###  DynamicKeyGroup 的使用

```
interface Providers {        
    Observable<List<Mock>> getMocksPaginateWithFiltersEvictingPerFilter(Observable<List<Mock>> oMocks, DynamicKeyGroup filterPage, EvictDynamicKey evictFilter);
}

```
```
public class Repository {
    private final Providers providers;

    public Repository(File cacheDir) {
        providers = new RxCache.Builder()
                .persistence(cacheDir)
                .using(Providers.class);
    }

    public Observable<List<Mock>> getMocksWithFiltersPaginate(final String filter, final int page, final boolean updateFilter) {
        return providers.getMocksPaginateWithFiltersEvictingPerFilter(getExpensiveMocks(), new DynamicKeyGroup(filter, page), new EvictDynamicKey(updateFilter));
    }

    //在实际的使用情况下，这里是当你建立你观察到的与昂贵的操作。
    //如果你正在使用HTTP调用可以改造出来的盒子。
    private Observable<List<Mock>> getExpensiveMocks() {
        return Observable.just(Arrays.asList(new Mock("")));
    }
}

```
### 混淆配置

```
-dontwarn io.rx_cache.internal.**

```
