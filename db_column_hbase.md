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
  RSRpcServices.get
    RSRpcServices.getRegion
    RegionCoprocessorHost.preExists
      RegionCoprocessorHost.execOperationWithResult

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
  HMaster.run
    =HRegionServer.run


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
    
```
