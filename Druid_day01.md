# Druid 初始化过程

## 安装准备工作

1.maven install

```
wget https://archive.apache.org/dist/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz

mv apache-maven-3.6.3 maven-3.6.3

cd /
vim /etc/profile

# 设置Maven环境变量
MAVEN_HOME=/opt/open/maven-3.6.3
export PATH=$PATH:$MAVEN_HOME/bin

source /etc/profile

mvn clean install-DskipTests
```

2.download druid

git clone https://github.com/alibaba/druid

## 初始化过程code analyse
主要文件:

druid-master\src\main\java\com\alibaba\druid\pool\DruidDataSource.java
druid-master\src\main\java\com\alibaba\druid\pool\DruidAbstractDataSource.java

```
//Line800:

    public void init() throws SQLException {
        if (inited) {
            return;
        }

        // bug fixed for dead lock, for issue #2980
        DruidDriver.getInstance();

        final ReentrantLock lock = this.lock;
        try {
            lock.lockInterruptibly();
        } catch (InterruptedException e) {
            throw new SQLException("interrupt", e);
        }
---

更新JMX监控指标:
                this.connectionIdSeedUpdater.addAndGet(this, delta);
                this.statementIdSeedUpdater.addAndGet(this, delta);
                this.resultSetIdSeedUpdater.addAndGet(this, delta);
                this.transactionIdSeedUpdater.addAndGet(this, delta);
				
解析连接串:
                this.jdbcUrl = this.jdbcUrl.trim();
                initFromWrapDriverUrl();
				
其中initFromWrapDriverUrl:
//从jdbc url中解析出连接和驱动信息，以及将filters的名字，解析成对应的filter类。
    private void initFromWrapDriverUrl() throws SQLException {
        if (!jdbcUrl.startsWith(DruidDriver.DEFAULT_PREFIX)) {
            return;
        }

        DataSourceProxyConfig config = DruidDriver.parseConfig(jdbcUrl, null);
        this.driverClass = config.getRawDriverClassName();

        LOG.error("error url : '" + jdbcUrl + "', it should be : '" + config.getRawUrl() + "'");

        this.jdbcUrl = config.getRawUrl();
        if (this.name == null) {
            this.name = config.getName();
        }

        for (Filter filter : config.getFilters()) {
            addFilter(filter);
        }
    }
	
//filter.init进行filters的初始化：
            filter.init(this);
            this.filters.add(filter);

//解析数据库类型:			
            if (this.dbTypeName == null || this.dbTypeName.length() == 0) {
                this.dbTypeName = JdbcUtils.getDbType(jdbcUrl, null);
            }

//通过SPI加载自定义的filter:
            initFromSPIServiceLoader();
			
//解析驱动：
            resolveDriver();
			
			
//创建连接:
            connections = new DruidConnectionHolder[maxActive];
            evictConnections = new DruidConnectionHolder[maxActive];
            keepAliveConnections = new DruidConnectionHolder[maxActive];
			
//初始化连接(同步或异步)
 if (createScheduler != null && asyncInit) {
                for (int i = 0; i < initialSize; ++i) {
                    submitCreateTask(true);
                }
            } else if (!asyncInit)
			...
		
//开启额外线程:		
            createAndLogThread();
            createAndStartCreatorThread();
            createAndStartDestroyThread();

//为datasource 注册jmx监控指标:
            registerMbean();
	
//连接池使用的核心逻辑:首先，使用数组作为连接的容器，对于真实连接的加入和移除，使用lock进行同步，另外，在加入和移除连接的时候，对比生产消费模型，通过lock上的条件，来通知是否可以获取或者加入连接。	
    public DruidAbstractDataSource(boolean lockFair){
        lock = new ReentrantLock(lockFair);

        notEmpty = lock.newCondition();
        empty = lock.newCondition();
    }
	
```