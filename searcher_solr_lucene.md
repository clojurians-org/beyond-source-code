```text
线程关系:
  <<coreZkRegister>> (ZkContainer.registerInZk => ZkController.register)
    # ZkController.register -> SolrCore.getUpdateHandler().getUpdateLog().recoverFromLog()
    <<recoveryExecutor>> (UpdateLog.recoverFromLog => LogReplayer)
    # ZkController.register -> checkRecovery -> SolrCore.getUpdateHandler().getSolrCoreState().doRecovery
    <<UpdateExecutor>> (DefaultSolrCoreState.doRecovery -> recoveryExecutor.submit)
      <<recoveryExecutor>> (DefaultSolrCoreState.doRecovery => RecoveryStrategy)
        # RecoveryStrategy.run -> RecoveryStrategy.doRecovery -> RecoveryStrategy.doSyncOrReplicateRecovery
        # -> RecoveryStrategy.replay -> UpdateLog.applyBufferedUpdates
        <<recoveryExecutor>> (UpdateLog.applyBufferedUpdates -> LogReplayer)

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
        HttpShardHandlerFactory[new]
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
            <<coreZkRegister>>
              ZkController.register{CoreDescriptor, recoverReloadedCores=false, afterExpiration=false, skipRecovery=false}
  SolrDispatcherFilter.doFilter
    HttpSolrCall[new]{CoreContainer}
    HttpSolrCall.call
      HttpSolrCall.init
        CoreContainer.getRequestHandler
          SolrRequestHandler[new]: RequestHandlerBase.getRequestHandler
            SearchHandler
            UpdateRequestHandler@ContentStreamHandlerBase
            ReplicationHandler
            CoreAdminHandler
            ReloadCacheRequestHandler
      HttpSolrCall.execute
        SolrCore.execute(SolrRequestHandler)
          SolrRequestHandler.handleRequest
            SolrRequestHandler.handleRequestBody
    

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
    getLeader{600000} 使用zk信息, 防止过期
      getLeaderProps.getCoreUrl
    ZkController.checkRecovery
      DefaultSolrCoreState.doRecovery
        cancelRecovery
          RecoveryStrategy.close
            LOG.warn("Stopping recovery for core=[{}] coreNodeName=[{}]", coreName, coreZkNodeName)
        RecoveryStrategy[+new]
        <<UpdateExecutor>>.submit{"updateExecutor"} 
          <<RecoveryExecutor>>.submit{"recoveryExecutor""} 使用线程池提交RecoveryStrategy任务
      getLeaderInitiatedRecoveryState
    public{Replica.State.ACTIVE}

RecoveryStrategy.java

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
      最终由RunUpdateProcessorFactory$RunUpdateProcessor分布式提交给各个UpdateHandler[DirectUpdateHandler2]进行实际处理
        <updateRequestProcessorChain name="uima">... </updateRequestProcessor>
    RequestHandlers/initHandlersFromConfig 加载RequestHandlers[SearchHandler, ReplicationHandler]
      <requestHandler name="/select" class="solr.SearchHandler"/>
      <requestHandler name="/replication" class="solr.ReplicationHandler" startup="lazy" />
    initStatsCache 初始化统计缓存
    initUpdateHandler 加载UpdateHandler
      <updateHandler class="solr.DirectUpdateHandler2"> ... </updateHandler>
      [*] DirectUpdateHandler2[new]
    initSearcher [使用<<searcherExecutor>>阻塞，在SeedVersionBuckets前进行异步调用]
      getSearcher{forceNew=false, returnSearcher=false, waitSearcher=null, updateHandlerReopens=true}
        openNewSearcher{updateHandlerReopens=true, realtime=false}
          DefaultSolrCoreState.getIndexWriter{core=this} 
            DefaultSolrCoreState.createMainIndexWriter{SolrCore}
              SolrIndexWriter.create
                SolrIndexWriter[new]
          DirectoryReader[new]: StandardIndexReaderFactory.newReader{IndexWriter}
            org.apache.lucene.index.DirectoryReader.open{IndexWriter}
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

SolrIndexWriter.java
  SolrIndexWriter.ctor 
    注册metrics
  
SolrIndexSearcher.java
  SolrIndexSearcher.ctor
    初始化cache
  SolrIndexSearcher.register
    设置registerTime为当前时间

SearchHandler.java [:debugQuery :debug :distrib :shards.info]
  SearchHandler.handleRequestBody
    SearchHandler.getComponents
      SearchHandler.initComponents
        ${components}: List<SearchComponent> 
    SearchHandler.getAndPrepShardHandler
      {distrib -> HttpShardHandler, else -> nil}
    ${components}*.prepare
    {distrib -> 
      按Stage过程循环
        ${components}*.distributedProcess  进行分布式处理后得到已完成stage的下一stage
          QueryComponent.distributedProcess
        HttpShardHandler.submit 
            提交给所有shard
            {去除[shards, indent, echoParams], distrib=false, isShard=true, shard.url, shards.qt=, qt=}
        等待所有请求
        如果SHARDS_TOLERANT不为true, 出现异常, 则进行取消操作
        HttpShardHandler.takeCompletedOrError
        ${components}*.processResponse
      ${components}*.finishStage
     else -> ${components}*.process

QueryComponent.java
  QueryComponent.distributedProcess
    QueryComponent.regularDistributedProcess
      STAGE_PARSE_QUERY -> createDistributedStats 如果包含GET_SCORES标记或者以SCORE排名
      STAGE_EXECUTE_QUERY -> createMainQuery [distrib.singlePass]
      STAGE_GET_FIELDS -> createRetrieveDocs 
      STAGE_DONE
  QueryComponent.handleResponses
    QueryComponent.handleRegularResponses
      PURPOSE_GET_TOP_IDS -> mergeIds
      PURPOSE_GET_TERM_STATS -> updateStats
      PURPOSE_GET_FIELDS -> returnFields 
  QueryComponent.finishStage

UpdateRequestHandler.java
  ContentStreamHandlerBase.handleRequestBody
    UpdateRequestProcessorChain.createProcessor
      {LogUpdateProcessorFactory -> LogUpdateProcessorFactory$LogUpdateProcessor,
       DistributedUpdateProcessorFactory -> DistributedUpdateProcessor
       RunUpdateProcessorFactory -> RunUpdateProcessorFactory$RunUpdateProcessor} 
    UpdateRequestHandler.ContentStreamLoader[new]
    UpdateRequestHandler.ContentStreamLoader.load
      JavabinLoader.load{RunUpdateProcessorFactory$RunUpdateProcessor}
        JavabinLoader.parseAndLoadDocs
          ~JavaBinUpdateRequestCodec$StreamingUpdateHandler[new]
          JavaBinUpdateRequestCodec.unmarshal{JavaBinUpdateRequestCodec$StreamingUpdateHandler}
            JavaBinUpdateRequestCodec$JavaBinCodec.unmarshal -> ${docMap}
              JavaBinCodec.readVal
              JavaBinUpdateRequestCodec$JavaBinCodec.readOuterMostDocIterator
                ~JavaBinUpdateRequestCodec$StreamingUpdateHandler.update
                  AddUpdateCommand{:overwrite ~> true, :commitWithin ~> -1} 
                  RunUpdateProcessorFactory$RunUpdateProcessor.processAdd
            updateRequest.deleteById 如果delById
            updateRequest.deleteByIdMap 如果delByIdMap
            updateRequest.delByQ 如果delByQ
          JavabinLoader.delete
            RunUpdateProcessorFactory$RunUpdateProcessor.processDelete
    RunUpdateProcessorFactory$RunUpdateProcessor.processCommit 如果[:optimize :commit :softCommit :prepareCommit]
      [:openSearcher :waitSearcher :softCommit :expungeDeletes :maxSegments :prepareCommit]
    RunUpdateProcessorFactory$RunUpdateProcessor.processRollback 如果[:rollback]

RunUpdateProcessorFactory$RunUpdateProcessor.java
  RunUpdateProcessorFactory$RunUpdateProcessor.update

DistributedUpdateProcessor.java
  DistributedUpdateProcessor.processAdd
    setupRequest
      如果是REPLAY或者PEER_SYNC {:isLeader->false, :forwardToLeader-false} 不需要leader逻辑, 返回
      DocCollection.getRouter.getTargetSlice 通过路由找到目标Shard所有主机, 没有则抛出异常
      update.distrib 为DistribPhase.FROMLEADER且自己不为leader，则可能有问题, 继续往下处理
        否则不需要leader逻辑{:isLeader=false, :forwardToLeader=false}
      如果distrib.from及distrib.from.parent不为空, 且slice不为active, 则报错
      如果父slice 不包含子slice, 则报错
      如果fromLeader, 由distrib.from.collection路由而来, 但自己为leaer, 则报错
      {fromLeader -> [:forwardToLeader=false]
       isLeader -> 加入replica到${nodes}
       else -> [:forwardToLeader=true], 加入leader到${nodes}
      }
    DistributedUpdateProcessor.versionAdd
      AddUpdateCommand.getIndexedId
      确保UniqueKeyField字段有且仅有一个
      确保UpdateLog读取的${vinfo} 存在
      如果是原子更新
        getUpdatedDocument
          RealTimeGetComponent.getInputDocument
            getInputDocumentFromTlog
              UpdateLog.lookup
            SolrDocumentFetcher.doc
          AtomicUpdateDocumentMerger.merge
        UpdateLog.add | 如果UpdateLog不是Active状态
      DistributedUpdateProcessor.doLocalAdd
        UpdateRequestProcessor.processAdd
          RunUpdateProcessorFactory$RunUpdateProcessor.processAdd
            DirectUpdateHandler2.addDoc
    SolrCmdDistributor.distribAdd 如果${nodes}不为空, {:distrib.from=}
      SolrCmdDistributor.submit 遍历${nodes}提交
        <<updateExecutor>> -> doRequest
          ErrorReportingConcurrentUpdateSolrClient.request@StreamingSolrClients

HdfsUpdateLog.java
  HdfsUpdateLog.ctor
    HdfsTransactionLog
      org.apache.hadoop.hdfs.DistributedFileSystem.append
      ${fos}: FastOutputStream[new] {org.apache.hadoop.hdfs.client.HdfsDataOutputStream}
  UpdateLog.add
    如果不是UpdateCommand.REPLAY
      HdfsTransactionLog[new] 如果不存在则新建
      TransactionLog.write
        checkWriteHeader
        FastOutputStream.write
          flush | 如果buf满了, 否则存在buf
            org.apache.hadoop.hdfs.client.HdfsDataOutputStream.write

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
  DirectUpdateHandler2.addDoc
    DirectUpdateHandler2.addDoc0
      DirectUpdateHandler2.doNormalUpdate
        ${updateTerm} <- idTerm | 如果updateTerm为空
        DirectUpdateHandler2.updateDocOrDocValues{IndexWriter}
          DirectUpdateHandler2.updateDocument
            {block -> writer.updateDocuments{${updateTerm}, AddUpdateCommand@Iterable<Document>}
             else -> writer.updateDocument{${updateTerm}, Document} 
          UpdateLog.add 操作成功后, 写入tlog


IndexWriter.java
  IndexWriter.updateDocuments{AddUpdateCommand}
    AddUpdateCommand.iterator
      AddUpdateCommand.flatten
        AddUpdateCommand.recUnwrapp 递归增加所有子文档
    DocumentsWriter.updateDocuments
      DocumentsWriterPerThread.updateDocuments
        DefaultIndexingChain.processDocument 循环处理
        DocumentsWriterDeleteQueue.add
  IndexWriter.updateDocument
    DocumentsWriter.updateDocument
  IndexWriter.deleteDocuments
    DocumentsWriter.deleteTerms
  IndexWriter.commit
    IndexWriter.commitInternal
      IndexWriter.prepareCommitInternal
        DocumentsWriter.flushAllThreads
          DocumentsWriterFlushControl.markForFullFlush
            ${fullFlush} <- true
            deleteQueue <- new DocumentsWriterDeleteQueue{generation++, seqNo}
              ; seqNo <- ${deleteQueue}.getLastSequenceNumber + 2 * perThreadPool.getActiveThreadStateCount + 2
            对于每个threadState
              perThread.flushPending <- true
              internalTryCheckOutForFlush
                增加至${flushingWriters}
        IndexWriter.processEvents{:triggerMerge=false, :forcePurge=false}
        IndexWriter.applyAllDeletesAndUpdates
        DocumentsWriter.finishFullFlush
        ${maybeMerge} <- true
        IndexWriter.startCommit
      IndexWriter.maybeMerge
      IndexWriter.finishCommit

DocumentsWriter.java
  DocumentsWriter.updateDocument
    <<DocumentsWriterPerThreadPool>>:DocumentsWriterPerThread.updateDocument
      DefaultIndexingChain.processDocument
      DocumentsWriterPerThread.finishDocument
        DocumentsWriterDeleteQueue.add
          增加到当前slice链表中
          DocumentsWriterDeleteQueue.tryApplyGlobalSlice 添加到${globalSlice}
    DocumentsWriterFlushControl.doAfterDocument{DocumentsWriterPerThread, isUpdate=true}
      DocumentsWriterFlushControl.commitPerThreadBytes 更新ThreadState消耗字节信息
      ThreadState不为flushPending状态
        FlushByRamOrCountsPolicy.onUpdate
          =FlushPolicy.onUpdate
            FlushByRamOrCountsPolicy.onInsert
            FlushByRamOrCountsPolicy.onDelete
                DocumentsWriterFlushControl.setFlushPending{ThreadState}
                  设置ThreadState.flushPending
              如果ramBufferSizeMB达到了->
                FlushByRamOrCountsPolicy.markLargestWriterPending
                  findLargestNonPendingWriter 找到最大的非PendingWriter
                  DocumentsWriterFlushControl.setFlushPending
                    设置其ThreadState.flushPending
              如果单个线程超过2G, 设置ThreadState.flushPending}
      DocumentsWriterFlushControl.tryCheckoutForFlush
        internalTryCheckOutForFlush
          ${flushingWriters}.put{DocumentsWriterPerThread, bytes} 记录字节数
    DocumentsWriter.postUpdate
      DocumentsWriter.applyAllDeletes
      DocumentsWriter.doFlush{DocumentsWriterPerThread}
        DocumentsWriterPerThread.flush()
          DocumentsWriterPerThread.codec.liveDocsFormat()
            ~Lucene50LiveDocsFormat.newLiveDocs{numDocsInRAM} 创建
              FixedBitSet[new]
          DefaultIndexingChain.flush 写入常规文件
          sealFlushedSegment
            如果getUseCompoundFile 合并常规文件为.cfs, .cfe文件
              DocumentsWriterPerThread.codec.compoundFormat().write [.cfs, .cfe]
                ~Lucene50CompoundFormat.write
            DocumentsWriterPerThread.codec.segmentInfoFormat().write [*.si]
              ~Lucene70SegmentInfoFormat.write 写入.si文件
            如果存在删除doc
              DocumentsWriterPerThread.codec.liveDocsFormat
                ~Lucene50LiveDocsFormat.writeLiveDocs
            SegmentCommitInfo.advanceDelGen
  DocumentsWriter.deleteTerms 
    DocumentsWriterDeleteQueue.addDelete
      DocumentsWriterDeleteQueue.tryApplyGlobalSlice
        加入${globalBufferedUpdates}
    DocumentsWriterFlushControl.doOnDelete
      FlushByRamOrCountsPolicy.onDelete{state <- nil}
        DocumentsWriterFlushControl.setApplyAllDeletes
    DocumentsWriterFlushControl.getAndResetApplyAllDeletes |
      ticketQueue.addDeletes{DocumentsWriterDeleteQueue}
      ${events}增加ApplyDeletesEvent.INSTANCE
  DocumentsWriter.commit
    DocumentsWriterFlushControl.markForFullFlush
      ${fullFlush} <- true
      deleteQueue <- new DocumentsWriterDeleteQueue{generation++, seqNo}
        ; seqNo <- ${deleteQueue}.getLastSequenceNumber + 2 * perThreadPool.getActiveThreadStateCount + 2
      对于每个threadState
        perThread.flushPending <- true
        internalTryCheckOutForFlush
          增加至${flushingWriters}
    对于每个PendingFlush
      doFlush
        DocumentsWriterFlushQueue.addFlushTicket 增加至${ticketQueue}
          [new]SegmentFlushTicket{frozenDeletes:FrozenBufferedUpdates <- DocumentsWriterPerThread.prepareFlush}

          增加至${DocumentsWriterFlushQueue.queue}
        DocumentsWriterPerThread.flush
    DocumentsWriterFlushControl.waitForFlush

DefaultIndexingChain.java
  DefaultIndexingChain.ctor
    [FreqProxTermsWriter -> TermVectorsConsumer, StoredFieldsConsumer]
  DefaultIndexingChain.processDocument
    FreqProxTermsWriter.startDocument 
      TermVectorsConsumer.startDocument 重置字段
    DefaultIndexingChain.startStoredFields
      StoredFieldsConsumer.startDocument{docID}
        DocumentsWriterPerThread.codec.storedFieldsFormat().fieldsWriter
          Lucene70Codec -> Lucene50StoredFieldsFormat ->  CompressingStoredFieldsFormat -> CompressingStoredFieldsWriter[new]
          CompressingStoredFieldsWriter.startDocument
    DefaultIndexingChain.processField 循环处理字段生成PerField信息
      如果索引选项不为NONE
        DefaultIndexingChain.getOrAddField
          1. ${fieldHash}[hashcode]通过hash值得到PerField
          2. ${fieldInfos}.getOrAdd
          3. PerField[new]
        PerField.invert
          IndexableField.tokenStream 得到TokenStream
            > FreqProxTermsWriter.addField -> FreqProxTermsWriterPerField[new]{TermsHashPerField}
            FreqProxTermsWriterPerField.start
              TermsHashPerField.start
              循环field.tokenStream
                FreqProxTermsWriterPerField.add
        加入${fields}, 如果字段第一次出现
      如果字段类型为STORED
        StoredFieldsConsumer.writeField
          CompressingStoredFieldsWriter.writeField

    DefaultIndexingChain.finishStoredFields
    FreqProxTermsWriter.finishDocument
      Lucene70Codec -> Lucene50TermVectorsFormat ->  CompressingTermVectorsFormat -> CompressingTermVectorsWriter[new]

DirectoryReader.java
  DirectoryReader.open
    StandardDirectoryReader.open{IndexCommit=null}
      SegmentInfos.FindSegmentsFile[new]
      SegmentInfos.FindSegmentsFile.run
        SegmentInfos.getLastCommitGeneration
          generationFromSegmentsFileName
        #SegmentInfos.doBody
          SegmentInfos.readCommit
            codec.segmentInfoFormat().read
              ~Lucene70SegmentInfoFormat.read
          ${readers}: SegmentReader[new]
          StandardDirectoryReader[new]
            =DirectoryReader.ctor{Directory, SegmentReader[]}
              =BaseCompositeReader{SegmentReader[]} 
                ${subReaders} <- SegmentReader[]
                ${starts} <- 每个segment里面的maxDoc数累积和

IndexSearcher.java
  IndexSearcher.doc
    StandardDirectoryReader.document
      =BaseCompositeReader.document
        readerIndex{docId} 通过${starts}二分查找得到SegmentReader
        SegmentReader.document
          CompressingStoredFieldsReader.visitDocument{docId}
        

```
