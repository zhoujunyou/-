### CacheInterceptor

 `Serves requests from the cache and writes responses to the cache.`

1. DiskLruCache用于磁盘存取
2.  CacheStrategy 缓存的策略是否使用网络，缓存或者两者都使用
3. Cache主要用于http的请求响应在DiskLruCache上的存取.

---
CacheStrategy

```java
/** The request to send on the network, or null if this call doesn't use the network. */
public final @Nullable Request networkRequest;

/** The cached response to return or validate; or null if this call doesn't use a cache. */
public final @Nullable Response cacheResponse;
```

用这两个字段判断是否需要使用缓存 具体的http的缓存规则处理在getCandidate()方法中

 isCacheable(Response response, Request request)用来判断是否需要将获取的response缓存下来

