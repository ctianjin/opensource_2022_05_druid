# Druid wallfilter

#执行wallfilter 
```
druid-master\src\main\java\com\alibaba\druid\wall\WallFilter.java

//根据 dbType 生成 WallContext

    @Override
    public PreparedStatementProxy connection_prepareStatement(FilterChain chain, ConnectionProxy connection, String sql)
                                                                                                                        throws SQLException {
        String dbType = connection.getDirectDataSource().getDbType();
        WallContext context = WallContext.create(dbType);
        try {
            WallCheckResult result = checkInternal(sql);
            context.setWallUpdateCheckItems(result.getUpdateCheckItems());
            sql = result.getSql();
            PreparedStatementProxy stmt = chain.connection_prepareStatement(connection, sql);
            setSqlStatAttribute(stmt);
            return stmt;
        } finally {
            WallContext.clearContext();
        }
    }
//
//checkInternal主要做了以下几个事情：
//1、检查这个 SQL 是否在白名单中，假如是就直接返回结果。
//2、对 SQL 进行解析，生成 SQLStatement 列表，因为可能存在复合语句。
//3、调用 SQLStatement 的 accept 方法，将 config 生成的 WallVisitor //放进去，然后检查是否会抛出异常，假如会，就代表存在语法错误，记录到 Result 中。
//
private WallCheckResult checkInternal(String sql) throws SQLException {
        WallCheckResult checkResult = provider.check(sql);
        List<Violation> violations = checkResult.getViolations();

        if (violations.size() > 0) {
            Violation firstViolation = violations.get(0);
            if (isLogViolation()) {
                LOG.error("sql injection violation, dbType "
                        + getDbType()
                        + ", druid-version "
                        + VERSION.getVersionNumber()
                        + ", "
                        + firstViolation.getMessage() + " : " + sql);
            }

            if (throwException) {
                if (violations.get(0) instanceof SyntaxErrorViolation) {
                    SyntaxErrorViolation violation = (SyntaxErrorViolation) violations.get(0);
                    throw new SQLException("sql injection violation, dbType "
                            + getDbType() + ", "
                            + ", druid-version "
                            + VERSION.getVersionNumber()
                            + ", "
                            + firstViolation.getMessage() + " : " + sql,
                            violation.getException());
                } else {
                    throw new SQLException("sql injection violation, dbType "
                            + getDbType()
                            + ", druid-version "
                            + VERSION.getVersionNumber()
                            + ", "
                            + firstViolation.getMessage()
                            + " : " + sql);
                }
            }
        }

        return checkResult;
    }
	
//provider初始化
    public synchronized void init(DataSourceProxy dataSource)
	...
	    DbType dbType = DbType.of(this.dbTypeName);

        switch (dbType) {
            case mysql:
            case oceanbase:
            case drds:
            case mariadb:
            case tidb:
            case h2:
            case presto:
            case trino:
                if (config == null) {
                    config = new WallConfig(MySqlWallProvider.DEFAULT_CONFIG_DIR);
                }

                provider = new MySqlWallProvider(config);
其中，config为配置的 WallFilter 相关的 config 配置信息

	
```

## 数据源配置
```
import java.util.HashMap;
import java.util.Map;

import javax.sql.DataSource;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.DependsOn;

import com.alibaba.druid.pool.DruidDataSourceFactory;
import com.alibaba.druid.wall.WallConfig;
import com.alibaba.druid.wall.WallFilter;

@Configuration
public class DatasourceConfig {

	@Value("${spring.datasource.1.url}")
	private String url;
	@Value("${spring.datasource.1.username}")
	private String username;
	@Value("${spring.datasource.1.password}")
	private String password;
	@Value("${spring.datasource.1.driver-class-name}")
	private String driverClassName;

	@Bean(name = "dataSource")
	public DataSource dataSource() {
		Map<String, Object> properties = new HashMap<>();
		properties.put(DruidDataSourceFactory.PROP_DRIVERCLASSNAME, driverClassName);
		properties.put(DruidDataSourceFactory.PROP_URL, url);
		properties.put(DruidDataSourceFactory.PROP_USERNAME, username);
		properties.put(DruidDataSourceFactory.PROP_PASSWORD, password);
		// 添加统计、SQL注入、日志过滤器
		properties.put(DruidDataSourceFactory.PROP_FILTERS, "stat,wallFilter");
		// sql合并，慢查询定义为5s
		properties.put(DruidDataSourceFactory.PROP_CONNECTIONPROPERTIES,
				"druid.stat.mergeSql=true;druid.stat.slowSqlMillis=5000");
		try {
			return DruidDataSourceFactory.createDataSource(properties);
		} catch (Exception e) {
			e.printStackTrace();
		}
		return null;
	}

	@Autowired
	WallFilter wallFilter;

	@Bean(name = "wallConfig")
	WallConfig wallFilterConfig() {
		WallConfig wc = new WallConfig();
		wc.setMultiStatementAllow(true);
		return wc;
	}

	@Bean(name = "wallFilter")
	@DependsOn("wallConfig")
	WallFilter wallFilter(WallConfig wallConfig) {
		WallFilter wfilter = new WallFilter();
		wfilter.setConfig(wallConfig);
		return wfilter;
	}
}
```

