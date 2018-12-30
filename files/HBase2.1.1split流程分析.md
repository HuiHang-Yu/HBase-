## split

### 1、什么是split

##### 1.1、split策略

###### 1.1.1、预切分（pre-splitting) ：

    当一个table刚被创建的时候，Hbase默认的分配一个region给table。如果table需要写入的数据量较大，所有的读写请求都会访问到同一个regionServer的同一个region中，这个regionserver负载就可能较高，而集群中的其他regionServer就可能会处于比较空闲的状态。为了避免这种情况需要根据rowkey进行提前切分。
    
hbase提供了两种 RegionSplitter ：HexStringSplit 和  UniformSplit。
如果我们的row key是十六进制的字符串作为前缀的，就比较适合用HexStringSplit。

    hbase shell eg:
        create 't1', 'f1', {NUMREGIONS => 15, SPLITALGO => 'HexStringSplit'}
        get_splits 't1'
        result :
        => ["11111111", "22222222", "33333333", "44444444", "55555555", "66666666", "77777777", "88888888", "99999999", "aaaaaaaa", "bbbbbbbb", "cccccccc", "dddddddd", "eeeeeeee"]
    UniformSplit使用随机的字母来作为切分点：
    hbase shell  eg :
        create 't2', 'f1', {NUMREGIONS => 15, SPLITALGO => 'UniformSplit'}
        get_splits 't2' 
        
        result :
        => ["\\x11\\x11\\x11\\x11\\x11\\x11\\x11\\x11", "\"\"\"\"\"\"\"\"", "33333333", "DDDDDDDD", "UUUUUUUU", "ffffffff", "wwwwwwww", "\\x88\\x88\\x88\\x88\\x88\\x88\\x88\\x88", "\\x99\\x99\\x99\\x99\\x99\\x99\\x99\\x99", "\\xAA\\xAA\\xAA\\xAA\\xAA\\xAA\\xAA\\xAA", "\\xBB\\xBB\\xBB\\xBB\\xBB\\xBB\\xBB\\xBB", "\\xCC\\xCC\\xCC\\xCC\\xCC\\xCC\\xCC\\xCC", "\\xDD\\xDD\\xDD\\xDD\\xDD\\xDD\\xDD\\xDD", "\\xEE\\xEE\\xEE\\xEE\\xEE\\xEE\\xEE\\xEE"]
当然也可以定义自己的splitter，也可以设置自己的切分点。
        
        hbase shell eg :
        create 't5', 'f1', {SPLITS=>['111111','222222','333333','444444','555555']}
        create 't1', 'f1', SPLITS_FILE => 'splits.txt'
        splits.txt定义你自己的切分点。

###### 1.1.2、自动切分（auto-splitting) ：

    当一个reion达到一定的大小，它会自动split成两个region。hbase 提供三种策略：ConstantSizeRegionSplitPolicy(0.94版本之前默认策略),IncreasingToUpperBoundRegionSplitPolicy(0.94版本之后默认策略)还有 KeyPrefixRegionSplitPolicy。策略的选择可以通过配置hbase.regionserver.region.split.policy 来指定。也可以根据自己的实际场景指定自己的策略。
    ConstantSizeRegionSplitPolicy是0.94版本之前 默认和唯一的split策略。当某个store（对应一个column family）的大小大于配置值 ‘hbase.hregion.max.filesize’的时候（默认10G）region就会自动分裂。
    IncreasingToUpperBoundRegionSplitPolicy策略中，最小的分裂大小和table的某个regionserver的region个数有关。具体公式如下：
        当R == 0 || R > 100 时候：checksize 为 regionmaxsize(tabledescribe 'MAX_FILESIZE' or hbase.hregion.max.filesize (default 10 G))。
        当 R < 100 时：checksize 为 Math.min(regionmaxsize ,（R^3 * initialSize) //initialSize(hbase.increasing.policy.initial.size or 2 * tabledescribe 'MEMSTORE_FLUSHSIZE'  or 2 * hbase.hregion.memstore.flush.size (default 128M))
        
        storeFileSize > checksize时需要进行split。
        
        R为同一个table中在同一个region server中region的个数。
        hbase.hregion.memstore.flush.size 默认值 128MB。
        hbase.hregion.max.filesize默认值为10GB 。
        如果初始时R=1,那么Min(1^3 * 2 * 128MB,10GB)=256MB,也就是说某个store file大小达到256MB的时候。
        当R=2的时候Min(2^3 * 2 * 128MB,10GB)=2G ,当某个store file大小达到2G的时候，就会触发分裂。
         如此类推，当R = 3 的时候，store file 达到10GB的时候就会分裂，也就是说当R > = 3的时候，store file 达到10GB的时候就会分裂。
    
    KeyPrefixRegionSplitPolicy可以保证相同的前缀的row保存在同一个region中。
    指定rowkey前缀位数划分region，通过读取 KeyPrefixRegionSplitPolicy.prefix_length  属性，该属性为数字类型，表示前缀长度，在进行split时，按此长度对splitPoint进行截取。此种策略比较适合固定前缀的rowkey。当table中没有设置该属性，指定此策略效果等同与使用IncreasingToUpperBoundRegionSplitPolicy。
    BusyRegionSplitPolicy策略是继承了IncreasingToUpperBoundRegionSplitPolicy策略，该策略根据region的busy程度和上述IncreasingToUpperBoundRegionSplitPolicy策略来选择是否需要进行split。

