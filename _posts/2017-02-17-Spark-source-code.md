---
published: true
layout: post
title: Spark源码阅读笔记
category: spark
tags: 
  - 分布式系统
time: 2017.02.17 11:36:00
excerpt: Spark源码再次详细阅读笔记。

---

## 一、Spark的启动

Master和Worker启动后会确定Master，启动RPC通信框架（RPCEndPoint），Metric子系统，关联WebUI等。Master会获取每个Worker的资源信息，如内存、核数。具体为Master类和Worker类。

## 二、 作业的提交

提交作业会启动一个新的JVM，即Driver。Driver上执行作业的初始化、通信初始化、作业的调度等操作。

### 1、SparkSubmit
runMain(): SparkSubmit根据--class参数的名字，通过反射来执行用户提交的类的main方法。【1.5以后版本已经不使用deploy.Client类了。】

Spark Application首先会实例化一个SparkContext对象，该对象会初始化一些重要的对象：

（1）StandaloneSchedulerBackend（RPC通信、调度，不同模式下实例化为不同对象）
（2）DAGScheduler（作业划分）
（3）TaskSchedulerImpl（任务管理和任务调度）
（4）BlockManager（数据管理）
（5）MemoryManager（内存管理）

### 2、StandaloneSchedulerBackend
该类的主要作用是与Master通信，获取Executor的信息。具体工作由其中的Client对象完成，Client对象对应的类为AppClient【最新Spark中称为StandAloneAppClient】。AppClient会发送ApplicationDescription给Master，让Master分配资源。具体流程如下：

 - StandaloneSchedulerBackend生成ApplicationDescription，包含了application的名字，对executor的内存、核数的需求等信息。信息关联给APPClient。
 - 启动AppClient，AppClient向Master发送RegisterApplication请求。
 - Master收到RegisterApplication，注册Application，开始分配资源。具体的方法为schedule()方法：洗牌worker，根据worker的资源总量是否够用逐个launchDriver()，最后startExecutorOnWorker。
 - launchDriver，Master通知Worker上LaunchDriver，Worker上启动DriverRunner管理该App对应的Driver。
 - startExecutorOnWorkers，优先满足第一个app(FIFO)，分配内存和核数。核数的分配采取均分到所有worker上的原则，先每个worker上分配1个executor，如果每个worker上核数小于executor的核数（尽管spark配置的总核数大于executor所需核数），不会分配executor。
 - launchExecutor，Master通知Worker上LaunchExecutor，Worker上启动ExecutorRunner来启动Executor。具体的启动方法是：第一步的ApplicationDescription里包含了一个Command，其中记录了启动的Executor的命令；实际上命令中启动的是CoarseExecutorBackend类；CoarseExecutorBackend启动RPC框架，而不是利用Worker通信，发送RegisterExecutorResponse给Driver；Driver返回RegisterExecutor信息；CoarseExecutorBackend创建新的Executor。 
 - ExecutorAdded，Master通知Driver已经添加了Executor。

至此，Executor已经分配到了Woker上，即资源均已经准备完成。StandaloneSchedulerBackend后续承载了Driver与Master、Worker上的通信，包括Executor Lost、Remove Application等消息的传递。用户代码继续往下执行，直到RDD的action操作触发Job的提交，才真正的提交作业。

## 三、作业的执行

通过SparkSubmit提交作业后，SparkContext实例化后，资源的分配已经在配置中决定，因此首先完成资源分配。然后由SparkContext来提交Job，由DAGScheduler来控制Job的执行，TaskScheduler调度和管理Task。

### 1、DAGScheduler

