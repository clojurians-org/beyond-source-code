```text
;--------
; SERVER
;-------
hbase.sh
  master [start|stop]
    org.apache.hadoop.hbase.master.HMaster
  regionserver [start|stop]
    org.apache.hadoop.hbase.regionserver.HRegionServer

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
    
