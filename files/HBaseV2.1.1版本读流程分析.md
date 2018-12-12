# HBaseV2.1.1版本读流程分析
## 简介 HBase读
hbase作为bigtable系的分布式K-V数据库，自身目前不提供类似SQL的查询方式。本身的查询依赖于扫描文件和内存查询的方式返回需要的数据。
HBase 自身提供get/scan的方式方便客户端与服务端之间进行数据的交互。
## 1 client客户端如何使用读
这里以hbase  api的方式实例：
### 1.1 Get
    以随机读(Scan.ReadType.PREAD) 的方式读取某个rowkey对应的信息。
##### 1.1.1 构建Get请求demo
    。。。省略config 的创建过程。。。
    HTable table = new HTable(conf,Bytes.toBytes(tableName));
    Get get = new  Get(Bytes.toBytes(rowkey));
    Result r = table.get(get);
    for (KeyValue kv : result.list()){
        System.out.println("family:" + Bytes.toString(kv.getFamily()));  //列簇
		System.out.println("qualifier:" + Bytes.toString(kv.getQualifier())); // 列
		System.out.println("value:" + Bytes.toString(kv.getValue()));  //属性值
		System.out.println("Timestamp:" + kv.getTimestamp()); //版本信息 timestamp  
    }
##### 1.1.2 Get 一些重要的方法和参数设置
    addFamily(byte[] family)   //指定get指定的列簇
    addColumn(byte[] family, byte[] qualifier)  //指定get某一个具体的列簇中的列
    setCacheBlocks(boolean cacheBlocks)  //服务端是否对这次get中的block缓存起来
    setConsistency(Consistency consistency)  //设置get的一致性水平 default STRONG
    setFilter(Filter filter)    //添加过滤器过滤部分数据
    readVersions(int)    //读取的版本数
    readAllVersions()   //读取所有的版本数，列族的maxVersions
    setTimeRange(long minStamp, long maxStamp)   //设置查询的版本信息范围 版本默认使用put的时间戳
    setTimestamp(long timestamp)    //查询指定版本号的数据
    setCheckExistenceOnly(boolean checkExistenceOnly)   
    ... and so on 

##### 1.1.3关于batch Get 
    
    try (Table table = conn.getTable(TN)) {
      // 基于多个Get对象构建一个List
      List<Get> gets = new ArrayList<>(8);
      gets.add(new Get(Bytes.toBytes("11110000431^201803011300")));
      gets.add(new Get(Bytes.toBytes("22220000431^201803011300")));
      gets.add(new Get(Bytes.toBytes("33330000431^201803011300")));
      gets.add(new Get(Bytes.toBytes("44440000431^201803011300")));
      gets.add(new Get(Bytes.toBytes("55550000431^201803011300")));
      gets.add(new Get(Bytes.toBytes("66660000431^201803011300")));
      gets.add(new Get(Bytes.toBytes("77770000431^201803011300")));
      gets.add(new Get(Bytes.toBytes("88880000431^201803011300")));
      // 调用Table的Batch Get的接口获取多行记录
      Result[] results = table.get(gets);
      for (Result result : results) {
        CellScanner scanner = result.cellScanner();
          while (scanner.advance()) {
            Cell cell = scanner.current();
              // 读取Cell中的信息...
          }
      }
    }
    batch get 待续。。。(continued ...)
### 1.2 Scan
顺序查询（Scan.ReadType.STREAM ）符合某一范围的的rowkey集合的信息。
![image](2FEC4056055E4234AEA6706B51825D7C)
##### 1.2.1 构建scan客户端请求
。。。省略config 的创建过程。。。

    Scan scan = new Scan();
    ResultScanner rst = table.getScanner(scan);
    for(Result result : rst){
        for(Cell cell : result.rawCells()){
            System.out.println(new String(CellUtil.cloneRow(cell))+"\t"
                    +new String(CellUtil.cloneFamily(cell))+"\t"
                    +new String(CellUtil.cloneQualifier(cell))+"\t"
                    +new String(CellUtil.cloneValue(cell),"UTF-8")+"\t"
                    +cell.getTimestamp());
        }
    }
    rst.close();
    table.close();

