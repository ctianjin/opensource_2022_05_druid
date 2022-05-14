# Druid 监控

## 如何监控
```
public interface PreparedStatementProxy extends PreparedStatement, StatementProxy 
作为实现类：PreparedStatementProxyImpl，就持有一个java.sql.PreparedStatement。
```
查询方法：
```
public ResultSet executeQuery() throws SQLException { 
..... 
        return createChain().preparedStatement_executeQuery(this);//产生过滤链，并由过滤链执行。 
    } 
  FilterChainImpl中有： 
    public ResultSetProxy preparedStatement_executeQuery(PreparedStatementProxy statement) throws SQLException { 
        if (this.pos < filterSize) { 
            return nextFilter().preparedStatement_executeQuery(this, statement); 
        } 
        ResultSet resultSet = statement.getRawObject().executeQuery(); 
        return wrap(statement, resultSet); 
    } 
```  上面的方法说明：在执行查询前，要经过过滤链处理，等处理完了，再由statement执行，执行完了，得到一个ResultSet后，包装一个产生最后返回的代理类。 

## 统计过滤器
StatFilter中的一个统计连接提交的方法：
``` 
public void connection_commit(FilterChain chain, ConnectionProxy connection) throws SQLException { 
        chain.connection_commit(connection); 
        JdbcDataSourceStat dataSourceStat = chain.getDataSource().getDataSourceStat(); 
        dataSourceStat.getConnectionStat().incrementConnectionCommitCount(); 
    } 
``` 
先是让chain去作一步（就是nextFilter开始干活，所有的filter都干完了，就真正commit一下）然后，对数据源的commit的操作计数进行增加。
```  
    public void connection_commit(ConnectionProxy connection) throws SQLException { 
        if (this.pos < filterSize) { 
            nextFilter().connection_commit(this, connection);//让下一个干活 
            return; 
        } 
        connection.getRawObject().commit();//都干完了，才真正提交。这个连接也是一个代理，让里面真正的java.sql.connection提交。 
    } 
    private Filter nextFilter() { 
        Filter filter = getFilters().get(pos++); 
        return filter; 
    } 
``` 
DataSourceProxyConfig中有一个private final List<Filter> filters = new ArrayList<Filter>();//就是普通的arraylist放过滤器。 

上面两点合在一起，就是原来执行一个数据库操作，现在给代理类执行，执行中先经过一个个过滤器进行统计，之后再真正执行数据库操作。对最终用户透明的。

执行计数：
``` 
　　private final AtomicLong    createCount      = new 　　AtomicLong(0);                                     // 执行createStatement的计数 
    private final AtomicLong    prepareCount     = new AtomicLong(0);                                     // 执行parepareStatement的计数 
    private final AtomicLong    prepareCallCount = new AtomicLong(0);                                     // 执行preCall的计数 
    private final AtomicLong    closeCount       = new AtomicLong(0); 
``` 
DruidConnectionHolder：
``` 
　　private final DruidAbstractDataSource       dataSource; 
    private final Connection                    conn; 
    private final List<ConnectionEventListener> connectionEventListeners = new CopyOnWriteArrayList<ConnectionEventListener>(); 
    private final List<StatementEventListener>  statementEventListeners  = new CopyOnWriteArrayList<StatementEventListener>(); 
    private PreparedStatementPool               statementPool;   //这是一个LRU算法的池。放的就是下面的PreparedStatementHolder!!!! 
    private final List<Statement>               statementTrace           = new ArrayList<Statement>(2); 

PreparedStatementHolder中呢？new DruidPooledPreparedStatement时，正好用这个holder。 
    private final PreparedStatementKey key; 
    private final PreparedStatement    statement; 
``` 
Holder从名字来看就是持有什么，DruidConnectionHolder必然持有Connection，PreparedStatementHolder必然持有PreparedStatement。DruidConnectionHolder当然还持有属于这个连接的PreparedStatement之类的。透过几个调用关系，差不多可以猜测出设计思路： 
一般我们用connect对象，再产生statment对象，再执行SQL之类的，当我们在一个对象执行前后做一些统计之类的操作时，那就用代理对象来做，比如前面的filterchain用于代理对象中。可是如果调用其它对象时，想把它们之间的一些关联保持下来，比如一个连接下的所有的PreparedStatement，那就需要一个holder对象来帮忙了。

DruidPooledConnection中的PreparedStatement prepareStatement(String sql)方法，就是看看池子里有没有（stmtHolder = holder.getStatementPool().get(key);），没有的话才new PreparedStatementHolder(key, conn.prepareStatement(sql));，有的话内存容器中取了。 

大概关系这样的：DruidPooledConnection-->DruidConnectionHolder-->ConnectionProxy-->filterChain---connection。 