###### 1.1.3、强制切分（forced-splitting）：
    hbase shell eg :
        split 'regionName', 'splitKey' 
        split 'tableName'
        split 'regionName'
        
##### 2、什么时候触发split检查
    人为触发：属于上面讲的强制切分的情况。
    自动触发：memstore flush 写入文件/compaction 之前 / 当前storefiles 数据达到block条件的时候会进行split的check。
        
            /**
           * Return the splitpoint. null indicates the region isn't splittable
           * If the splitpoint isn't explicitly specified, it will go over the stores
           * to find the best splitpoint. Currently the criteria of best splitpoint
           * is based on the size of the store.
           */
          public byte[] checkSplit() {
            // Can't split META
            if (this.getRegionInfo().isMetaRegion() ||
                TableName.NAMESPACE_TABLE_NAME.equals(this.getRegionInfo().getTable())) {
              if (shouldForceSplit()) {
                LOG.warn("Cannot split meta region in HBase 0.20 and above");
              }
              return null;
            }
        
            // Can't split a region that is closing.
            if (this.isClosing()) {
              return null;
            }
        
            if (!splitPolicy.shouldSplit()) {
              return null;
            }
        
            byte[] ret = splitPolicy.getSplitPoint();
        
            if (ret != null) {
              try {
                checkRow(ret, "calculated split");
              } catch (IOException e) {
                LOG.error("Ignoring invalid split", e);
                return null;
              }
            }
            return ret;
          }
        
    策略判断是否需要进行切分：
        首先 IncreasingToUpperBoundRegionSplitPolicy
                
                  protected boolean shouldSplit() {
                    boolean force = region.shouldForceSplit();   //是否需要切分
                    boolean foundABigStore = false;
                    // Get count of regions that have the same common table as this.region  
                    int tableRegionsCount = getCountOfCommonTableRegions();   //获取当前regionserver中该表下的所有region的数量。
                    
                    // Get size to check
                    long sizeToCheck = getSizeToCheck(tableRegionsCount);   
                
                    for (HStore store : region.getStores()) {
                      // If any of the stores is unable to split (eg they contain reference files)
                      // then don't split
                      if (!store.canSplit()) {
                        return false;
                      }
                
                      // Mark if any store is big enough
                      long size = store.getSize();
                      if (size > sizeToCheck) {
                        LOG.debug("ShouldSplit because " + store.getColumnFamilyName() +
                          " size=" + StringUtils.humanSize(size) +
                          ", sizeToCheck=" + StringUtils.humanSize(sizeToCheck) +
                          ", regionsWithCommonTable=" + tableRegionsCount);
                        foundABigStore = true;
                      }
                    }
                
                    return foundABigStore || force;
                  }
                  
                  来看一下getSizeToCheck()
                  
                  /**
                   * @return Region max size or {@code count of regions cubed * 2 * flushsize},
                   * which ever is smaller; guard against there being zero regions on this server.
                   */
                  protected long getSizeToCheck(final int tableRegionsCount) {
                    // safety check for 100 to avoid numerical overflow in extreme cases
                    return tableRegionsCount == 0 || tableRegionsCount > 100
                               ? getDesiredMaxFileSize()   //tabledescribe 'MAX_FILESIZE' or hbase.hregion.max.filesize (default 10 G)
                               : Math.min(getDesiredMaxFileSize(),
                                          initialSize * tableRegionsCount * tableRegionsCount * tableRegionsCount);  //initialSize  hbase.increasing.policy.initial.size or 2 * tabledescribe 'MEMSTORE_FLUSHSIZE'  or 2 * hbase.hregion.memstore.flush.size (default 128M)
                  }
                
            其次 ConstantSizeRegionSplitPolicy ：
                if (store.getSize() > desiredMaxFileSize) { //tabledescribe 'MAX_FILESIZE' or hbase.hregion.max.filesize (default 10 G)
                    foundABigStore = true;  
                  }
            再次 BusyRegionSplitPolicy：
            
            /**
             * This class represents a split policy which makes the split decision based
             * on how busy a region is. The metric that is used here is the fraction of
             * total write requests that are blocked due to high memstore utilization.
             * This fractional rate is calculated over a running window of
             * "hbase.busy.policy.aggWindow" milliseconds. The rate is a time-weighted
             * aggregated average of the rate in the current window and the
             * true average rate in the previous window.
             *
             */
             BusyRegionSplitPolicy extends IncreasingToUpperBoundRegionSplitPolicy
             
                minage // 最小window hbase.busy.policy.minAge default 600000
                maxBlockedRequests //hbase.busy.policy.blockedRequests default 0.2f
                aggregationWindow //hbase.busy.policy.aggWindow  default 300000 5min
                
               protected boolean shouldSplit() {
                    float blockedReqRate = updateRate();  //通过定义当region中的的请求的时候该region的memstore的和大于 blockingMemStoreSize(hbase.hregion.memstore.flush.size * hbase.hregion.memstore.block.multiplier default 4 * 128M)   
                    if (super.shouldSplit()) {   //IncreasingToUpperBoundRegionSplitPolicy 策略
                      return true;
                    }
                
                    if (EnvironmentEdgeManager.currentTime() <  startTime + minAge) {  
                      return false;
                    }
                
                    for (HStore store: region.getStores()) {
                      if (!store.canSplit()) {
                        return false;
                      }
                    }
                
                    if (blockedReqRate >= maxBlockedRequests) {
                      if (LOG.isDebugEnabled()) {
                        LOG.debug("Going to split region " + region.getRegionInfo().getRegionNameAsString()
                            + " because it's too busy. Blocked Request rate: " + blockedReqRate);
                      }
                      return true;
                    }
                
                    return false;
                  }
                  
        如何计算rate:
            将window内的blockcount 和 writeCount 相除。得到一个busyrate。
                 /**
                   * Update the blocked request rate based on number of blocked and total write requests in the
                   * last aggregation window, or since last call to this method, whichever is farthest in time.
                   * Uses weighted rate calculation based on the previous rate and new data.
                   *
                   * @return Updated blocked request rate.
                   */
                  private synchronized float updateRate() {
                    float aggBlockedRate;
                    long curTime = EnvironmentEdgeManager.currentTime();
                
                    long newBlockedReqs = region.getBlockedRequestsCount();
                    long newWriteReqs = region.getWriteRequestsCount();
                
                    aggBlockedRate =
                        (newBlockedReqs - blockedRequestCount) / (newWriteReqs - writeRequestCount + 0.00001f);
                
                    if (curTime - prevTime >= aggregationWindow) {
                      blockedRate = aggBlockedRate;
                      prevTime = curTime;
                      blockedRequestCount = newBlockedReqs;
                      writeRequestCount = newWriteReqs;
                    } else if (curTime - startTime >= aggregationWindow) {
                      // Calculate the aggregate blocked rate as the weighted sum of
                      // previous window's average blocked rate and blocked rate in this window so far.
                      float timeSlice = (curTime - prevTime) / (aggregationWindow + 0.0f);
                      aggBlockedRate = (1 - timeSlice) * blockedRate + timeSlice * aggBlockedRate;
                    } else {
                      aggBlockedRate = 0.0f;
                    }
                    return aggBlockedRate;
                  }
                  
            如果blockrate >= maxBlockedRequests 就应该split。
        最后 KeyPrefixRegionSplitPolicy extends IncreasingToUpperBoundRegionSplitPolicy ：
            切分策略与IncreasingToUpperBoundRegionSplitPolicy相同只不过是切分点的确定不一致。
            
