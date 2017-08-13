```text
Main.scala:[程序入口]
  modules:
    ChronosRestModule: HttpModule + MetricsModule[@chaos web framework]
      Iso8601JobResource
      DependentJobResource
      JobManagementResource
      TaskManagementResource
      GraphManagementResource
      StatsResource
    MainModule:
     ZookeeperModule[Zookeeper]
     JobMetricsModule[Graphite]
     JobStatsModule[Cassandra]


MainModule.scala:[组件初始化]
  provideTaskScheduler -> JobScheduler
    taskManager: TaskManager
      provideListeningExecutorService -> java.util.concurrent.ScheduledThreadPoolExecutor
      provideOfferReviver -> MesosOfferReviverDelegate
        provideOfferReviverActor -> MesosOfferReviverActor[@actorSystem]
    jobGraph: JobGraph
    persistenceStore: ZookeeperModule.provideStore -> MesosStatePersistenceStore[Zookeeper]
    mesosDriver: provideMesosSchedulerDriverFactory -> MesosDriverFactory
      MesosJobFramework -> Scheduler
    jobsObserver: provideJobsObservers -> JobStats + JobNotificationObserver
      JobStatsModule.provideCassandraClusterBuilder -> Cassandra:com.datastax.driver.core.Cluster
      provideNotificationClients -> MailClient + RavenClient + SlackClient + MattermostClient + HttpClient
    jobMetrics: JobMetricsModule.provideMetricReporterService -> MetricReporterService[com.codahale.metrics.graphite.Graphite]
    actorSystem: akka.actor.ActorSystem


JobScheduler.scala:[作业调度启动]  
  JobScheduler.startUp 启动代码进行zk选主  
    JobScheduler.onElected zk选主成功后进入主循环  
      JobScheduler.mainLoop 进行作业调度  
        如果running[运行标记]被外界重置[例如shutdown事件]，则中断  
        ReentrantLock.newCondition: 进行每一轮等待:
          如果第一次加载作业，不等待  
          否则等待至最近一个未调度作业的开始时间为止  
          如果没有剩余作业，则默认等待1分钟  
          *如果有事件通知[比如新增作业], 取消等待  
        JobUtils.loadJobs 从存储层获取作业列表  
          默认为MesosStatePersistenceStore, 底层为Zookeeper  
        JobScheduler.registerJobs  
          将作业分为Scheduled作业跟Dependency作业  
          对Scheduled作业按时间进行排序  
          将Scheduled作业加入JobGraph的结点  
          将Dependency作业加入JobGraph的结点  
          如果Dependency作业所引用的父作业不存在，则进行删除  
        JobScheduler.getJobsToRun 将作业分为即将运行及不被运行两类  
          Iso8601Expressions.parse 对作业信息进行解析  
          如果作业未禁用, 且剩余重复次数不为0, 且调度时间小于当前时间则进行调度  
        JobScheduler.runJobs  
          TaskUtils.getTaskId 
            新建作业的任务实例[名称为"ct:%d:%d:%s:%s"], 当前时间微秒, 尝试次数, 作业名称及作业参数
            并初始化状态为TASK_STAGING
          TaskManager.enqueue 将即将运行的任务按优先级加入任务管理器中  
        JobScheduler.nanosUntilNextJob 计算下一次等待时间  
          找到不被禁用且不被调度运行的下一个最早的作业，得到等待周期  
          如果作业全被禁用，则默认为1分钟  


MesosStatePersistenceStore.scala: [作业存储]
1. 作业创建
  Jobs.scala 作业格式: 分为Schedule作业及DependencyBasedJob作业
    Schedule作业: 包含额外的schedule[String]字段
    DependencyBasedJob作业: 包含额外的parents[Set]字段
  MesosStatePersistenceStore.persistJob 
  将作业数据转化json序列化后存入对应的zk节点
2. 作业删除
  从zk中查找节点并删除
3. 作业遍历
  从zk中找到目录下的所有作业节点，并取出对应的数据
4. 作业查找
  从zk中找到对应的节点，并取出对应的数据
5. 作业任务存储
  将作业正在运行的任务实例放至作业相应的节点下


JobGraph.scala: [作业注册]


Iso8601Expressions.scala: [作业调度解析]
  通过正则表达式"(R[0-9]*)/(.*)/(P.*)"得到重复次数, 开始日期, 调度周期
  如果重次数未设置, 则为-1(不限次数)
  如果开始日期未设置, 则为当前时间-1秒, 使用joda日期库进行解析


TaskManager.scala: [任务调度]
  检查队列中是否已经包括了该任务
  将对应的作业加入作业队列中
  在相应的优先级队列中加入该任务
  从JobGraph中查找节点信息, 如找不到则警告作业未注册
  JobsObserver:  发送job事件至Observer[JobQueued]
    JobNotificationObserver + JobStats
    JobNotificationObserver
      当作业去除，禁用，失败，成功时发送邮件
  MesosOfferReviverDelegate.reviveOffers: 请求Mesos资源
    MesosOfferReviverActor.receive
      MesosOfferReviverActor.reviveOffers


JobStats.scala[作业状态存到Cassandra]
  JobStats.jobQueued [作业id, 任务id, 尝试次数]
    JobStats.updateJobState 更新作业状态, 默认为CurrentState.queued
      如果作业历史没有running，且当前作业状态为CurrentState.queued则更新作业状态 
      如果作业尝试次数不为0, 则更新作业状态设为"%attemp running"
```
