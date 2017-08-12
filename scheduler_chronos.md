
```text
JobScheduler.scala:[作业调度启动]  
  JobScheduler.startUp 启动代码进行zk选主  
    JobScheduler.onElected zk选主成功后进入主循环  
      JobScheduler.mainLoop 进行作业调度  
        如果running[运行标记]被外界重置[例如shutdown事件]，则中断  
        进行每一轮等待:  
          如果第一次加载作业，不等待  
          如果有事件通知[比如新增作业], 则不等待  
          否则等待至下一轮剩余作业的开始时间为止  
          如果没有剩余作业，则默认等待1分钟  
        JobUtils.loadJobs 从存储层获取作业列表  
          默认为MesosStatePersistenceStore, 底层为Zookeeper  
        JobScheduler.registerJobs  
          将作业分为Scheduled作业跟Dependency作业  
          对Scheduled作业按时间进行排序  
          将Scheduled作业加入DAG的结点  
          将Dependency作业加入DAG的结点  
          如果Dependency作业所引用的父作业不存在，则进行删除  
        JobScheduler.getJobsToRun 将作业分为即将运行及不被运行两类  
          Iso8601Expressions.parse 对作业信息进行解析  
          如果作业未禁用, 且剩余重复次数不为0, 且调度时间小于当前时间则进行调度  
        JobScheduler.runJobs  
          TaskManager.enqueue 将即将运行的作业按优先级加入任务管理器中  
        JobScheduler.nanosUntilNextJob 计算下一次等待时间  
          找到不被禁用且不被调度运行的下一个最早的作业，得到等待周期  
          如果作业全被禁用，则默认为1分钟  
 ```         
        
        