##### 3、怎样确定切分点
    IncreasingToUpperBoundRegionSplitPolicy采取的方案是按照哪个storefile大，选择使用哪个storefile的keypoint，列簇的splitrow 的选择是通过storefile最大的那个的splitpoint。top daughter will create first row in the end rowkey:null:null:255:LONG.MAX_VALUE。
    at the same time bottom daughter region will create a startrowkey
      rowkey:null:null:255:LONG.MAX_VALUE。
      refenrence文件中写入 Refenrence.Range.Top/Bottom 和 splitkey 。

          /**
           * @return the key at which the region should be split, or null
           * if it cannot be split. This will only be called if shouldSplit
           * previously returned true.
           */
          protected byte[] getSplitPoint() {
            byte[] explicitSplitPoint = this.region.getExplicitSplitPoint();   //get force split region splitkeypoint
            
            if (explicitSplitPoint != null) {
              return explicitSplitPoint;
            }
            List<HStore> stores = region.getStores();
        
            byte[] splitPointFromLargestStore = null;
            long largestStoreSize = 0;
            for (HStore s : stores) {
              Optional<byte[]> splitPoint = s.getSplitPoint();
              // Store also returns null if it has references as way of indicating it is not splittable
              long storeSize = s.getSize();
              if (splitPoint.isPresent() && largestStoreSize < storeSize) {
                splitPointFromLargestStore = splitPoint.get();   //哪个storefile听谁的。
                largestStoreSize = storeSize;
              }
            }
        
            return splitPointFromLargestStore;
          }
    store列簇选择splitpoint的过程
        
        static Optional<byte[]> getSplitPoint(Collection<HStoreFile> storefiles,
         CellComparator comparator) throws IOException {
            Optional<HStoreFile> largestFile = StoreUtils.getLargestFile(storefiles);
            return largestFile.isPresent() ? StoreUtils.getFileSplitPoint(largestFile.get(), comparator)
                : Optional.empty();
            }
            
    HFile选择splitpoint的过程为：mid-point of the given file
        
        static Optional<byte[]> getFileSplitPoint(HStoreFile file, CellComparator comparator)
              throws IOException {
            StoreFileReader reader = file.getReader();
            if (reader == null) {
              LOG.warn("Storefile " + file + " Reader is null; cannot get split point");
              return Optional.empty();
            }
            // Get first, last, and mid keys. Midkey is the key that starts block
            // in middle of hfile. Has column and timestamp. Need to return just
            // the row we want to split on as midkey.
            Optional<Cell> optionalMidKey = reader.midKey();
            if (!optionalMidKey.isPresent()) {
              return Optional.empty();
            }
            Cell midKey = optionalMidKey.get();
            Cell firstKey = reader.getFirstKey().get();
            Cell lastKey = reader.getLastKey().get();
            // if the midkey is the same as the first or last keys, we cannot (ever) split this region.
            if (comparator.compareRows(midKey, firstKey) == 0 ||
                comparator.compareRows(midKey, lastKey) == 0) {
              if (LOG.isDebugEnabled()) {
                LOG.debug("cannot split {} because midkey is the same as first or last row", file);
              }
              return Optional.empty();
            }
            return Optional.of(CellUtil.cloneRow(midKey));
          }
          
          
          