RDD中的Action操作触发SparkContext的runJob方法时，执行的是DAGScheduler的submitJob方法。submit后实际上是发送一个JobSubmit给自己，然后执行handleJobSubmit方法。这个方法的流程是：

 - 创建JobID。
 - 根据提交的RDD（称为FinalRDD，因为该RDD触发了Action操作，前面的RDD均还未执行）创建ResultStage，然后submitStage。
 - 获取提交的Stage的Missing parent Stage，也就是DAG图中指向该Stage的Parent Stage。如果没有Parent Stage，直接提交该Stage；如果有没处理的Parent Stage，继续调用submitStage方法提交Parent Stage。因此，该操作最终会有一个没有未处理的Parent Stage的Stage被提交。调用submitMissingTasks执行该Stage。
 - 提交的Stage根据Partition创建Task，ShuffleMapStage每个partition创建一个ShuffleMapTask，ResultStage每个partition创建一个ResultTask。Task中需要包含以下信息：stage的id，stage的attemptId，用于序列化广播的taskBinary（记录了对应的RDD和shuffleDependency），处理的partition，task执行的偏好（本地，同集群，同机架，任何），Accumulator。
 - 将创建的Tasks封装为TaskSet，交给TaskScheduler调度。

在DAGScheduler的工作中，最主要的还是Stage的划分。划分后的Stage是提交执行的依据，也是划分Task的关键。Job的最后一个Stage是ResultStage，ResultStage的Parent Stage都是ShuffleMapStage。那么，具体如何划分Stage呢？细化一下前文中的第二步和第三步：

 - 调用newResultStage()方法，根据FinalRDD创建ResultStage。
 - 调用getParentStages()方法，获取FinalRDD的Parent Stages。方法是从FinalRDD开始，遍历其Dependencies，如果是ShuffleDependency，则调用getShuffleMapStage()方法从shuffleDependency中获取相应的stage；如果不是shuffleDependency，则继续向前遍历其依赖的RDD。只有找到ShuffledRDD，其Dependency才是shuffleDependency；或者HadoopRDD，没有依赖的RDD。
 - 如何从ShuffleDependency中获取ShuffleMapStage？如果之前已经创建过了，直接返回；如果没有创建过，不仅仅是返回该parent stage，还需要找出shuffleDependency.rdd的所有parent dependencies及祖先dependencies，判断这些dependencies对应的stage是否cache了结果，有cache结果的话需要更新到对应的stage中，然后再返回当前依赖的parent stage。

### 2、TaskSchedulerImpl
TaskScheduler的实际对象，该对象生成后完成以下工作：生成SchedulerBuilder。根据调度模式，FIFO或者FAIR（FAIR下需要修改配置文件），生成调度器。该对象被创建后，SparkContext会调用start()启动推测执行的线程（如果打开了推测执行）。

TaskScheduler开始调度Task是从DAGScheduler调用submitTasks开始的，工作流程如下：

 - 创建TaskSetManager。
 - 通知StandaloneSchedulerBackend发送一个Reviveoffers给自己（StandaloneSchedulerBackend继承自CoarseGrainedSchedulerBackend的方法reviveOffers()）。【最新版本的Spark将消息的接收整体移到了CoarseGrainedSchedulerBackend中】
 - TaskSchedulerImpl为每个Executor生成一个WorkerOffer，记录了executor的id，executor的host和cores。WorkerOffer重新洗牌，根据核数创建TaskDescription的数组，按核数依次填充数组，返回给StandaloneSchedulerBackend。
 - StandaloneSchedulerBackend调用launchTask将每个Executor对应的TaskDescription的数组序列化发送给Executor（通过CoarseExecutorBackend发送），消息类型为LaunchTask。
 - CoarseExecutorBackend接收LaunchTask消息及消息中序列化的TaskDescription，通知Executor执行launchTask()。
 - Executor创建TaskRunner（继承自Runnable）作为执行的Task，放到threadPool中执行。
 
Driver会根据WorkerOffer的信息一条条的发送TaskDescription，Executor最终在线程池中创建core数量的TaskRunner。

## 四、任务的执行

Job根据partition和stage划分为Task后，有两种实例化类型：ShuffleMapTask和ResultTask。Task分发到Executor后执行的流程如下：

 - 每个Task创建自己的TaskMemoryManager。
 - 从序列化的TaskDescription中反序列化出Task信息，这时可以得到Task具体是ShuffleMapTask还是ResultTask。
 - 关联并更新Task的Metrics。
 - 执行Task，返回结果（具体返回什么后面讨论）。
 - 序列化结果，通过更新Task状态返回给Driver。
 - Driver接收StatusUpdate信息，如果是TaskFinish，更新executor的可用核数，重新调用makeOffers()来launchTask。

