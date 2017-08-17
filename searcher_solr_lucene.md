```text
web.xml web启动入口
  <filter>
    <filter-name>SolrRequestFilter</filter-name>
    <filter-class>org.apache.solr.servlet.SolrDispatchFilter</filter-class>
  </filter>

SolrDispatchFilter.java 
  SolrDispatchFilter.init solr项目初始化
    SolrDispatchFilter.createCoreContainer 创建CoreContainer
      CoreContainer[new]
        CorePropertiesLocator[new]
      CoreContainer.load 开始加载core
        coreContainerWorkExecutor 建立coreLoadExecutor线程池
        UpdateShardHandler[new]
        ZkContainer.initZooKeeper 初始化zookeeper节点
          ZkController/checkChrootPath 检查solr chroot节点是否存在
            SolrZkClient.makePath 按需要创建chroot节点
          ZkController[new]
            ZkCmdExecutor[new]
            ZkStateReader[new]
            SolrZkClient.onReconnect 建立重连机制
            ZkController.init
          如果存在confDir
            ZkConfigManager.uploadConfigDir
        创建{/admin/zookeeper, /admin/collections, /admin/info, /admin/cores, /admin/configs, 
             /admin/metrics, /admin/metrics/collector}Handler
        ConfigSetService/createConfigSetService
          CloudConfigSetService[new]
        CoresLocator.discover 从本地找到core.properties文件创造CoreDescriptor
        更新状态为CORE_DISCOVERY_COMPLETE
        对于每个CoreDescriptor     
          CoreContainer.createFromDescriptor
            ZkController.preRegister
            SolrCore[new]
            registerCore[registerInZk=false]
          ZkContainer.registerInZk [background=true, skipRecovery=false] 

ZkController.java
  ZkController.init
    ZkController.createClusterZkNodes -> ZkCmdExecutor.ensureExists
      确保{/live_nodes, /collections, /aliases.json, /clusterstate.json, /security.json, /autoscaling.json}存在
      检查/configs/_default是否存在
    ZkStateReader.createClusterStateWatchersAndUpdate 从zookeeper更新集群信息
      ZkStateReader.loadClusterProperties
        SolrZkClient.getData 加载/clusterstate.json
        ZkStateReader.clusterPropertiesWatcher 增加watcher进行loadClusterProperties
      ZkStateReader.refreshLiveNodes 加载/live_nodes, 添加ZkStateReader.LiveNodeWatcher
      ZkStateReader.refreshLegacyClusterState 添加ZkStateReader.LegacyClusterStateWatcher
      ZkStatereader.refreshStateFormat2Collections 添加ZkStateReader.StateWatcher
      ZkStatereader.refreshCollectionList 添加ZkStateReader.CollectionsChildWatcher
      ZkStatereader.constructState
      SolrZkClient.exists 添加/aliases.json Watcher
      SolrZkClient.updateAliases 加载别名信息
    LeaderElector.joinElection[OverseerElectionContext] 进行overseer选举
    publishAndWaitForDownStates 将所有节点置为down状态
    在/live_nodesg下创建节点名
  ZkController.preRegister
  ZkController.register

```