##### 1.2.2 scan 常用参数

    	
	addFamily(byte[] family)   //扫描指定的列簇
	addColumn(byte[] family, byte[] qualifier)  //扫描指定的列
	readAllVersions()  //获取所有的version
	readVersions(int versions)  //获取version数量
	setBatch(int batch)   //每次next返回的最大cell数量
	setCacheBlocks(boolean cacheBlocks)  //是否缓存blocks
	setAsyncPrefetch(boolean asyncPrefetch)    //是否prefetch 这个点后面会补充 这篇暂时不讨论
	setCaching(int caching)   //每次next返回的数量  default integer.max
	setMaxResultSize(long maxResultSize)   //内存占用量的维度限制一次Scan的返回结果集  default 2M
	setRaw(boolean raw)   //可以读取到删除标识以及被删除但尚未被清理的数据
	withStartRow(byte[] startRow)  //设置startkey起点
	withStopRow(byte[] stopRow)    //设置stopkey
	setFilter()    //添加过滤器 过滤部分数据
    setTimeRange(long minStamp, long maxStamp)   //设置查询的版本信息范围 版本默认使用put的时间戳
    setTimestamp(long timestamp)    //查询指定版本号的数据
    setLimit(Long limit)    //设置查询的row的最大数量
    setFamilyMap(Map<byte[],NavigableSet<byte[]>> familyMap) //设置列
## 2  流程分析
    
#### 2.1 Get Client
简要流程图：
![image](862B2A5052F24E2D975D06DF6A1AC1B1)    

    //Consistency.STRONG  write&read the same regionserver  
    //Consistency.TIMELINE   write the primary and read replicas
![image](BD69A58265884ADF940BEB4EA11120D2)

#### 2.2 Get Server
    get是一个特殊的scan过程
    scan的最大值和最小值都为rowkey
    因此下面get在服务端的流程可以参考scan服务端的过程。
    
#### 2.3 Scan client
scan 是一个串行的过程，每一次scan将数据放入cache中。当cache中数据遍历完之后，进行下一次scan。通过client的cache中的将数据返回到客户端。
![image](75CD87C83FAD4C719E78C1899BACEA04)
   
   
#### 2.4 Scan Server
regionserver 会有一个scanners来存储scannerid 与 RegionScannerHolder的关系。scan的过程构建了一个scanner体系。

##### 2.4.1 scanner体系。
scanner体系的核心在于三层scanner：RegionScanner、StoreScanner以及StoreFileScanner。三者是层级的关系，一个RegionScanner由多个StoreScanner构成，一张表由多个列族组成，就有多少个StoreScanner负责该列族的数据扫描。一个StoreScanner又是由多个StoreFileScanner组成。每个Store的数据由内存中的MemStore和磁盘上的StoreFile文件组成，相对应的，StoreScanner对象会雇佣一个MemStoreScanner和N个StoreFileScanner来进行实际的数据读取，每个StoreFile文件对应一个StoreFileScanner，注意：StoreFileScanner和MemstoreScanner是整个scan的最终执行者。

下面借用网上的一个图,memstoreScanner包含了（active、snapshots) ：

![image](F2290E6D8A0149FB8C22F5A4FDC12195)


构建最小堆的过程需要比较KeyValue的大小。HBase KeyValue中Key由RowKey，ColumnFamily，Qualifier ，TimeStamp，KeyType等5部分组成，HBase设定Key大小首先比较RowKey，RowKey越小Key就越小；RowKey如果相同就看CF，CF越小Key越小；CF如果相同看Qualifier，Qualifier越小Key越小；Qualifier如果相同再看Timestamp，Timestamp越大表示时间越新，对应的Key越小。如果Timestamp还相同，就看KeyType，KeyType按照DeleteFamily -> DeleteColumn -> Delete -> Put 顺序依次对应的Key越来越大。
eg:
![image](A64C74AD744840F1ACA64E834549726D)

读取顺序：hf2 -> hf1 -> hf4 -> hf3 -> hf5

代码如下：

     protected KeyValueScanner pollRealKV() throws IOException {
            KeyValueScanner kvScanner = heap.poll();
            if (kvScanner == null) {
              return null;
            }

        while (kvScanner != null && !kvScanner.realSeekDone()) {
          if (kvScanner.peek() != null) {
            kvScanner.enforceSeek();
            KeyValue curKV = kvScanner.peek();
            if (curKV != null) {
              KeyValueScanner nextEarliestScanner = heap.peek();
              if (nextEarliestScanner == null) {
                // The heap is empty. Return the only possible scanner.
                return kvScanner;
              }
    
              // Compare the current scanner to the next scanner. We try to avoid
              // putting the current one back into the heap if possible.
              KeyValue nextKV = nextEarliestScanner.peek();
              if (nextKV == null || comparator.compare(curKV, nextKV) < 0) {
                // We already have the scanner with the earliest KV, so return it.
                return kvScanner;
              }
    
              // Otherwise, put the scanner back into the heap and let it compete
              // against all other scanners (both those that have done a "real
              // seek" and a "lazy seek").
              heap.add(kvScanner);
            } else {
              // Close the scanner because we did a real seek and found out there
              // are no more KVs.
              kvScanner.close();
            }
          } else {
            // Close the scanner because it has already run out of KVs even before
            // we had to do a real seek on it.
            kvScanner.close();
          }
          kvScanner = heap.poll();
        }

        return kvScanner;
    }

