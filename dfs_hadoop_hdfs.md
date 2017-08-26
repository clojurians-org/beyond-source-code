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
    DFSOutputStream$DataStreamer[new]
  DFSOutputStream.start
    DFSOutputStream$DataStreamer.start->run()
      DFSOutputStream$DataStreamer.processDatanodeError
        ${dataQueue}.addAll 从${ackQueue}队列回至${dataQueue}队列后清空
        DFSOutputStream$DataStreamer.setupPipelineForAppendOrRecovery
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