HFIle通过midkey 快速定位split的splitkeypoint  



HFile 多级索引图如下：

![image](578DBB6AFF944911B3CE3B08BF5D383F)
 

Mid-key and multi-level
对于multi-level root index，除了上面index entry数组之外还带有格外的数据mid-key的信息，这个mid-key是用于在对hfile进行split时，快速定位HFile的中间位置所使用。Multi-level root index在硬盘中的格式见图3.4。

Mid-key的含义：如果HFile总共有n个data block，那么mid-key就是能定位到第(n - 1)/2个data block的信息。

Mid-key的信息组成如下：

1、Offset：所在的leaf index chunk的起始偏移量

2、On-disk size：所在的leaf index chunk的长度

3、Key：在leaf index chunk中的位置。

如下图所示：第(n – 1)/2个data block位于第i个LeafIndexChunk，如果LeafIndexChunk[i]的第一个data block的序号为k，那么offset、on-disk size以及key的值如下：

Offset为 LeafIndexChunk[i] 的offset

On-disk size为LeafIndexChunk[i] 的size

Key为(n – 1)/2 – k         
![image](7DA5F5B397D143CD91084973C79FB927)



通过offset ondisksize 读取block 然后获取对应的key值。

hfile block在存midkey的时候采用了shortKey的思路，比如上个block最后一个rowkey为"the quick brown fox", 当前block第一个rowkey为"the who", 那么我们可以用"the r"来作为midkey，和hbase rowkey的scan规则保持一致

存储短一点的虚拟midkey有两个好处：

1. 减少index部分的存储空间，因为自定义的rowkey可以会出现几KB这样极端的长度，精简过后，只需要几个字节

2. 采用与上一个block的最后一个rowkey更接近的虚拟key作为midkey，可以避免潜在的io浪费。如果midkey采用当前block的第一个rowkey，那么当查询的rowkey比midkey小但是比上一个block的最后一个rowkey大时，会去遍历上一个block，这就出现了无用功。而midkey更接近上一个block的最后一个rowkey时，可以在很大程度上避免这个问题，即直接返回该rowkey不在此hfile中。

KeyPrefixRegionSplitPolicy 策略还会在返回的keypoint的基础取 TableDescriptor中的值：
    
        KeyPrefixRegionSplitPolicy.prefix_length / prefix_split_key_policy.prefix_length  //前者没有的话才会选后者

