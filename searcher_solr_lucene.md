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
            [*] SolrCore[new]
            registerCore[registerInZk=false]
          ZkContainer.registerInZk [background=true, skipRecovery=false] 
            ZkController.register{CoreDescriptor, recoverReloadedCores=false, afterExpiration=false, skipRecovery=false}

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
  ZkController.preRegister shard相关
    checkStateInZk 检查zk节点是否存在Replica
    publish{CoreDescriptor, Replica.State.DOWN, updateLastState=false, forcePublish=true} 
      OverseerJobQueue.offer 将Replica状态置为DOWN
    ZkStateReader.registerCore 注册StateWatcher监听
    doGetShardIdAndNodeNameProcess
      waitForShardId 同步zk shardId到本地
  ZkController.register collection相关
    joinElection{CloudDescriptor, afterExpiration=false, joinAtHead=false}
      LeaderElector.joinElection{ElectionContext$ShardLeaderElectionContext}
        创建选举EPHEMERAL_SEQUENTIAL节点路径:ElectionContext$ShardLeaderElectionContext.electionPath
        ZkController.checkIfIamLeader
          {isLeader -> runIamLeaderProcess
             ElectionContext$ShardLeaderElectionContext.runLeaderProcess
             确保/collections/{collection}/leader存在
             创建leaderPath节点, 写入leaderProps
             Overseer.getStateUpdateQueue更新Replica为Active
           :else -> ElectionWatcher 
             watch leader节点
          }
        getLeader
        ...

SolrCore.java
  SolrCore.ctor
    CoreContainer.SolrCores 增加当前CoreDescriptor
    initDirectoryFactory
      HdfsDirectoryFactory.init
    initRecoveryStrategyBuilder
    DefaultSolrCoreState[new]
    initSchema
      IndexSchemaFactory.buildIndexSchema
        ManagedIndexSchemaFactory.create
          加载manage_schema文件
          ManagedIndexSchema[new]
    initListeners
      增加firstSearcher及newSearcher监听器
    initIndex
      StandardIndexReaderFactory[new]
      如果索引目录不存在则新建索引目录
    initWriters
    loadSearchComponents 加载SpellCheckComponentHighlightComponent,SuggestComponent 等SearchComponent
      <searchComponent class="solr.SuggestComponent" name="analyzing_infix_suggester_random_short_dictionary">
      </searchComponent>
    loadUpdateProcessorChains 处理更新请求[默认为LogUpdateProcessorFactory, DistributedUpdateProcessorFactory, RunUpdateProcessorFactory]
      LogUpdateProcessorFactory -> RunUpdateProcessorFactory$LogUpdateProcessor, 
      DistributedUpdateProcessorFactory -> DistributedUpdateProcessor
      RunUpdateProcessorFactory -> RunUpdateProcessorFactory$RunUpdateProcessorFactory
      最终由RunUpdateProcessorFactory$RunUpdateProcessorFactory分布式提交给各个UpdateHandler[DirectUpdateHandler2]进行实际处理
        <updateRequestProcessorChain name="uima">... </updateRequestProcessor>
    RequestHandlers/initHandlersFromConfig 加载RequestHandlers[SearchHandler, ReplicationHandler]
      <requestHandler name="/select" class="solr.SearchHandler"/>
      <requestHandler name="/replication" class="solr.ReplicationHandler" startup="lazy" />
    initStatsCache 初始化统计缓存
    initUpdateHandler 加载UpdateHandler
      <updateHandler class="solr.DirectUpdateHandler2"> ... </updateHandler>
      [*] DirectUpdateHandler2[new]
    initSearcher [使用searcherExecutor阻塞，在SeedVersionBuckets前进行异步调用]
      getSearcher{forceNew=false, returnSearcher=false, waitSearcher=null, updateHandlerReopens=true}
        openNewSearcher{updateHandlerReopens=true, realtime=false}
          DefaultSolrCoreState.getIndexWriter{core=this} 
            DefaultSolrCoreState.createMainIndexWriter{SolrCore}
              SolrIndexWriter.create
                SolrIndexWriter[new]
          DirectoryReader[new]: StandardIndexReaderFactory.newReader{IndexWriter}
            org.apache.lucene.index.DirectoryReader.openl{IndexWriter}
          SolrIndexSearcher[new]{SolrCore, DirectoryReader} 打开IndexSeacher
        firstSearcherListeners.newSearcher 触发firstSearcher事件
        registerSearcher
    initRestManager
      RestManager[new]
      RestManager.init
    seedVersionBuckets
      UpdateLog.seedBucketsWithHighestVersion 得到updateLog及index的最大version
        UpdateLog.seedBucketsWithHighestVersion
          UpdateLog.getMaxRecentVersion,
          VersionInfo.getMaxVersionFromIndex{Searcher}
            VersionInfo.getMaxVersionFromIndexedPoints
          将所有bucket最大版本置为最大值
    bufferUpdatesIfConstructing
      UpdateLog.bufferUpdates 如果状态为CONSTRUCTION
        versionInfo.blockUpdates
    registerConfListener

DirectUpdateHandler2.java
  DirectUpdateHandler2.ctor
    DirectUpdateHandler2.super{SolrCore}
      UpdateHandler.{SolrCore,UpdateLog=null}
        HdfsUpdateLog[new]
        HdfsUpdateLog.init{PluginInfo}
        HdfsUpdateLog.init{UpdateHandler, SolrCore}
          如果不存在tlog目录，则创建
          getLogList 得到所有的日志文件并加入logs列表中[最老在前面]
          从logs取出最早的两个日志加入newestLogsOnStartup中
          VersionInfo[new]{HdfsUpdateLog}
            IndexSchema.readSchema
            getAndCheckVersionField 验证version字段
          getRecentUpdates  如果存在,则加入tlog, prevTlog到LogList[最新在前面]
            UpdateLog.RecentUpdates[new]
              UpdateLog.update
                HdfsTransactionLog.getReverseReader
                  HDFSReverseReader[new]
                读取LogList里面的UpdateLog操作至updateList以及version至updates[version:update]
            添加操作记录至oldDeletes及deleteByQueries
    CommitTracker[new]:Hard
    CommitTracker[new]:Soft


SolrIndexWriter.java
  SolrIndexWriter.ctor 
    注册metrics
  
SolrIndexSearcher.java
  SolrIndexSearcher.ctor
    初始化cache
  SolrIndexSearcher.register
    设置registerTime为当前时间

```