Task的执行即runTask()方法，根据实例化对象的不同，该方法被重写。

### 1、ShuffleMapTask

ShuffleMapTask定义了一个ShuffleWriter，用于将RDD的计算结果写入磁盘。这时调用的是rdd.iterator()方法，如果有Cache就不需要继续向前计算了，如果没有就调用compute()或者checkpoint()，依次向前遍历。

由于是以Stage为单位提交的，所以，如果不存在cache的话，向前遍历调用RDD的iterator，最终会有两种情况（ResultTask执行时也是这两种情况）：

 - 碰到HadoopRDD读入数据。
 - 碰到Stage的最初始的ShuffledRDD。

HadoopRDD的compute()：

利用RecordReader读入数据，创建(K,V)，构造迭代器InterruptibleInterator返回。

ShuffledRDD的compute()：

 - 获取ShuffleManger（默认为SortShuffleManger），获取BlockStoreShuffleReader。
 - ShuffleReader读取数据。读取数据时通过ShuffleBlockFetchIterator完成，因为有可能有结果不在本地。首先通过MapOutputTracker跟踪该shuffle分散在不同Executor上的partition。
 - 通过ShuffleBlockFetchIterator表示所有的partition块，每块如果不在local则从远程拉取。
 - 如果需要compress则压缩数据，因为这里所有的数据均在内存中，如果很大需要压缩。
 - 反序列化partition中的数据流为KV迭代器。
 - 如果在Map端已经做了Combine操作，即已经根据K做了V的聚合，则直接定义V的类型，如果没有Combine操作则定义V类型为Nothing；如果是aggregator操作，调用aggregator做聚合，聚合使用ExternalAppendOnlyMap（重要）。
 - 如果定义了key ordering，则需要按K值排序，排序使用ExternalSort（重要）。排序算法使用TimSort。

该过程即Shuffle的ShuffleRead阶段。

最终compute返回一个Iterator。ShuffleMapTask在执行完成后写入最后计算的一个RDD，即compute返回的Iterator。通过ShuffleWriter写入ExternalSort，然后写入磁盘。写入时，按partition写入，所有的partition写入一个文件，通过长度偏移定位。这里的关键问题是partition的使用，因为MapOutputTracker利用partitioner决定ShuffleMapTask的输出分为多少了partition。paritioner出现的时刻为RDD执行shuffle操作时，RDD会生成一个新的HashPartitioner，窄依赖关系的操作不会更改partition。

### 2、ResultTask

ResultTask执行时的起始RDD情况和ShuffleMapTask相同，在最后处理结果时有所不同。ResultTask中传入了Function对象，直接在RDD上执行该方法，例如count()，saveAsTxtFile()等。

## 五、作业/任务的调度

CoarseGrainedSchedulerBackend分发TaskDescription时，按照TaskScheduler提供的Tasks来分发，因此调度规则位于TaskScheduler的resourceOffers()方法中，具体为rootPool.getSortedTaskSetQueue()中对TaskSets排序这一部分。在Pool中定义了taskSetSchedulingAlgorithm决定对TaskSet中的task进行排序的comparator，即调度的顺序。

 - FIFOSchedulingAlgorithm：先比较两个TaskSetManager的优先级，相同则比较stageId。
 - FairSchedulingAlgorithm：依次比较minShare（最小启动Task的比例），已经运行的Task数目，TaskSet设置的weight，TaskSetManager的名字。

如果SparkContext定义了多个pool，那么pool之间采用FairSchedulingAlgorithm；pool内部采用FIFO或者Fair。