## wallFilter 
```
druid-master\src\main\java\com\alibaba\druid\wall\WallFilter.java

public class WallFilter extends FilterAdapter implements WallFilterMBean

```

## WallFilterTest
```
druid-master\src\test\java\com\alibaba\druid\bvt\filter\wall\WallFilterTest.java

public class WallFilterTest extends TestCase {

    private DruidDataSource dataSource;
    private WallFilter      wallFilter;

    protected void setUp() throws Exception {
        dataSource = new DruidDataSource();

        dataSource.setUrl("jdbc:h2:mem:wall_test;");
        dataSource.setFilters("wall");
        dataSource.init();

        wallFilter = (WallFilter) dataSource.getProxyFilters().get(0);
    }

    protected void tearDown() throws Exception {
        dataSource.close();
    }

    public void test_wallFilter() throws Exception {
        {
            Connection conn = dataSource.getConnection();
            Statement stmt = conn.createStatement();
            stmt.execute("CREATE TABLE t (FID INTEGER, FNAME VARCHAR(50))");
            stmt.close();
            conn.close();
        }
        Assert.assertEquals(1, wallFilter.getProvider().getTableStat("t").getCreateCount());

        {
            Connection conn = dataSource.getConnection();
            Statement stmt = conn.createStatement();
            for (int i = 0; i < 10; ++i) {
                stmt.execute("INSERT INTO t (FID, FNAME) VALUES (" + i + ", 'a" + i + "')");
            }
            stmt.close();
            conn.close();
        }
        {
            String sql = "SELECT * FROM T";

            Connection conn = dataSource.getConnection();
            PreparedStatement stmt = conn.prepareStatement(sql);
            ResultSet rs = stmt.executeQuery();
            while (rs.next()) {

            }
            rs.close();
            stmt.close();
            conn.close();
        }
        Assert.assertEquals(10, wallFilter.getProvider().getTableStat("t").getFetchRowCount());
        {
            Connection conn = dataSource.getConnection();
            Statement stmt = conn.createStatement();
            stmt.execute("DELETE from t where FID = 0");
            stmt.close();
            conn.close();
        }
        Assert.assertEquals(1, wallFilter.getProvider().getTableStat("t").getDeleteDataCount());
        {
            Connection conn = dataSource.getConnection();
            Statement stmt = conn.createStatement();
            stmt.execute("DELETE from t where FID = 1 OR FID = 2");
            stmt.close();
            conn.close();
        }
        Assert.assertEquals(3, wallFilter.getProvider().getTableStat("t").getDeleteDataCount());

        {
            Connection conn = dataSource.getConnection();
            Statement stmt = conn.createStatement();
            stmt.execute("update t SET fname = 'xx' where FID = 3 OR FID = 4");
            stmt.close();
            conn.close();
        }
        Assert.assertEquals(2, wallFilter.getProvider().getTableStat("t").getUpdateDataCount());

        {
            Connection conn = dataSource.getConnection();
            PreparedStatement stmt = conn.prepareStatement("update t SET fname = 'xx' where FID = ? OR FID = ?");
            stmt.setInt(1, 3);
            stmt.setInt(2, 4);
            stmt.execute();
            stmt.close();
            conn.close();
        }
        Assert.assertEquals(4, wallFilter.getProvider().getTableStat("t").getUpdateDataCount());
        
        {
            Connection conn = dataSource.getConnection();
            PreparedStatement stmt = conn.prepareStatement("update t SET fname = 'xx' where FID = ? OR FID = ?");
            stmt.setInt(1, 3);
            stmt.setInt(2, 4);
            stmt.execute();
            stmt.close();
            conn.close();
        }
        Assert.assertEquals(6, wallFilter.getProvider().getTableStat("t").getUpdateDataCount());
        {
            Connection conn = dataSource.getConnection();
            PreparedStatement stmt = conn.prepareStatement("update t SET fname = 'xx' where FID = ?");
            
            stmt.setInt(1, 3);
            stmt.addBatch();
            
            stmt.setInt(1, 4);
            stmt.addBatch();
            
            stmt.executeBatch();
            stmt.close();
            conn.close();
        }
        Assert.assertEquals(8, wallFilter.getProvider().getTableStat("t").getUpdateDataCount());

        {
            Connection conn = dataSource.getConnection();
            Statement stmt = conn.createStatement();
            stmt.execute("truncate table t");
            stmt.close();
            conn.close();
        }
        Assert.assertEquals(1, wallFilter.getProvider().getTableStat("t").getTruncateCount());
        {
            Connection conn = dataSource.getConnection();
            Statement stmt = conn.createStatement();
            stmt.execute("drop table t");
            stmt.close();
            conn.close();
        }
        Assert.assertEquals(1, wallFilter.getProvider().getTableStat("t").getDropCount());
    }
}


```