最后取得key的前缀（DelimitedKeyPrefixRegionSplitPolicy咱不讨论）。
       
##### 4、split流程

通过regionflush之后的流程开始分析，略过了rpc过程。
    
下图是hbase1版本过程的简单流程图:

![image](4B90CF9BDD134F31AE0708BE2FD20DC8)

下面是hbase2.1.1 split过程流程图。

![image](6496746E1ACA40C39EE176C148796AE2)


对比1 和 2 从大致流程上看在去除了zk的步骤之后，split整体步骤减少。而且显得很有条理性。

###### 4.1、准备工作：

RegionServer流程：

    1、checkSplit()
        首先meta表是不能split的，closing中的表也是不能split的，列簇中存在引用文件的也是不能切的（可能考虑merge_region过程吧）。regionserver的region数目不能超过 hbase.regionserver.regionSplitLimit（default 1000)
    2、获取切分点
    3、创建SplitRequest并提交。//executor
    4、创建RegionStateTransitionContext，并reportRegionStateTransition。
    5、创建ReportRegionStateTransitionRequest。report to master for region state change。
    6、masterrpcservice通过request状态进行相应的操作。

    
master split流程
    
    1、准备工作
        检查master服务是否正常运行
    2、创建SplitTableRegionProcedure并提交。
    3、检查region对应的table是否存在，region对应的表是否处于enabled状态。检查region对应的RegionState是否存在，状态是否为open。是否是一个split完成的region（完成split的parentregion的状态应该为offline）。
    4、这里还需要call Region#checkSplit重新检查是否可split。
    5、创建daughter regions的regioninfo。
    6、 executefromstate
    Step1:
        SplitTableRegionState.SPLIT_TABLE_REGION_PREPARE
              => prepareSplitRegion(env)
                    state = SPLIT_TABLE_REGION_PRE_OPERATION
              <= Flow.HAS_MORE_STATE
    Step2:
        SPLIT_TABLE_REGION_PRE_OPERATION
            =>preSplitRegion(env)
                SPLIT_TABLE_REGION_CLOSE_PARENT_REGION
            <= Flow.HAS_MORE_STATE
    Step3:
        SPLIT_TABLE_REGION_CLOSE_PARENT_REGION
            =>addChildProcedure(createUnassignProcedures(env, getRegionReplication(env)));
                SPLIT_TABLE_REGIONS_CHECK_CLOSED_REGIONS
            <= Flow.HAS_MORE_STATE
    Step4:
        SPLIT_TABLE_REGIONS_CHECK_CLOSED_REGIONS
            =>  if has edits file :
                    reopen parent region and then re-run the close by going back to  // SPLIT_TABLE_REGION_CLOSE_PARENT_REGION.
                
                else :
                    SPLIT_TABLE_REGION_CREATE_DAUGHTER_REGIONS
            <= Flow.HAS_MORE_STATE 
    Step5:
        SPLIT_TABLE_REGION_CREATE_DAUGHTER_REGIONS
            => createDaughterRegions(env);
                SPLIT_TABLE_REGION_WRITE_MAX_SEQUENCE_ID_FILE
            <= Flow.HAS_MORE_STATE 
    Step6:
        SPLIT_TABLE_REGION_WRITE_MAX_SEQUENCE_ID_FILE
            => writeMaxSequenceIdFile(env);
                SPLIT_TABLE_REGION_PRE_OPERATION_BEFORE_META
            <= Flow.HAS_MORE_STATE
    Step7:        
        SPLIT_TABLE_REGION_PRE_OPERATION_BEFORE_META:
            =>  preSplitRegionBeforeMETA(env);
              SPLIT_TABLE_REGION_UPDATE_META
            <= Flow.HAS_MORE_STATE
    Step8:
        SPLIT_TABLE_REGION_UPDATE_META:
            =>  updateMeta(env);
                SPLIT_TABLE_REGION_PRE_OPERATION_AFTER_META
            <= Flow.HAS_MORE_STATE
    Step9:
        SPLIT_TABLE_REGION_PRE_OPERATION_AFTER_META:
            =>  preSplitRegionAfterMETA(env);
              SPLIT_TABLE_REGION_OPEN_CHILD_REGIONS
            <= Flow.HAS_MORE_STATE
    Step10:
       SPLIT_TABLE_REGION_OPEN_CHILD_REGIONS:
            =>  addChildProcedure(createAssignProcedures(env, getRegionReplication(env)));
              SPLIT_TABLE_REGION_POST_OPERATION
            <= Flow.HAS_MORE_STATE
    Step11:
        SPLIT_TABLE_REGION_POST_OPERATION:
            => postSplitRegion(env);
            <= Flow.NO_MORE_STATE;    
            
            
