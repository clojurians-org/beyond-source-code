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
      CoreContainer.load 开始加载core
        ZkContainer.initZooKeeper 初始化zookeeper节点
          ZkController/checkChrootPath 检查solr chroot节点是否存在
            SolrZkClient.makePath 按需要创建chroot节点
          ZkController[new]
            ZkCmdExecutor[new]
            ZkStateReader[new]
            SolrZkClient.onReconnect 建立重连机制
            ZkController.init
              ZkCmdExecutor.ensureExists 
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
          如果存在confDir
            ZkConfigManager.uploadConfigDir
                
      
        CoresLocator[new]
        UpdateShardHandler[new]
        coreContainerWorkExecutor 建立coreLoadExecutor线程池
        CoresLocator.discover 从本地找到core.properties文件创造CoreDescriptor

```
