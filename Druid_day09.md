# Druid Segment Balance

## Balance 相关的配置
```
druid.coordinator.startDelay=PT30S
druid.coordinator.period=PT30S 调度的时间
druid.coordinator.balancer.strategy=cost 默认

动态配置：
maxSegmentsToMove = 5 ##每次Balance最多移动多少个Segment

```
## DruidCoordinatorBalancer 类
```
@Override
public DruidCoordinatorRuntimeParams run(DruidCoordinatorRuntimeParams params)
{
  final CoordinatorStats stats = new CoordinatorStats();
  // 不同tier层的分开Balance
  params.getDruidCluster().getHistoricals().forEach((String tier, NavigableSet<ServerHolder> servers) -> {
   balanceTier(params, tier, servers, stats);
  });
  return params.buildFromExisting().withCoordinatorStats(stats).build();
}

```

##  balanceTier 方法
```
private void balanceTier(DruidCoordinatorRuntimeParams params, String tier, SortedSet<ServerHolder> servers,CoordinatorStats stats){
  final BalancerStrategy strategy = params.getBalancerStrategy();
  final int maxSegmentsToMove = params.getCoordinatorDynamicConfig().getMaxSegmentsToMove();

  currentlyMovingSegments.computeIfAbsent(tier, t -> new ConcurrentHashMap<>());

  final List<ServerHolder> serverHolderList = Lists.newArrayList(servers);

  //集群中只有一个 Historical 节点时不进行Balance
  if (serverHolderList.size() <= 1) {
   log.info("[%s]: One or fewer servers found. Cannot balance.", tier);
   return;
  }

  int numSegments = 0;
  for (ServerHolder server : serverHolderList) {
   numSegments += server.getServer().getSegments().size();
  }

  if (numSegments == 0) {
   log.info("No segments found. Cannot balance.");
   return;
  }
  long unmoved = 0L;
  for (int iter = 0; iter < maxSegmentsToMove; iter++) {
   //通过随机算法选择一个候选Segment，该Segment会参与后面的Cost计算
   final BalancerSegmentHolder segmentToMove = strategy.pickSegmentToMove(serverHolderList);

   if (segmentToMove != null && params.getAvailableSegments().contains(segmentToMove.getSegment())) {
     //找Cost最小的节点，Cost计算入口
    final ServerHolder holder = strategy.findNewSegmentHomeBalancer(segmentToMove.getSegment(), serverHolderList);
    //找到候选节点，发起一次Move Segment的任务
    if (holder != null) {
     moveSegment(segmentToMove, holder.getServer(), params);
    } else {
     ++unmoved;
    }
   }
  }
  ......
}

```
##  Reservoir 随机算法
```
public class ReservoirSegmentSampler
{

 public BalancerSegmentHolder getRandomBalancerSegmentHolder(final List<ServerHolder> serverHolders)
 {
  final Random rand = new Random();
  ServerHolder fromServerHolder = null;
  DataSegment proposalSegment = null;
  int numSoFar = 0;

  //遍历所有List上的Historical节点
  for (ServerHolder server : serverHolders) {
   //遍历一个Historical节点上所有的Segment
   for (DataSegment segment : server.getServer().getSegments().values()) {
    int randNum = rand.nextInt(numSoFar + 1);
    // w.p. 1 / (numSoFar+1), swap out the server and segment
    // 随机选出一个Segment，后面的会覆盖前面选中的，以最后一个被选中为止。
    if (randNum == numSoFar) {
     fromServerHolder = server;
     proposalSegment = segment;
    }
    numSoFar++;
   }
  }
  if (fromServerHolder != null) {
   return new BalancerSegmentHolder(fromServerHolder.getServer(), proposalSegment);
  } else {
   return null;
  }
 }
}

@Override
public ServerHolder findNewSegmentHomeBalancer(DataSegment proposalSegment, List<ServerHolder> serverHolders){
  return chooseBestServer(proposalSegment, serverHolders, true).rhs;
}

protected Pair<Double, ServerHolder> chooseBestServer(
 final DataSegment proposalSegment,
 final Iterable<ServerHolder> serverHolders,
 final boolean includeCurrentServer
){
  Pair<Double, ServerHolder> bestServer = Pair.of(Double.POSITIVE_INFINITY, null);

  List<ListenableFuture<Pair<Double, ServerHolder>>> futures = Lists.newArrayList();

  for (final ServerHolder server : serverHolders) {
   futures.add(
     exec.submit(
       new Callable<Pair<Double, ServerHolder>>()
       {
        @Override
        public Pair<Double, ServerHolder> call() throws Exception
        {
         //计算Cost：候选Segment和Historical节点上所有Segment的cost和
         return Pair.of(computeCost(proposalSegment, server, includeCurrentServer), server);
        }
       }
     )
   );
  }

  final ListenableFuture<List<Pair<Double, ServerHolder>>> resultsFuture = Futures.allAsList(futures);
  final List<Pair<Double, ServerHolder>> bestServers = new ArrayList<>();
  bestServers.add(bestServer);
  try {
   for (Pair<Double, ServerHolder> server : resultsFuture.get()) {
    if (server.lhs <= bestServers.get(0).lhs) {
     if (server.lhs < bestServers.get(0).lhs) {
      bestServers.clear();
     }
     bestServers.add(server);
    }
   }

   //Cost最小的如果有多个，随机选择一个
   bestServer = bestServers.get(ThreadLocalRandom.current().nextInt(bestServers.size()));
  }
  catch (Exception e) {
   log.makeAlert(e, "Cost Balancer Multithread strategy wasn't able to complete cost computation.").emit();
  }
 return bestServer;
}

protected double computeCost(final DataSegment proposalSegment, final ServerHolder server,final boolean includeCurrentServer){
  final long proposalSegmentSize = proposalSegment.getSize();

  // (optional) Don't include server if it is already serving segment
  if (!includeCurrentServer && server.isServingSegment(proposalSegment)) {
   return Double.POSITIVE_INFINITY;
  }

  // Don't calculate cost if the server doesn't have enough space or is loading the segment
  if (proposalSegmentSize > server.getAvailableSize() || server.isLoadingSegment(proposalSegment)) {
   return Double.POSITIVE_INFINITY;
  }

  // 初始cost为0
  double cost = 0d;

  //计算Cost：候选Segment和Historical节点上所有Segment的totalCost
  cost += computeJointSegmentsCost(
    proposalSegment,
    Iterables.filter(
      server.getServer().getSegments().values(),
      Predicates.not(Predicates.equalTo(proposalSegment))
    )
  );

  // 需要加上和即将被加载的Segment之间的cost
  cost += computeJointSegmentsCost(proposalSegment, server.getPeon().getSegmentsToLoad());

  // 需要减掉和即将被加载的 Segment 之间的 cost
  cost -= computeJointSegmentsCost (proposalSegment, server.getPeon().getSegmentsMarkedToDrop());

  return cost;
}


static double computeJointSegmentsCost(final DataSegment segment, final Iterable<DataSegment> segmentSet){
  double totalCost = 0;
  // 此处需要注意，当新增的Historical节点第一次上线的时候，segmentSet应该是空，所以totalCost=0最小
  // 新增节点总会很快的被均衡
  for (DataSegment s : segmentSet) {
   totalCost += computeJointSegmentsCost(segment, s);
  }
  return totalCost;
}

public static double intervalCost(double x1, double y0, double y1){
  if (x1 == 0 || y1 == y0) {
   return 0;
  }

  // 保证Segment A开始时间小于B的开始时间
  if (y0 < 0) {
   // swap X and Y
   double tmp = x1;
   x1 = y1 - y0;
   y1 = tmp - y0;
   y0 = -y0;
  }

  if (y0 < x1) {
   // Segment A和B 时间有重叠的情况，这个分支暂时不分析
   .......
  } else {
   // 此处就是计算A和B两个Segment之间的cost，代价计算函数：See https://github.com/druid-io/druid/pull/2972
   final double exy0 = FastMath.exp(x1 - y0);
   final double exy1 = FastMath.exp(x1 - y1);
   final double ey0 = FastMath.exp(0f - y0);
   final double ey1 = FastMath.exp(0f - y1);

   return (ey1 - ey0) - (exy1 - exy0);
  }
}

```




