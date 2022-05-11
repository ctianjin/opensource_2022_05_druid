# Druid 

## 创建连接

druid-master\src\main\java\com\alibaba\druid\pool\DruidDataSource.java

CreateConnectionThread:

```
//Line2809
                    if (emptyWait) {
                        // 必须存在线程等待，才创建连接
                        if (poolingCount >= notEmptyWaitThreadCount //
                                && (!(keepAlive && activeCount + poolingCount < minIdle))
                                && !isFailContinuous()
                        ) {
                            empty.await();
                        }

                        // 防止创建超过maxActive数量的连接
                        if (activeCount + poolingCount >= maxActive) {
                            empty.await();
                            continue;
                        }
                    }

```
必须存在线程等待获取连接时候，才能创建连接，并且要保持总的连接数，不能超过配置的最大连接。
创建完连接之后，执行notEmpty.signalAll(),通知消费者。

## 获取连接

```
poolableConnection = getConnectionInternal(maxWaitMillis);
```

1.连接不足，需要直接去创建新的连接
2.从connections里面取连接
```
                if (maxWait > 0) {
                    holder = pollLast(nanos);
                } else {
                    holder = takeLast();
                }
				
```

## 连接回收

druid-master\src\main\java\com\alibaba\druid\pool\DruidDataSource.java

```
    public class DestroyTask implements Runnable {
        public DestroyTask() {

        }

        @Override
        public void run() {
            shrink(true, keepAlive);

            if (isRemoveAbandoned()) {
                removeAbandoned();
            }
        }

    }


```
执行shrink
```
    public void shrink(boolean checkTime, boolean keepAlive) {
```

druid连接池源码里面有其他工具，比如数据库驱动工具，jdbc工具，解析SQL的语法树，ibatis的支持，wall过滤，多数据源…
