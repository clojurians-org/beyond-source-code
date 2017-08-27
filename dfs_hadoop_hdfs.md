```text
FileSystem.java
  FileSystem/get
    FileSystem/createFileSystem
      FileSystem/getFileSystemClass 读取配置
        hdfs -> DistributedFileSystem
      DistributedFileSystem[new]
  FileSystem.append
    DistributedFileSystem.append
      DFSClient.append

DFSClient.java
  DFSClient.append
    DFSClient.callAppend
      DFSOutputStream/newStreamForAppend
    HdfsDataOutputStream{DFSOutputStream}
    HdfsDataOutputStream.write
      DataOutputOutputStream.write
        DFSOutputStream.write
      
DFSOutputStream.java
  DFSOutputStream.newStreamForAppend
    DFSOutputStream[new]
    DFSOutputStream.start
  DFSOutputStream.ctor{DFSClient}
    DFSOutputStream$DataStreamer[new]{BlockConstructionStage.PIPELINE_SETUP_CREATE}
  DFSOutputStream.start
    DFSOutputStream$DataStreamer.start->run()
      循环:
        DFSOutputStream$DataStreamer.processDatanodeError
          ${dataQueue}.addAll 从${ackQueue}队列回至${dataQueue}队列后清空
          DFSOutputStream$DataStreamer.setupPipelineForAppendOrRecovery
        等待${dataQueue}队列
        createHeartbeatPacket | 如果${dataQueue}为空
        取出packet
        {BlockConstructionStage.PIPELINE_SETUP_CREATE -> setPipeline
          nextBlockOutputStream 得到LocatedBlock用于写
            locateFollowingBlock 
              DfsClient.namenode.addBlock
            记录当前LocatedBlock的${nodes}
            createBlockOutputStream
              DfsClient.saslClient.socketSend{in,out}
              ${blockStream}:DataOutputStream{BufferedOutputStream}
              ${blockReplyStream}: DataInputStream[new]{in}
              Sender[new]{out}
              Sender.writeBlock
              BlockOpResponseProto/parseFrom{${blockReplyStream}} 建立连接
              如果返回为重启响应, 则设置checkRestart
          initDataStreaming
            DFSOutputStream$ResponseProcessor[new]
            DFSOutputStream$ResponseProcessor.start -> run()
            进入BlockConstructionStage.DATA_STREAMING阶段
        BlockConstructionStage.PIPELINE_SETUP_APPEND -> setupPipelineForAppendOrRecovery}
        DFSOutputStream$DataStreamer.initDataStreaming
        如果是block的最后一个package, 进入BlockConstructionStage.PIPELINE_CLOSE
        从${dataQueue}移除加入至${ackQueue}后${dataQueue}.notifyAll
        ${blockStream}.flush
  DFSOutputStream$ResponseProcessor.start -> run()
    PipelineAck[new]
      PipelineAckProto.parseFrom
    对于每个应答，如果有datanode重启, 则打上标记${restartingNodeIndex}
    ${ackQueue}处理完成后去除
    ${dataQueue}.notifyAll 通知${dataQueue}队列

  DFSOutputStream.write
    =FSOutputSummer.write
      FSOutputSummer.flushBuffer | 如果缓存满了
        writeChecksumChunks 
          DataChecksum.calculateChunkedSums {bytesPerChecksum=chunkSize}
          DFSOutputStream.writeChunk
  DFSOutputStream.writeChunk
    DFSOutputStream.writeChunkImpl
      createPacket
        DFSPacket[new]
      如果package满了
        waitAndQueueCurrentPacket
          queueCurrentPacket
            ${dataQueue} 往队列增加package
            ${dataQueue}.notifyAll
```