推测执行机制：如果打开了推测执行机制，那么会启动一个线程定期的检查是否有需要推测执行的Task，方法为TaskSchedulerImpl中的checkSpeculatableTasks()方法。该方法在TaskSetManager中查找运行时间大于阈值/已经执行完Task的中位数时间，的Task，标记为SpeculatableTasks。为SpeculatbleTask提供资源启动副本任务。

## 六、数据管理

在SparkContext初始化时，会初始化BlockManager（env.blockManager）来管理RDD所对应的数据。每个Executor上都会启动一个BlockManager，BlockManager同时会初始化一个BlockTransferService，用来拉取远程的Block。

Task的执行实际上是根据partitionID来获取输入数据和写入输出数据，其本质上是通过BlockManager获取的。具体关联流程如下：

 - Task执行RDD的某个partition，调用RDD.getOrCompute()方法。
 - 依据partitionId生成RDDBlockId，与partitionId对应。
 - 调用BlockManager.getOrElseUpdate()方法，参数中指定该block的storageLevel（与归属的RDD相同）。
 - 如果已经存在的Block，直接返回改block；否则，计算该RDD，调用doPutIterator()将迭代器记录进BlockManager，然后返回迭代器。
 
BlockManager针对的是ShuffleMapTask的最后一个RDD，所以中间计算的RDD并不会分配给BlockManager存储。这里写入BlockManager与ShuffleWriter的写入的区别是：RDD计算出来后就会在BlockManager中更新BlockId的信息，然后返回这个结果Iterator，然后给ShuffleWriter写入磁盘。

在沿RDD链向上计算的过程中，如果RDD的StorageLevel中有useMemory，即需要persist/cache到内存，需要MemoryStore将数据存储到内存，调用的是MemoryStore.putIteratorAsValues()方法。MemoryStore向MemoryManager申请StorageMemory来将该block保存到内存，保存的结构是一个LinkedHashMap&lt;BlockId, MemoryEntry>类型，BlockId对应persist的block，MemoryEntry对应cache的数据。【目前最新的master分支上没有使用CacheManager来管理Cache的数据了。】

## 七、内存管理

前文中有两处地方提到了内存管理，一个是在Executor上生成Task时会初始化一个TaskMemoryManager，一个是MemoryStore持久化block时向MemoryManager申请内存。MemoryManager是在SparkContext的SparkEnv中实例化的，默认的实例化对象为UnifiedMemoryManager（统一内存管理）。【1.6以前版本默认为StaticMemoryManger静态内存管理。】

MemoryManager通过两个Pool：StorageMemoryPool和ExecutionMemoryPool来控制Memory的分配。同时，MemoryManager保证N个运行的Task中，每个Task所占用的内存大于1/2N而小于1/N。

 - StorageMemoryPool与MemoryStore关联，MemoryStore中每次cache block或者drop block时都会首先请求MemoryManager，如果StorageMemoryPool总空间不够，会从ExecutionMemoryPool划分（StaticMemoryManager不会动态划分）。如果StorageMemoryPool空间空闲部分不够会释放不需要的block，然后分配并更新StorageMemoryPool中memoryUsed的值。
 - 在Memory中有一部分unroll memory，用来存放临时的block数据。BlockManager将这block持久化到内存时需要这一部分空间，UnifiedMemoryManager将这部分空间的请求归属到了StorageMemoryPool中。
 - ExecutionMemoryPool的大小除了分配给StorageMemoryPool会动态更新外，外界不会主动更新该值。
 - UNSAFE/Tungsten模式下，MemoryManager会利用page来管理block数据，TaskMemoryManager通过请求ExecutionMemoryPool中的page来获取偏移量。

TaskMemoryManager如何使用MemoryManager呢？非Tungsten模式下，TaskMemoryManager并没有显式的调用acquire的一些方法，也就是说，实际上MemoryManager除了管理StorageMemoryPool和ExecutionMemoryPool的大小动态变化，以及整体上适应JVM的堆大小，并没有很细粒度的管理。而TaskMemoryManager中的getMemoryConsumptionForThisTask()实际上就只是一个按1/2N和1/N的比例依次递增Task的消耗而已，并没有实际的代表意义。
  