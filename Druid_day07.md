# Druid 
## 使用Druid与MyBatis连接数据库

## Test Code
```
druid-master\src\test\java\com\alibaba\druid\bvt\sql\MybatisTest.java
public class MybatisTest extends TestCase {

    private String sql = "select * from t where id = #{id}";

    public void test_mysql() throws Exception {
        Assert.assertEquals("SELECT *\nFROM t\nWHERE id = #{id}", SQLUtils.format(sql, JdbcUtils.MYSQL));
        Assert.assertEquals("SELECT *\nFROM t\nWHERE id = #{id}", SQLUtils.format(sql, JdbcUtils.OCEANBASE));
    }

    public void test_oracle() throws Exception {
        Assert.assertEquals("SELECT *\nFROM t\nWHERE id = #{id}", SQLUtils.format(sql, JdbcUtils.ORACLE));
        Assert.assertEquals("SELECT *\nFROM t\nWHERE id = #{id}", SQLUtils.format(sql, JdbcUtils.OCEANBASE_ORACLE));
        Assert.assertEquals("SELECT *\nFROM t\nWHERE id = #{id}", SQLUtils.format(sql, JdbcUtils.ALI_ORACLE));
    }

    public void test_postgres() throws Exception {
        Assert.assertEquals("SELECT *\nFROM t\nWHERE id = #{id}", SQLUtils.format(sql, JdbcUtils.POSTGRESQL));
    }

    public void test_sql92() throws Exception {
        Assert.assertEquals("SELECT *\nFROM t\nWHERE id = #{id}", SQLUtils.format(sql, (DbType) null));
    }
}


druid-master\src\test\java\com\alibaba\druid\bvt\pool\SpringMybatisFilterTest.java

public class SpringMybatisFilterTest extends TestCase {

    protected void setUp() throws Exception {
        DruidDataSourceStatManager.clear();
    }

    protected void tearDown() throws Exception {
        Assert.assertEquals(0, DruidDataSourceStatManager.getInstance().getDataSourceList().size());
    }

    public void test_spring() throws Exception {
        Assert.assertEquals(0, DruidDataSourceStatManager.getInstance().getDataSourceList().size());

        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(
                                                                                    "com/alibaba/druid/pool/mybatis/spring-config-mybatis.xml");

        DataSource dataSource = (DataSource) context.getBean("dataSource");

        {
            Connection conn = dataSource.getConnection();
            Statement stmt = conn.createStatement();
            stmt.execute("CREATE TABLE sequence_seed (value INTEGER, name VARCHAR(50))");
            stmt.close();
            conn.close();
        }
        {
            Connection conn = dataSource.getConnection();
            Statement stmt = conn.createStatement();
            stmt.execute("CREATE TABLE t_User (id BIGINT, name VARCHAR(50))");
            stmt.close();
            conn.close();
        }
        {
            Connection conn = dataSource.getConnection();
            conn.setAutoCommit(false);
            Statement stmt = conn.createStatement();
            stmt.execute("insert into sequence_seed (value ,name) values (0, 'druid-spring-test')");
            stmt.close();
            conn.commit();
            conn.close();
        }

        UserMapper userMapper = (UserMapper) context.getBean("userMapper");

        {
            User user = new User();
            user.setName("xx");

            userMapper.addUser(user);
        }
        
        {
            userMapper.errorSelect(1);
        }
        
        {
            Connection conn = dataSource.getConnection();
            Statement stmt = conn.createStatement();
            stmt.execute("DROP TABLE sequence_seed");
            stmt.close();
            conn.close();
        }
        {
            Connection conn = dataSource.getConnection();
            Statement stmt = conn.createStatement();
            stmt.execute("DROP TABLE t_User");
            stmt.close();
            conn.close();
        }

        context.close();

        Assert.assertEquals(0, DruidDataSourceStatManager.getInstance().getDataSourceList().size());
    }

    public static interface UserMapper {

        @Insert(value = "insert into t_User (id, name) values (#{user.id}, #{user.name})")
        void addUser(@Param("user") User user);
        
        @Select(value = "delete from t_User where id = #{id}")
        void errorSelect(@Param("id") long id);
    }
}

druid-master\src\test\java\com\alibaba\druid\bvt\pool\SpringMybatisFilterTest.java

ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(
"com/alibaba/druid/pool/mybatis/spring-config-mybatis.xml");

druid-master\src\test\resources\com\alibaba\druid\pool\mybatis\spring-config-mybatis.xml

	<bean class="org.mybatis.spring.mapper.MapperFactoryBean"
		id="userMapper">
		<property name="sqlSessionFactory" ref="sqlSessionFactory" />
		<property name="mapperInterface" value="com.alibaba.druid.bvt.pool.SpringMybatisFilterTest$UserMapper" />
	</bean>
	


```

## 配置
每个MyBatis应用程序主要都是使用SqlSessionFactory实例的，一个SqlSessionFactory实例可以通过SqlSessionFactoryBuilder获得。SqlSessionFactoryBuilder可以从一个xml配置文件或者一个预定义的配置类的实例获得。
```
@MapperScan(value = { "trade.user.dal.dataobject",
        "trade.user.dal.mapper" }, sqlSessionFactoryRef = "OrderSqlSessionFactory")
@ConditionalOnProperty(name = "yw.order.druid.datasource.url", matchIfMissing = false)
public class MssqlDataSource {
    static final String  MAPPER_LOCATION = "classpath*:sqlconfig/*Mapper.xml";
    @Bean(name = "OrderSqlSessionFactory")
    @ConditionalOnMissingBean(name = "OrderSqlSessionFactory")
    public SqlSessionFactory sqlSessionFactory(@Qualifier("OrderDruidDataSource") DruidDataSource druidDataSource)
            throws Exception {
        final SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();
        sessionFactory.setDataSource(druidDataSource);
        sessionFactory.setMapperLocations(new PathMatchingResourcePatternResolver().getResources(MAPPER_LOCATION));
        SqlSessionFactory sqlSessionFactory = sessionFactory.getObject();
        sqlSessionFactory.getConfiguration().setMapUnderscoreToCamelCase(true);
        return sqlSessionFactory;
    }
}

@MapperScan("trade.user.**")
public class StartMain {
    public static void main(String[] args) {
        SpringApplication.run(StartMain.class, args);
    }
}

@Resource
    OrderDiscountDOMapper orderDiscountDOMapper;
    @RequestMapping(value = "/getInfo")
    public  String getInfo(int id)
    {
        OrderDiscountDO rt=orderDiscountDOMapper.selectByPrimaryKey(1L);
        return  id+"----"+ JSON.toJSONString(rt);
    }
	
	
<dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>1.3.0</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
            <version>2.1.6.RELEASE</version>
        </dependency>
```


