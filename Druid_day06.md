# Druid 

## druid多数据源配置

## 数据源选择器DataSourceSelector
druid-master\src\main\java\com\alibaba\druid\pool\ha\selector\DataSourceSelector.java
```
public interface DataSourceSelector {
    /**
     * Return a DataSource according to the implemention.
     */
    DataSource get();

    /**
     * Set the target DataSource name to return.
     * Wether to use this or not, it's decided by the implemention.
     */
    void setTarget(String name);

    /**
     * Return the name of this DataSourceSelector.
     * e.g. byName
     */
    String getName();

    /**
     * Init the DataSourceSelector before use it.
     */
    void init();

    /**
     * Destroy the DataSourceSelector, maybe interrupt the Thread.
     */
    void destroy();
}

```
NamedDataSourceSelector
druid-master\src\main\java\com\alibaba\druid\pool\ha\selector\NamedDataSourceSelector.java
```
    public NamedDataSourceSelector(HighAvailableDataSource highAvailableDataSource) {
        this.highAvailableDataSource = highAvailableDataSource;
    }
	...
```
HighAvailableDataSource
druid-master\src\main\java\com\alibaba\druid\pool\ha\HighAvailableDataSource.java
```
public class HighAvailableDataSource extends WrapperAdapter implements DataSource 

```
PoolUpdater数据库连接池更新器
druid-master\src\main\java\com\alibaba\druid\pool\ha\node\PoolUpdater.java
```
/**
 * Update the DataSource Connection Pool when notified.
 *
 * @author DigitalSonic
 */
public class PoolUpdater implements Observer 

```
DataSourceCreator(动态创建DataSource)
druid-master\src\main\java\com\alibaba\druid\pool\ha\DataSourceCreator.java
```
/**
 * An utility class to create DruidDataSource dynamically.
 *
 * @author DigitalSonic
 */
public class DataSourceCreator
```

RandomDataSourceSelector(使用随机数随机选择数据源)
druid-master\src\main\java\com\alibaba\druid\pool\ha\selector\RandomDataSourceSelector.java
```
/**
 * A selector which uses java.util.Random to choose DataSource.
 *
 * @author DigitalSonic
 */
public class RandomDataSourceSelector implements DataSourceSelector
```
StickyRandomDataSourceSelector(粘性随机选择数据源)
druid-master\src\main\java\com\alibaba\druid\pool\ha\selector\StickyRandomDataSourceSelector.java
```
/**
 * An extend selector based on RandomDataSourceSelector which can stick a DataSource to a Thread in a while.
 *
 * @author DigitalSonic
 * @see RandomDataSourceSelector
 * @see StickyDataSourceHolder
 */
public class StickyRandomDataSourceSelector extends RandomDataSourceSelector
```

## 配置:

通过注解指定接口对应哪个数据库
```
@Mapper
public class Mapper {
    @DataSource(name = "db1")
    void select(User user);
    
	@DataSource(name = "db2")
    void update(Item item);
}
```
注解定义
```
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface DataSource {
    String name() default "";
}
```
定义注解切面，对注解所在类代理，取注解值，切换数据源
```
@Aspect
@Order(1)
@Log4j2
@Component
public class DataSourceAspect {
    @Pointcut("@annotation(**.DataSource)")
    public void pointCut() {
        //do nothing
    }
    @Around("pointCut()")
    public Object around(ProceedingJoinPoint point) throws Throwable {
        MethodSignature signature = (MethodSignature) point.getSignature();
        Method method = signature.getMethod();
        DataSource dataSource = method.getAnnotation(DataSource.class);
        if (dataSource != null) {
            DataSourceHolder.setDataSourceType(dataSource.name());
        }
        try {
            return point.proceed();
        } finally {
            DataSourceHolder.clearDataSourceType();
        }
    }
}

```
继承动态数据源核心类AbstractRoutingDataSource
```
public class DynamicDataSource extends AbstractRoutingDataSource {
    public DynamicDataSource(DataSource defaultTargetDataSource, Map<Object, Object> targetDataSources) {
        super.setDefaultTargetDataSource(defaultTargetDataSource);
        super.setTargetDataSources(targetDataSources);
        super.afterPropertiesSet();
    }
    @Override
    protected Object determineCurrentLookupKey() {
        return DataSourceHolder.getDataSourceType();
    }
}

```
创建一个线程级datasourceHolder
```
public class DataSourceHolder {

    private DataSourceHolder(){
        //do nothing
    }
    /**
     * threadLocal存储数据源
     */
    private static final ThreadLocal<String> datasourcce = new ThreadLocal<>();
    /**
     * 设置当前数据源
     */
    public static void setDataSourceType(String type) {
        datasourcce.set(type);
    }
    /**
     * 获取当前数据源
     */
    public static String getDataSourceType() {
        return datasourcce.get();
    }
    /**
     * 清除当前数据源
     */
    public static void clearDataSourceType() {
        datasourcce.remove();
    }
}

```
指定目标DataSources的map，以lookup key为主键
```
@Configuration
public class DataSourceConfig {
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.druid.db1")
    public DruidDataSource dataSource1() {
        return DruidDataSourceBuilder.create().build();
    }
    
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.druid.db2")
    public DruidDataSource dataSource2() {
        return DruidDataSourceBuilder.create().build();
    }

    @Primary
    @Bean(name = "dynamicDataSource")
    public DynamicDataSource dataSource(@Qualifier("dataSource1") DataSource dataSource1,
                                        @Qualifier("dataSource2") DataSource dataSource2) {
        Map<Object, Object> targetDataSources = new HashMap<>(3);
        targetDataSources.put("db1", dataSource1);
        targetDataSources.put("db2", dataSource2);
        return new DynamicDataSource(dataSource1, targetDataSources);
    }
}

```
yml配置
```
spring:
 datasource:
  druid:
    db1:
      driver-class-name: oracle.jdbc.OracleDriver
      username: user
      password: pwdxxx
      url: urlExp
    db2:
      driver-class-name: com.microsoft.sqlserver.jdbc.SQLServerDriver
      username: user
      password: pwdxxx
      url: urlExp

```
springboot自动配置排除数据源autoConfigure
```
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})

```


