1.Explain what is checksum and the importance of checksum and how hadoop performs checksum
Ans:
-Checksum is uesd for detecting the errors which are generated during storage or processing.
-Users of Hadoop expect that no data will be lost or corrupted during storage or processing.
-However, since every I/O operation on the disk or network carries with it a small chance of introducing errors into the data that it is reading or writing, when the volumes of data flowing through the system Hadoop is capable of handling, the chance of data corruption occurring is high.
-The usual way of detecting corrupted data is by computing a checksum for the data when it first enters the system, and again whenever it is transmitted across a channel that is unreliable and hence capable of corrupting the data.
-The data is deemed to be corrupt if the newly generated checksum doesn’t exactly match the original.
-This technique doesn’t offer any way to fix the data—merely error detection.
-It is possible that it’s the checksum that is corrupt, not the data, but this is very unlikely,since the checksum is much smaller than the data.
-A commonly used error-detecting code is CRC-32, which computes a 32-bit integer checksum for input of any size.
-HDFS transparently checksums all data written to it and by default verifies checksums when reading data.
-A separate checksum is created for every io.bytes.per.checksum 83 bytes of data. The default is 512 bytes, and since a CRC-32 checksum is 4 bytes long,the storage overhead is less than 1%.
-Datanodes are responsible for verifying the data they receive before storing the data and its checksum. This applies to data that they receive from clients and from other datanodes during replication.
-A client writing data sends it to a pipeline of datanodes, and the last datanode in the pipeline verifies the checksum.
-If it detects an error, the client receives a ChecksumException, a subclass of IOException, which it should handle in an application specific manner, by retrying the operation.
-When clients read data from datanodes, they verify checksums as well, comparing them with the ones stored at the datanode. Each datanode keeps a persistent log of checksum verifications, so it knows the last time each of its blocks was verified.
-Each datanode runs a DataBlockScanner in a background thread that periodically verifies all the blocks stored on the datanode
-If a client detects an error when reading a block, it reports the bad block and the datanode it was trying to read from to the namenode before throwing a ChecksumException.
-The namenode marks the block replica as corrupt, so it doesn’t direct clients to it, or try to copy this replica to another datanode. It then schedules a copy of the block to be replicated on another datanode, so its replication factor is back at the expected level.Once this has happened, the corrupt replica is deleted.
-It is possible to disable verification of checksums by passing false to the setVerifyChecksum() method on FileSystem, before using the open() method to read a file.


2.Explain the anatomy of file write to HDFS.
Ans:
-The client creates the file by calling create() on DistributedFileSystem.
-DistributedFileSystem makes an RPC call to the namenode to create a new file in the filesystem’s namespace, with no blocks associated with it.
-The namenode performs various checks to make sure the file doesn’t already exist, and that the client has the right permissions to create the file. If these checks pass, the namenode makes a record of the new file; otherwise, file creation fails and the client is thrown an IOException.
-The DistributedFileSystem returns an FSDataOutputStream for the client to start writing data. FSDataOutputStream wraps a DFSOutputStream, which handles communication with the datanodes and namenode.
-DFSOutputStream splits it into packets, which it writes to an internal queue, called the data queue.
-The list of datanodes forms a pipeline.The DataStreamer streams the packets to the first datanode in the pipeline,which stores the packet and forwards it to the second datanode in the pipeline.
-Similarly,the second datanode stores the packet and forwards it to the third datanode in the pipeline.
-DFSOutputStream also maintains an internal queue of packets that are waiting to be acknowledged by datanodes, called the ack queue. A packet is removed from the ack queue only when it has been acknowledged by all the datanodes in the pipeline.
-If a datanode fails first the pipeline is closed.The current block on the good datanodes is given a new identity, which is communicated to the namenode,so that the partial block on the failed datanode will be deleted if the failed datanode recovers later on.
-The failed datanode is removed from the pipeline and the remainder of the block’s data is written to the two good datanodes in the pipeline.The namenode notices that the block is under-replicated, and it arranges for a further replica to be created on another node.
-When the client has finished writing data, it calls close() on the stream.
-This action flushes all the remaining packets to the datanode pipeline and waits for acknowledgments before contacting the namenode to signal that the file is complete.



3.Explain how HDFS handles failures during file write
-Suppose that we are writing the blocks in the data nodes and while writing one of the writing to the data node fails.
-Immediately the pipeline will be closed.
-The current block on the good datanodes is given a new identity, which is communicated to the namenode,so that the partial block on the failed datanode will be deleted if the failed datanode recovers later on.
-The failed datanode is removed from the pipeline and the remainder of the blocks data is written to the two good datanodes in the pipeline.
-The namenode notices that the block is under-replicated, and it arranges for a further replica to be created on another node.