##### 2.4.2 scan 流程

![image](213A30A255FF4ABA945400DA2DEAAE07)

在read过程中会涉及到block的cache过程。后面简单介绍下HBase2.1.1的blockcache 结构。


##### 2.4.3 seek row && scan HFile

这里涉及到HFile的结构

![image](51733395DE824C1C846E654A74E06A05)

这里简单介绍下：
    Data Block 用来存储KeyValue
    ROOT_INDEX 用来是Data Block的索引
    如果ROOT_INDEX 太多的话 会产生LEAF_IDEX 就构成了LEAF_INDEX是HFile的索引，ROOT_INDEX是LEAF_INDEX的索引的结构。
tailer会记录索引结构是两层还是一层。
![image](874DFC17D8424010AF4FE5BA7F16457E)






##### 2.4.4 scan过程中block与cache

![image](D155F19B28E44A2E9233CBD266114892)

##### 2.4.5 scan version

scan 通过NormalUserScanQueryMatcher类来判断version、ttl、columns、timerang、delete等信息。 
    这个类似于flush和compaction。
    

##### 2.4.6 lazy seek

在这个优化之前，读取一个column family(Store)，需要seek其下的所有HFile和MemStore到指定的查询KeyValue(seek的语义为如果KeyValue存在则seek到对应位置，如果不存在，则seek到这个KeyValue的后一个KeyValue，假设Store下有3个HFile和一个MemStore，按照时序递增记为[HFile1, HFile2, HFile3, MemStore],在lazy seek优化之前，需要对所有的HFile和MemStore进行seek，对HFile文件的seek比较慢，往往需要将HFile相应的block加载到内存，然后定位。大体来说，思路是请求seek某个KeyValue时实际上没有对StoreFileScanner进行真正的seek，而是对于每个StoreFileScanner，设置它的peek为(rowkey,family,qualifier,lastTimestampInStoreFile)

当我们使用scan的时候  scan.withStartRow(Bytes.toBytes(rowkey)) 这个将创建一个FirstOnRowDeleteFamilyCell  列簇cf 类型 KeyValue.Type.DeleteFamily timestamp为HConstants.LATEST_TIMESTAMP（LongMax)。

scan lazy使用条件：

首先需要设置setFamilyMap(Map<byte[],NavigableSet<byte[]>> familyMap)和
lazySeekEnabledGlobally （default true)。


逻辑如下：

    Step1:
        1：如果使用的BloomFilterType == ROWCOL的话：
           passesGeneralRowColBloomFilter(kv){
               kvKey = create FirstOnRowColCell  //extends FirstOnRowCell timestamp long.max type=255
               return  checkGeneralBloomFilter(null, kvKey, bloomFilter){
                   return exist?
               }
           }
                
        
        2、如果不是ROWCOL类型的话：
            anOptimizeForNonNullColumn && ((PrivateCellUtil.isDeleteFamily(kv) || PrivateCellUtil.isDeleteFamilyVersion(kv))
        
            reader.passesDeleteFamilyBloomFilter
    Step2 : 
        3、if  exist?
            realSeekDone = false;
            if kv timestamp > HFile Timestamp ?
                setCurrentCell(PrivateCellUtil.createFirstOnRowColTS(kv, maxTimestampInFile));  // KeyValue.Type.Maximum  HFile.maxtimestamp
            else 
                realSeek();
        4、if not exist in bloomfilter
            setCurrentKeyValue = LastOnRowColCell 
            //HConstants.OLDEST_TIMESTAMP  KeyValue.Type.Minimum
            realSeekDone = true;
        
![image](A68ABAE6EE334376BEE6F4C39A8EC203)



### 3 BlockCache



BlockCache 分为(L1) LRUBlockCache 和 （L2) BucketCache 两部分。

![image](FCC6BFBF20974DABAED696B0986FC68B)
这里主要简单介绍下L2的offheap方式，L2使用java nio 提供的DirectByteBuffer的方式使用对data block块进行存储。
在使用L2的前提下LRUBlockCache只保存Index/Bloom block。当L2不启用时候这时候LRUBlockCache中也会cache Data Block。
cacheBlock参数能设置是否缓存block，但是在存在leaf index的情况下root index block会被cache。