**Step1** 中逻辑如下：
    prepareSplitRegion确保parentRegion的regioninfo不在split/offline状态 以及state处于OPEN/CLOSED状态。最后设置RegionStateNode 处于 SPLITTING状态。
    
    
      /**
       * Prepare to Split region.
       * @param env MasterProcedureEnv
       */
      @VisibleForTesting
      public boolean prepareSplitRegion(final MasterProcedureEnv env) throws IOException {
        // Check whether the region is splittable
        RegionStateNode node =
            env.getAssignmentManager().getRegionStates().getRegionStateNode(getParentRegion());
    
        if (node == null) {
          throw new UnknownRegionException(getParentRegion().getRegionNameAsString());
        }
    
        RegionInfo parentHRI = node.getRegionInfo();
        if (parentHRI == null) {
          LOG.info("Unsplittable; parent region is null; node={}", node);
          return false;
        }
        // Lookup the parent HRI state from the AM, which has the latest updated info.
        // Protect against the case where concurrent SPLIT requests came in and succeeded
        // just before us.
        if (node.isInState(State.SPLIT)) {
          LOG.info("Split of " + parentHRI + " skipped; state is already SPLIT");
          return false;
        }
        if (parentHRI.isSplit() || parentHRI.isOffline()) {
          LOG.info("Split of " + parentHRI + " skipped because offline/split.");
          return false;
        }
    
        // expected parent to be online or closed
        if (!node.isInState(EXPECTED_SPLIT_STATES)) {
          // We may have SPLIT already?
          setFailure(new IOException("Split " + parentHRI.getRegionNameAsString() +
              " FAILED because state=" + node.getState() + "; expected " +
              Arrays.toString(EXPECTED_SPLIT_STATES)));
          return false;
        }
    
        // Since we have the lock and the master is coordinating the operation
        // we are always able to split the region
        if (!env.getMasterServices().isSplitOrMergeEnabled(MasterSwitchType.SPLIT)) {
          LOG.warn("pid=" + getProcId() + " split switch is off! skip split of " + parentHRI);
          setFailure(new IOException("Split region " + parentHRI.getRegionNameAsString() +
              " failed due to split switch off"));
          return false;
        }
    
        // set node state as SPLITTING
        node.setState(State.SPLITTING);
    
        return true;
      }
      
**Step2** 为预切分主要逻辑如下：quotamanager 和 RegionNormalizer check 。
    
         /**
           * Action before splitting region in a table.
           * @param env MasterProcedureEnv
           */
          private void preSplitRegion(final MasterProcedureEnv env)
              throws IOException, InterruptedException {
            final MasterCoprocessorHost cpHost = env.getMasterCoprocessorHost();
            if (cpHost != null) {
              cpHost.preSplitRegionAction(getTableName(), getSplitRow(), getUser());
            }
        
            // TODO: Clean up split and merge. Currently all over the place.
            // Notify QuotaManager and RegionNormalizer
            try {
              env.getMasterServices().getMasterQuotaManager().onRegionSplit(this.getParentRegion());
            } catch (QuotaExceededException e) {
              env.getMasterServices().getRegionNormalizer().planSkipped(this.getParentRegion(),
                  NormalizationPlan.PlanType.SPLIT);
              throw e;
            }
          }
            
**Step3** close parent region ：
    
        创建子procedure UnassignProcedure 。接下来的流程会等unassignprocedure完成之后才进行。
        
**Step4** check close parent region :如果WALROOTDIR中的parent  region 存在recovery.edits文件夹（-?[0-9]+ match的文件（去除seqid文件和temp文件（splitWAL thread正在写的文件）））。 这时候尝试reopen这个region并重试step3。
 这些seqid文件到底是什么，WALSplitter 都和他们相关是server crash之后写的文件吧。嗯嗯 这个有很大的可能，回头review下WALSplitter。
 
**Step5** 创建daughter regions：

