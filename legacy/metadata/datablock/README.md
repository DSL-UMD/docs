# Datablock Management

Hadoop HDFS split large files into small chunks known as Blocks. Block is the physical representation of data. It contains a minimum amount of data that can be read or write. HDFS stores each file as blocks. HDFS client doesn’t have any control on the block like block location, Namenode decides all such things. We introduced that all inodes and data block indexes in HDFS Namespace will be serialized into FSImage file and persisted on Namenode's hard drive eventually (See [Section 2.1](https://dsl-umd.github.io/docs/intro/hdfs.html#persistence)). 


However, the relation between data blocks and datanodes haven't been stored in FSImage. In contrast, it's dynamically constructed from the heartbelt of Datanode. No matter where the relations of `inodes <-> data block indexes` and `data blocks <-> datanodes` were before, instead, they can be considered to store through the database.

## Data Block

Hadoop HDFS stores terabytes and petabytes of data which far exceeds the size of a single disk as Hadoop framework break file into blocks and distribute across various nodes. By default, HDFS block size is 128MB which you can change as per your requirement. All HDFS blocks are the same size except the last block, which can be either the same size or smaller. Hadoop framework break files into 128 MB blocks and then stores into the Hadoop file system.  If HDFS Block size is 4kb like Linux file system, then we will have too many data blocks in Hadoop HDFS, hence too much of metadata. So, maintaining and managing this huge number of blocks and metadata will create huge overhead and traffic which is something which we don’t want. Block size can’t be so large that the system is waiting a very long time for one last unit of data processing to finish its work.

Namenode maintains the two most important relations in HDFS:

- Directory tree of HDFS file system and data block indexes of files (inodes **<-->** data block indexes);
- The mapping relationship between data blocks and datanodes, that is, the information on which datanodes a datablock is stored (data blocks **<-->** datanodes).

## INodeFile

[INodeFile.blocks](https://github.com/DSL-UMD/hadoop-calvin/blob/calvin/hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/namenode/INodeFile.java#L251) field records all data blocks a file contains. It is also through this field that HDFS associates the first relation with the second relation. INodeFile.blocks is an array of `BlockInfo` which inherits from `Block` class, HDFS uses `Block` to abstract the data structure in Namenode.

```java
public class INodeFile extends INodeWithAdditionalFields
    implements INodeFileAttributes, BlockCollection {
    private BlockInfo[] blocks;
}
```


## Block

`Block` is used to uniquely identify data blocks in Namenode and is the most basic abstract interface of the data block in HDFS. Block class defines [three fields](https://github.com/DSL-UMD/hadoop-calvin/blob/c337680e23ded375df17c09a878f719102a47773/hadoop-hdfs-project/hadoop-hdfs-client/src/main/java/org/apache/hadoop/hdfs/protocol/Block.java#L92-L94):

```java
public class Block implements Writable, Comparable<Block> {
    private long blockId;
    private long numBytes;
    private long generationStamp;
}
```

- **blockId** uniquely identifies this Block object;
- **numBytes** is the size of this data block (in bytes);
- **generationStamp** is the timestamp of this data block.

## BlockInfo

`BlockInfo` extends from Block class and is a supplementary of Block class. For a given block, BlockInfo class maintains BlockCollection and datanodes
where the replicas of the block are stored. [The following fields](https://github.com/DSL-UMD/hadoop-calvin/blob/c337680e23ded375df17c09a878f719102a47773/hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/blockmanagement/BlockInfo.java#L41-L64) are the most meaningful to us:

```java
public abstract class BlockInfo extends Block {
    // Replication factor.
    private short replication;
    // Block collection ID.
    private volatile long bcId;
    // Storages this block is replicated on
    protected DatanodeStorageInfo[] storages;
}
```

- **replication**: If the replication factor was set to 3 (default value in HDFS), there would be one original block and two replicas. For each block stored in HDFS, there will be n – 1 duplicated blocks distributed across the cluster.
- **bcId**: Block collection ID is an alias for INode ID which can uniquely identifies the INode object of the HDFS file through `INodeMap` (see [Section 3.1 - FSDirectory](https://dsl-umd.github.io/docs/metadata/namespace/index.html#fsdirectory)). With this design, both data blocks and inode can easily find each other. For example, FSNameSystem ([FSNamesystem.java#L3640-L3648](https://github.com/DSL-UMD/hadoop-calvin/blob/88528d2ef1ac4926c7716d35ad6c7cd3aa2bc5f0/hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/namenode/FSNamesystem.java#L3640-L3648)) has a member function `getBlockCollection` to find INode from BlockInfo:

    ```java
    // hadoop/hdfs/server/namenode/FSNamesystem.java
    INodeFile getBlockCollection(BlockInfo b) {
        return getBlockCollection(b.getBlockCollectionId());
    }

    @Override
    public INodeFile getBlockCollection(long id) {
        INode inode = getFSDirectory().getInode(id);
        return inode == null ? null : inode.asFile();
    }
    ```

- **storages** is an array of `DatanodeStorageInfo`. A Datanode has one or more types of storages such as HDD, SSD, RAM, RAID, tape, remote storage (such
as NAS) etc. A storage in the Datanode is represented by this class. More information about DatanodeStorageInfo is covered by [Section 3.3 - Datanode Management](https://dsl-umd.github.io/docs/metadata/datenode/index.html)

## BlocksMap

`BlocksMap` is the most important class associated with data blocks. It manages the metadata of data blocks on the Namenode, including which HDFS file the current data block belongs to, and which Datanodes the current data block is stored on. When the Datanode starts, it scans the local disk and reports the data block information to the Namenode. After the Namenode receives the block report from the Datanode, it wi).ll establish the correspondence between the data block and the Datanode that holds the data block, and save this information in the BlocksMap. Whether to obtain the HDFS file corresponding to a certain data block or to acquire which Datanodes the data block is stored on, it is necessary to use BlocksMap object.

```java
class BlocksMap {
  private final int capacity;
  private GSet<Block, BlockInfo> blocks;
  private final LongDataAdder totalReplicatedBlocks = new LongAdder();
  private final LongAdder totalECBlockGroups = new LongAdder();
}
```

We will analyse the memory consumption of `BlocksMap` in [Section 3.4 - Quantitative Analysis - Data Block](https://dsl-umd.github.io/docs/analysis.html#data-block).