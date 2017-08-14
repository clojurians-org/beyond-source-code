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
        CoresLocator[new]
          ZkContainer.initZooKeeper 初始化zookeeper节点
      CoreContainer.load 开始加载core
        UpdateShardHandler[new]
        coreContainerWorkExecutor 建立coreLoadExecutor线程池
        CoresLocator.discover 从本地找到core.properties文件创造CoreDescriptor

```