cleanup parent region 文件夹，并创建.split文件夹。每个StoreFilesSplitter创建对应的refenrence top/bottom文件。并将.split下的daughter Region 文件夹放置到table下。

        
    /**
       * Create daughter regions
       * @param env MasterProcedureEnv
       */
      @VisibleForTesting
      public void createDaughterRegions(final MasterProcedureEnv env) throws IOException {
        final MasterFileSystem mfs = env.getMasterServices().getMasterFileSystem();
        final Path tabledir = FSUtils.getTableDir(mfs.getRootDir(), getTableName());
        final FileSystem fs = mfs.getFileSystem();
        HRegionFileSystem regionFs = HRegionFileSystem.openRegionFromFileSystem(
          env.getMasterConfiguration(), fs, tabledir, getParentRegion(), false);
        regionFs.createSplitsDir();
    
        Pair<Integer, Integer> expectedReferences = splitStoreFiles(env, regionFs);
    
        assertReferenceFileCount(fs, expectedReferences.getFirst(),
          regionFs.getSplitsDir(daughter_1_RI));
        //Move the files from the temporary .splits to the final /table/region directory
        regionFs.commitDaughterRegion(daughter_1_RI);
        assertReferenceFileCount(fs, expectedReferences.getFirst(),
          new Path(tabledir, daughter_1_RI.getEncodedName()));
    
        assertReferenceFileCount(fs, expectedReferences.getSecond(),
          regionFs.getSplitsDir(daughter_2_RI));
        regionFs.commitDaughterRegion(daughter_2_RI);
        assertReferenceFileCount(fs, expectedReferences.getSecond(),
          new Path(tabledir, daughter_2_RI.getEncodedName()));
      }
      
这里为storefile创建refenrence文件。 
      
    private Pair<Path, Path> splitStoreFile(HRegionFileSystem regionFs, byte[] family, HStoreFile sf)
        throws IOException {
        if (LOG.isDebugEnabled()) {
          LOG.debug("pid=" + getProcId() + " splitting started for store file: " +
              sf.getPath() + " for region: " + getParentRegion().getShortNameToLog());
        }
    
        final byte[] splitRow = getSplitRow();
        final String familyName = Bytes.toString(family);
        final Path path_first = regionFs.splitStoreFile(this.daughter_1_RI, familyName, sf, splitRow,
            false, splitPolicy);
        final Path path_second = regionFs.splitStoreFile(this.daughter_2_RI, familyName, sf, splitRow,
           true, splitPolicy);
        if (LOG.isDebugEnabled()) {
          LOG.debug("pid=" + getProcId() + " splitting complete for store file: " +
              sf.getPath() + " for region: " + getParentRegion().getShortNameToLog());
        }
        return new Pair<Path,Path>(path_first, path_second);
      }
          
**Step6** writeMaxSequenceIdFile   以我现在的理解是server crash之后写入的recoveryedit文件。回头确认。这里将miss的信息收集起来然后写到daughter文件夹中。（/home/hbase/hbase2to1/default/TestTable/ba9b2c80ac1dde364c8a3412ce0564ed/recovered.edits/213904.seqid）一版本貌似没有这个要注意下是啥子东东。
        
         private void writeMaxSequenceIdFile(MasterProcedureEnv env) throws IOException {
            FileSystem walFS = env.getMasterServices().getMasterWalManager().getFileSystem();
            long maxSequenceId =
              WALSplitter.getMaxRegionSequenceId(walFS, getWALRegionDir(env, getParentRegion()));
            if (maxSequenceId > 0) {
              WALSplitter.writeRegionSequenceIdFile(walFS, getWALRegionDir(env, daughter_1_RI),
                  maxSequenceId);
              WALSplitter.writeRegionSequenceIdFile(walFS, getWALRegionDir(env, daughter_2_RI),
                  maxSequenceId);
            }
          }
          
**Step7** preSplitRegionBeforeMETA 有关MasterCoprocessorHost这里跳过。

      /**
       * Post split region actions before the Point-of-No-Return step
       * @param env MasterProcedureEnv
       **/
      private void preSplitRegionBeforeMETA(final MasterProcedureEnv env)
          throws IOException, InterruptedException {
        final List<Mutation> metaEntries = new ArrayList<Mutation>();
        final MasterCoprocessorHost cpHost = env.getMasterCoprocessorHost();
        if (cpHost != null) {
          cpHost.preSplitBeforeMETAAction(getSplitRow(), metaEntries, getUser());
          try {
            for (Mutation p : metaEntries) {
              RegionInfo.parseRegionName(p.getRow());
            }
          } catch (IOException e) {
            LOG.error("pid=" + getProcId() + " row key of mutation from coprocessor not parsable as "
                + "region name."
                + "Mutations from coprocessor should only for hbase:meta table.");
            throw e;
          }
        }
      }
    
**Step8** updateMeta 

     /**
       * Add daughter regions to META
       * @param env MasterProcedureEnv
       */
      private void updateMeta(final MasterProcedureEnv env) throws IOException {
        env.getAssignmentManager().markRegionAsSplit(getParentRegion(), getParentRegionServerName(env),
          daughter_1_RI, daughter_2_RI);
      }
      
      
这里为什么需要maxseqid呐？暂时先不考虑。貌似留了两个地方的坑都是和seqid有关系。划个重点吧。

