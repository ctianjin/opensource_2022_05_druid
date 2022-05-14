# DruidDriver

## java如何加载数据库驱动
classForName() 加载

显式声明加载,代码如下,最早写jdbc连接的时候都用它
```
try {
    Class.forName("com.mysql.jdbc.Driver");
    connection=DriverManager.getConnection("jdbc:mysql://localhost:3306/test", "root", "root");
} catch (Exception e) {
    e.printStackTrace();
}
```

## SPI
SPI的全称是Service Provider Interface，是Java提供的可用于第三方实现和扩展的机制，通过该机制，我们可以实现解耦，SPI接口方负责定义和提供默认实现，SPI调用方可以按需扩展
好比数据库连接池,各个厂商如oracle,mysql,postregsql等服务商,实现 java.sql.Driver的接口.并且在指定目录Meta-Info/services下按规则声明.程序就会通过加载程序加载进来.
## DriverManager
```
// 这就是java.sql.Driver 中 加载所有在Meta-Info/services目录下的实现类
ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
```
## Driud 加载驱动的过程
DruidDriver getInstance() -> 
registerDriver()->
loadInitialDrivers()
SPI加载机制，获取Meta-Info/services中声明为java.sql.Driver的文件
ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class)

## DruidDriver.getInstance()
连接数据库必须要有一个数据库驱动,在Druid创建连接池的过程中,创建的数据库驱动:
DruidDriver.getInstance();
Driver的创建过程在调用时,触发静态块:
```
// 触发静态方法随着类的加载而执行
static {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            registerDriver(instance);
            return null;
        });
    }

// 注册
public static boolean registerDriver(Driver driver) {
        try {
            // putIfAbsent registeredDrivers 将当前的数据库驱动类加入
            DriverManager.registerDriver(driver);

            try {
                MBeanServer mbeanServer = ManagementFactory.getPlatformMBeanServer();

                ObjectName objectName = new ObjectName(MBEAN_NAME);
                if (!mbeanServer.isRegistered(objectName)) {
                    mbeanServer.registerMBean(instance, objectName);
                }
            } catch (Throwable ex) {
                if (LOG == null) {
                    LOG = LogFactory.getLog(DruidDriver.class);
                }
                LOG.warn("register druid-driver mbean error", ex);
            }

            return true;
        } catch (Exception e) {
            if (LOG == null) {
                LOG = LogFactory.getLog(DruidDriver.class);
            }
            
            LOG.error("registerDriver error", e);
        }

        return false;
    }
    
    
private final static DruidDriver instance = new DruidDriver();

// 被外层调用getInstance()
public static DruidDriver getInstance() {
    return instance;
}
```



