## @Cacheable 和 @CacheEvict 不生效

由于 `@Cacheable` 和 `@CacheEvict` 是通过 Spring AOP 代理实现的，因此会导致以下两个问题：

1.  `@Cacheable` 和 `@CacheEvict` 标识的方法所在 Bean 需要是代理生成内部类。

   ```java
   @RequiredArgsConstructor
   @Slf4j
   public class CacheUtils implements ApplicationContextAware, ApplicationRunner {
       private final AccountService accountService;
       
       @Cacheable(value = "account:cache:60m", key = "#accountId", unless = "#result == null")
       public Account getAccount(Long accountId) {
           return accountService.get(accountId);
       }
   
       /**
        * 删除账号缓存
        */
       @CacheEvict(value = "account:cache:60m", key = "#accountId")
       public void delAccount(Long accountId) {
           log.info("del Account, id={}", accountId);
       }
   }
   ```

   ```java
   @RequiredArgsConstructor
   @Slf4j
   public class ContentUtil implements ApplicationContextAware, ApplicationRunner {
   
       private CacheUtils cacheUtils;
   
       @Override
       public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
           this.applicationContext = applicationContext;
       }
   
       @Override
       public void run(ApplicationArguments args) throws Exception {
           cacheUtils = applicationContext.getBean(CacheUtils.class);
       }
       
       public Account getAccount(Long accountId) {
           return cacheUtils.getAccount(accountId);
       }
   }
   ```

   ```java
   @Bean
   public CacheUtil cacheUtil(AccountService accountService) {
       return new CacheUtil(accountService);
   }
   
   @Bean
   public ContentUtil contentUtil() {
       return new ContentUtil();
   }
   ```

   上面的 `ContentUtil` 不能通过直接通过 `new ContentUtil(cacheUtil)` 构造参数直接传入 `CacheUtil` ，如果直接传入 `CacheUtil`，此时这个 Bean 是直接创建，不是代理后的内部类。导致 `ContentUtil` 持有的 `CacheUtil` 的方法没有 AOP 代理逻辑。

2.  `@Cacheable` 和 `@CacheEvict` 修饰的方法不能在该 Bean 内部方法调用。

   ```java
   @RequiredArgsConstructor
   @Slf4j
   public class CacheUtils implements ApplicationContextAware, ApplicationRunner {
       private final AccountService accountService;
       
       @Cacheable(value = "account:cache:60m", key = "#accountId", unless = "#result == null")
       public Account getAccount(Long accountId) {
           return accountService.get(accountId);
       }
   
       /**
        * 删除账号缓存
        */
       @CacheEvict(value = "account:cache:60m", key = "#accountId")
       public void delAccount(Long accountId) {
           log.info("del Account, id={}", accountId);
       }
       
       public Account innerGetAccount(Long accountId) {
           return this.getAccount(accountId);
       }
   }
   ```

   上述通过调用 `CacheUtil.innerGetAccount()` 时不会触发缓存，其原因在于当进入 `innerGetAccount()` 方法时，此时的 `CacheUtil` 已经不是代理的内部类，而是具体的 Bean，此时再调用 `getAccount()` 时，不会有相关代理逻辑。



## @Cacheable 默认key

`@Cacheable` 默认key为 ""。

如果需要自定义 key，则可以通过实现 `KeyGenerator` 接口，`@Cacheable` 默认会通过 `KeyGenerator` 实现类获取缓存 key。

### 缓存清空问题

如果缓存 key 为自定义时，此时会有一个问题：如何清空指定缓存目录下的所有缓存记录？

```java
@CacheEvict(value = "allClub:cache:60m", allEntries = true)
public void delAllClub() {
    log.info("delete all club");
}
```

可以通过 `@CacheEvict` 中的参数 `allEntries = true` ，将指定缓存目录下的所有记录清空。