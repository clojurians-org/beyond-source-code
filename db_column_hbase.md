```text
;--------
; SERVER
;-------
hbase.sh
  master [start|stop]
    org.apache.hadoop.hbase.master.HMaster
  regionserver [start|stop]
    org.apache.hadoop.hbase.regionserver.HRegionServer

HRegionServer.java
  HRegionServer.ctor
    createRpcServices
      RSRpcServices[new]
    CompactedHFilesDischarger[new]

RSRpcServices.java
  RSRpcServices.ctor
    RpcServerFactory.createRpcServer
      #getServices()
  RSRpcServices.getRegion
    HRegionServer.getRegionByEncodedName
      ${onlineRegions}.get
  RSRpcServices.get{RpcController,GetRequest}
    RSRpcServices.getRegion
    RegionCoprocessorHost.preExists
      RSRpcServices.get{Get, HRegion, RegionScannersCloseCallBack, RpcCallContext}
        RegionCoprocessorHost.preGet
        Scan[new]{Get}
          startRow <- Get.getRow
          stopRow <- Get.getRow
          HRegion$RegionScannerImpl.next{Scan}
            HRegion$RegionScannerImpl.next
        RegionCoprocessorHost.postGet
      <if>不为ExistenceOnly
        ProtobufUtil.toResultNoData(r)
      <else> RegionCoprocessorHost.postExists 

HRegion.java
  HRegion.ctor
  HRegion$RegionScannerImpl.next{Scan}
    HRegion$RegionScannerImpl.next
      HRegion$RegionScannerImpl.nextRaw
        HRegion$RegionScannerImpl.nextInternal
          HRegion$RegionScannerImpl.nextRow
            HRegion$RegionScannerImpl.postScannerFilterRow


HMaster.java
  HMaster.main
    HMasterCommandLine.doMain
      HMasterCommandLine[new]
      =ServerCommandLine.doMain
        ToolRunner.run{HBaseConfiguration.create, HMasterCommandLine, args}
          HMasterCommandLine.setConf
          HMasterCommandLine.run
            HMasterCommandLine.startMaster
              logProcessInfo
              HMaster.constructMaster[new]
                HMaster[new]{HBaseConfiguration, CoordinatedStateManager[new]}
              HMaster.start
                =Runnable@HasThread@HRegionServer.start
                  HMaster.execute()
              HMaster.join
                =Runnable@HasThread@HRegionServer.join
  HMaster.ctor
    =HRegionServer.ctor
      HRegionServer.createRpcServices
        =MasterRpcServices.createRpcServices
          MasterRpcServices[new]
            =RSRpcServices.ctor
              MasterRpcServices.getServices()
                MasterService
                RegionServerStatusService
                RSRpcServices.ClientService
                RSRpcServices.AdminService
              RpcServerFactory.createRpcServer 
                SimpleRpcServer[new]
                  =RpcServer[new]
      MasterRpcServices.start
        SimpleRpcServer.start
          authTokenSecretMgr.start
          responder.start
          listener.start
          scheduler.start
    ActiveMasterManager[new]
    putUpJettyServer
    startActiveMasterManager
      finishActiveMasterInitialization 等待成为leader
        MasterCoprocessorHost[new]
        createServerManager
          setupClusterConnection
          ServerManager[new]
        HMaster.initializeZKBasedSystemTrackers
        HMaster.startServiceThreads
          HMaster.startProcedureExecutor
            MasterProcedureEnv[new]
            WALProcedureStore[new]
            WALProcedureStore.start
            ProcedureExecutor[new]
            ProcedureExecutor.start
            RSProcedureDispatcher.start
              ~ MasterProcedureEnv.getRemoteDispatcher().start
              =RemoteProcedureDispatcher.start
                TimeoutExecutorThread.start
              addNode
        HMaster.waitForRegionServers
  HMaster.run
    =HRegionServer.run


    RpcExecutor.run
      RpcExecutor.consumerLoop
        CallRunner.run
          RpcServer.call
            ClientProtos$ClientService.callBlockingMethod[generated]
              ServerCall{SimpleServerCall,NettyServerCall}
              ~RpcCall.getService.callBlockingMethod
              RSRpcServices.multi
                RSRpcServices.getRegion
;--------
; CLIENT
;-------
ConnectionFactory.java
  ConnectionFactory/createConnection
    ConnectionImplementation[new] 

ConnectionImplementation.java
  ConnectionImplementation.ctor
    HTable[new]
  ConnectionImplementation.getAdmin
    HBaseAdmin[new]

HBaseAdmin.java
  createTable{HTableDescriptor, region}
  tableExists
    MetaTableAccessor.tableExists
      HTable.getScanner
        ClientScanner.nextScanner
          ClientScanner.call
            RpcRetryingCaller.callWithoutRetries
              ScannerCallableWithReplicas.call
                <<>>
                ScannerCallableWithReplicas$RetryingRPC
                  RpcRetryingCaller.callWithRetries
                    ScannerCallable.call[ClientServiceCallable]
                      ClientProtos$ClientService$BlockingStub.scan
                        AbstractRpcClient$BlockingRpcChannelImplementation.callBlockingMethod
                          AbstractRpcClient.callBlockingMethod
                            RpcClientImpl.call
                              RpcClientImpl$Connection.tracedWriteRequest
                                RpcClientImpl$Connection.writeRequest
  
ClientScanner.java
  ClientScanner.next
    RpcRetryingCaller.callWithRetries
      ProtobufUtil.getRemoteException
    
```
