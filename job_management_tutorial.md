
# DolphinDB作业管理

作业（Job）是DolphinDB中最基本的执行单位，可以简单理解为一段DolphinDB脚本代码在DolphinDB系统中的一次执行。Job根据阻塞与否可分成同步作业和异步作业。

### 同步作业

也称为交互式作业（Interactive Job），同步任务的主要来源：

- web notebook
- DolphinDB GUI
- DolphinDB命令行界面
- 通过DolphinDB提供的各个编程语言API接口

由于这种类型的作业对实时性要求较高，DolphinDB在执行过程中会自动给予较高的优先级，使其更快地得到计算资源。

### 异步作业

异步作业是在DolphinDB后台执行的作业，包括：

- 通过submitJob/submitJobEx函数提交的[**批处理作业**](https://www.dolphindb.cn/cn/help/BatchJobManagement.html)。
- 通过scheduleJob函数提交的[**定时作业**](https://www.dolphindb.cn/cn/help/ScheduleJobs.html)。
- Streaming 作业。

这类任务一般对结果的实时反馈要求较低，且需要长期执行，DolphinDB一般会给予较低的优先级。

### 子任务

在DolphinDB中，若数据表数据量过大，一般都需要进行[分区处理](https://www.dolphindb.cn/cn/help/DistributedDatabase.html)。如果一个Job A里含有分区表的查询计算任务（如SQL查询），将会分解成多个子任务并送到不同的节点上并行执行，等待子任务执行完毕之后，再合并结果，继续Job A的执行。类似的，DolphinDB[分布式计算](https://www.dolphindb.cn/cn/help/DistributedComputing.html)也会产生子任务。因此，Job也可以理解成一系列的子任务。

### Worker与Executor

DolphinDB是一个P2P架构的系统，即每一个Data Node的角色都是相同的，它们都可以执行来自用户提交的Job，而因为一个Job可能产生子任务，每个Data Node需要有负责Job内部执行的调度者，我们称它为Worker，它负责处理用户提交的Job，简单计算任务的执行，并执行Job的任务分解，任务分发，并汇集最终的执行结果。Job中分解出来的子任务将会被分发到集群中的Data Node上（也有可能是本地Data Node），并由Data Node上的Worker/Executor线程负责执行。

具体Worker与executor在执行job的时候主要有以下几种情况：

1. 当一个表没有进行分区，对其查询的job将会有worker线程执行掉。

2. 当一个表被分区存放在单机上时候，对其的查询Job可能会分解成多个子任务，并由该节点上的多个executor线程执行，达到并行计算的效果。

3. 当一个表被分区存储在DFS时，对其查询的Job可能会被分解成多个子任务，这些子任务会被分发给其他node的worker上执行，达到分布式计算的效果。



为了最大化性能，DolphinDB会将子任务发送到数据所在的Data Node上执行，以减少网络传输开销。比如：

- 对于存储在DFS中的分区表，worker将会根据分区模式以及分区当前所在Data Node来进行任务分解与分发。
- 对于分布式计算，worker将会根据数据源信息，发送子任务到相应的数据源Data Node执行。

### Job调度

#### Job优先级

在DolphinDB中，Job是按照优先级进行调度的，优先级的取值范围为0-9，取值越高优先级则越高。对于优先级高的Job，系统会更及时地给与计算资源。每个Job一般默认会有一个default priority，取值为4，然后根据job的类型又会有所调整。

#### Job调度策略

基于Job的优先级，DolphinDB设计了多级反馈队列来调度Job的执行。具体来说，系统维护了10个队列，分别对应10个优先级，系统总是分配线程资源给高优先级的job，对于处于相同优先级的Job，系统会以round robin的方式分配线程资源给job；当一个优先级队列为空的时候，才会处理低优先级的队列中的job。

#### Job并行度

由于一个Job可能会分成多个并行子任务，DolphinDB的Job还拥有一个并行度parallelism，表示在一个Data Node上，将会最多同时用多少个线程来执行job产生的并行任务，默认取值为2，可以认为是一种时间片单位。举个例子，若一个Job的并行度为2，job产生了100个并行子任务，那么Job被调度的时候系统只会分配2个线程用于子任务的计算，因此需要50轮调度才能完成整个job的执行。

#### Job优先级的动态变化

为了防止处于低优先级的Job被长时间饥饿，DolphinDB会适当降低Job的优先级。具体的做法是，当一个job的时间片被执行完毕后，如果存在比其低优先级的Job，那么将会自动降低一级优先级。当优先级到达最低点后，又回到初始的优先级。因此低优先级的任务迟早会被调度到，解决了饥饿问题。

#### 设置Job的优先级

DolphinDB的Job的优先级可以通过以下方式来设置：

- 对于console、web notebook以及API提交上来的都属于interactive job，其优先级取值为min(4，一个可调节的用户最高优先级)，因此可以通过改变用户自身的优先级值来调整。
- 对于通过submitJob提交上的batch job，系统会给与default priority，即为4。你也可以使用[submitJobEx](https://www.dolphindb.cn/cn/help/submitJobEx.html?search=submitJobEx)函数来提供用户指定的优先级。
- 定时任务的优先级无法改变，默认为4。



### 计算容错

DolphinDB的分布式计算含有一定的容错性，主要得益于分区副本冗余存储。当一个子任务被发送到一个分区副本节点上之后，若节点出现故障或者分区副本发生了数据校验错误(副本损坏），Job Scheduler(即某个Data Node的一个worke线程)将会发现这个故障，并且选择该分区的另一个副本节点，重新执行子任务。你可以通过[设置dfsReplicationFactor](https://www.dolphindb.cn/cn/help/ClusterSetup.html?search=replication)来调整这种冗余度。



### 计算与存储耦合以及作业之间的数据共享

DolphinDB的计算是尽量靠近存储的。DolphinDB之所以不采用计算存储分离，主要有以下几个原因：

1. 计算与存储分离会出现数据冗余。考虑存储于计算分离的Spark+Hive架构，spark应用程序之间是不共享存储的。若N个spark应用程序从Hive读取某个表T的数据，那么首先T要加载到N个spark应用程序的内存中，存在N份，这将造成机器内存的的浪费。在多用户场景下，比如一份tick数据可能会被多个分析人员共享访问，如果采取Spark那种模式，将会提高IT成本。

2. 拷贝带来的延迟问题。虽然说现在数据中心逐渐配备了RDMA，NVMe等新硬件，网络延迟和吞吐已经大大提高。但是这主要还是在数据中心，DolphinDB系统的部署环境可能没有这么好的网络环境以及硬件设施，数据在网络之间的传输会成为严重的性能瓶颈。

综上这些原因，DolphinDB采取了计算与存储耦合的架构。具体来说：

1. 对于内存浪费的问题，DolphinDB的解决方案是Job（对应Spark应用程序）之间共享数据。在数据经过分区存储到DolphinDB的DFS中之后，每个分区的副本都会有自己所属的节点，在一个节点上的分区副本将会在内存中只存在一份。当多个Job的子任务都涉及到同一个分区副本时，该分区副本在内存中可以被共享地读取，减少了内存的浪费。

2. 对于拷贝带来的延迟问题，DolphinDB的解决方案是将计算发送到数据所在的节点上。一个Job根据DFS的分区信息会被分解成多个子任务，发送到分区所在的节点上执行。因为发送计算到数据所在的节点上相当于只是发送一段代码，网络开销大大减少。