master中此时parent region的状态变为SPLIT，daughter region的状态变为SPLITTING_NEW。

     MetaTableAccessor.splitRegion(master.getConnection(), parent, parentOpenSeqNum, hriA, hriB,
      serverName, getRegionReplication(htd));   //修改meta表
      
    /**
       * Splits the region into two in an atomic operation. Offlines the parent region with the
       * information that it is split into two, and also adds the daughter regions. Does not add the
       * location information to the daughter regions since they are not open yet.
       * @param connection connection we're using
       * @param parent the parent region which is split
       * @param parentOpenSeqNum the next open sequence id for parent region, used by serial
       *          replication. -1 if not necessary.
       * @param splitA Split daughter region A
       * @param splitB Split daughter region B
       * @param sn the location of the region
       */
      public static void splitRegion(Connection connection, RegionInfo parent, long parentOpenSeqNum,
          RegionInfo splitA, RegionInfo splitB, ServerName sn, int regionReplication)
          throws IOException {
        try (Table meta = getMetaHTable(connection)) {
          long time = EnvironmentEdgeManager.currentTime();
          // Put for parent
          Put putParent = makePutFromRegionInfo(RegionInfoBuilder.newBuilder(parent)
                            .setOffline(true)
                            .setSplit(true).build(), time);
          addDaughtersToPut(putParent, splitA, splitB);
    
          // Puts for daughters
          Put putA = makePutFromRegionInfo(splitA, time);
          Put putB = makePutFromRegionInfo(splitB, time);
          if (parentOpenSeqNum > 0) {
            addReplicationBarrier(putParent, parentOpenSeqNum);
            addReplicationParent(putA, Collections.singletonList(parent));
            addReplicationParent(putB, Collections.singletonList(parent));
          }
          // Set initial state to CLOSED
          // NOTE: If initial state is not set to CLOSED then daughter regions get added with the
          // default OFFLINE state. If Master gets restarted after this step, start up sequence of
          // master tries to assign these offline regions. This is followed by re-assignments of the
          // daughter regions from resumed {@link SplitTableRegionProcedure}
          addRegionStateToPut(putA, RegionState.State.CLOSED);
          addRegionStateToPut(putB, RegionState.State.CLOSED);
    
          addSequenceNum(putA, 1, splitA.getReplicaId()); // new regions, openSeqNum = 1 is fine.
          addSequenceNum(putB, 1, splitB.getReplicaId());
    
          // Add empty locations for region replicas of daughters so that number of replicas can be
          // cached whenever the primary region is looked up from meta
          for (int i = 1; i < regionReplication; i++) {
            addEmptyLocation(putA, i);
            addEmptyLocation(putB, i);
          }
    
          byte[] tableRow = Bytes.toBytes(parent.getRegionNameAsString() + HConstants.DELIMITER);
          multiMutate(connection, meta, tableRow, putParent, putA, putB);
        }
      }
      
    这里涉及到多个建表时候的值所以只针对如下的方式进行分析：
    {NAME => 'info0', VERSIONS => '1', EVICT_BLOCKS_ON_CLOSE => 'false', NEW_VERSION_BEHAVIOR => 'false', K
    EEP_DELETED_CELLS => 'FALSE', CACHE_DATA_ON_WRITE => 'false', DATA_BLOCK_ENCODING => 'NONE', TTL => 'FO
    REVER', MIN_VERSIONS => '0', REPLICATION_SCOPE => '0', BLOOMFILTER => 'ROW', CACHE_INDEX_ON_WRITE => 'f
    alse', IN_MEMORY => 'false', CACHE_BLOOMS_ON_WRITE => 'false', PREFETCH_BLOCKS_ON_OPEN => 'false', COMP
    RESSION => 'NONE', BLOCKCACHE => 'true', BLOCKSIZE => '65536', METADATA => {'IN_MEMORY_COMPACTION' => '
    NONE'}}
    
    其中涉及到"REPLICATION_SCOPE   REGION_REPLICATION" 等参数的设置。

![image](B4EA673BBE9D452895617E78FEB90F6F)

![image](ED58B759C9044290A3230CA55AA27F08)   

    如上图的样式一样将对应的信息更新到meta表中。batch mutations operations。

        
    
**Step9** preSplitRegionAfterMETA  MasterCoprocessorHost presplit 这里直接跳过了。
 
**Step10** Assign daughterA 和 daughterB 。
    
        
**Step11** Post split region actions  这个都是MasterCoprocessorHost()干的 这里就不讨论了。

简单来说就和1版本差不多，只不过一些assign和unassign流程采用了procedureV2的方式。
 


