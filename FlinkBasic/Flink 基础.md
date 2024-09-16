
## API 分层

下图是基本的 Flink 分层 API: 

![[Flink'sApis.png]]



 Flink API 的分层架构：

1. **最低层抽象（ProcessFunction）**: 提供最灵活的事件处理和状态管理，可以直接控制流处理，适用于需要复杂计算的场景。
   
2. **核心 API（DataStream API 和 DataSet API）**: DataStream API 处理有界和无界流，支持常见的数据处理操作，如转换、聚合、窗口、状态管理等。DataSet API 提供额外的操作，用于有界数据集的循环和迭代。

3. **声明式 API（Table API）**: 基于表的 DSL，遵循关系模型，提供与 SQL 类似的表操作，简化了复杂逻辑的实现。虽然表达能力低于核心 API，但代码量较少，并通过优化器提高性能。

4. **最高层抽象（SQL）**: 类似 Table API 的抽象，通过 SQL 查询表达式来实现流处理，与 Table API 紧密集成，可用于表上执行 SQL 查询。

这些分层架构使得 Flink 既能满足对低层控制的需求，又能通过简洁的声明式 API 实现复杂的流处理逻辑。

## 有状态的流计算


## Flink 运行时架构

如下图是Flink运行时架构图: 



### 作业管理器（JobManager) 

JobManager是一个Flink集群中任务管理和调度的核心，是控制应用执行的主进程。也就是说，每个应用都应该被唯一的JobManager所控制执行。

JobManger又包含3个不同的组件。

#### JobMaster

JobMaster 是 JobManager 中最核心的组件，负责处理单独的作业（Job）。所以 JobMaster 和具体的 Job 是一一对应的，多个 Job 可以同时运行在一个 Flink 集群中, 每个 Job 都有一个自己的 JobMaster 。需要注意在早期版本的 Flink 中，没有 JobMaster 的概念；而 JobManager 的概念范围较小，实际指的就是现在所说的 JobMaster 。

在作业提交时， JobMaster 会先接收到要执行的应用。 JobMaster 会把 JobGraph 转换成一个物理层面的数据流图，这个图被叫作“执行图”（ExecutionGraph），它包含了所有可以并发执行的任务。 JobMaster 会向资源管理器（ResourceManager）发出请求，申请执行任务必要的资源。一旦它获取到了足够的资源，就会将执行图分发到真正运行它们的 TaskManager 上。

而在运行过程中， JobMaster 会负责所有需要中央协调的操作，比如说检查点（checkpoints）的协调。

#### ResourceManager

ResourceManager 主要负责资源的分配和管理，在 Flink  集群中只有一个。所谓“资源”，主要是指TaskManager的任务槽（task slots）。任务槽就是Flink集群中的资源调配单元，包含了机器用来执行计算的一组 CPU 和内存资源。每一个任务（Task）都需要分配到一个 slot 上执行。

这里注意要把 Flink 内置的 ResourceManager 和其他资源管理平台（比如YARN）的 ResourceManager 区分开。

#### Dispatcher

Dispatcher主要负责提供一个REST接口，用来提交应用，并且负责为每一个新提交的作业启动一个新的JobMaster 组件。Dispatcher也会启动一个Web UI，用来方便地展示和监控作业执行的信息。Dispatcher在架构中并不是必需的，在不同的部署模式下可能会被忽略掉。

### TaskManager

TaskManager是Flink中的工作进程，数据流的具体计算就是它来做的。Flink集群中必须至少有一个TaskManager；每一个TaskManager都包含了一定数量的任务槽（task slots）。Slot是资源调度的最小单位，slot的数量限制了TaskManager能够并行处理的任务数量。

启动之后，TaskManager会向资源管理器注册它的slots；收到资源管理器的指令后，TaskManager就会将一个或者多个槽位提供给JobMaster调用，JobMaster就可以分配任务来执行了。

在执行过程中，TaskManager可以缓冲数据，还可以跟其他运行同一应用的TaskManager交换数据。

## 核心概念

### 并行度 Parallelism

当要处理的数据量非常大时，我们可以把一个算子操作，“复制”多份到多个节点，数据来了之后就可以到其中任意一个执行。这样一来，一个算子任务就被拆分成了多个并行的“子任务”（subtasks），再将它们分发到不同节点，就真正实现了并行计算
Flink执行过程中，每一个算子（operator）可以包含一个或多个子任务（operator subtask），这些子任务在不同的线程、不同的物理机或不同的容器中完全独立地执行。
一个特定算子的子任务（subtask）的个数被称之为其并行度（parallelism）。这样，包含并行子任务的数据流，就是并行数据流，它需要多个分区（stream partition）来分配并行任务。一般情况下，一个流程序的并行度，可以认为就是其所有算子中最大的并行度。一个程序中，不同的算子可能具有不同的并行度。

#### 设置并行度

在Flink中，可以用不同的方法来设置并行度，它们的有效范围和优先级别也是不同的。

1. 代码中针对算子设置并行度
我们在代码中，可以很简单地在算子后跟着调用setParallelism()方法，来设置当前算子的并行度：

```java
stream.map(word -> Tuple2.of(word, 1L)).setParallelism(2);
```
这种方式设置的并行度，只针对当前算子有效。
2. 代码中针对全局设置并行度
另外，我们也可以直接调用执行环境的setParallelism()方法，全局设定并行度：
```java
env.setParallelism(2);
```
这样代码中所有算子，默认的并行度就都为2了。我们一般不会在程序中设置全局并行度，因为如果在程序中对全局并行度进行硬编码，会导致无法动态扩容。

这里要注意的是，由于keyBy不是算子，所以无法对keyBy设置并行度。
3. 提交应用时通过命令行配置并行度
在使用flink run命令提交应用时，可以增加-p参数来指定当前应用程序执行的并行度，它的作用类似于执行环境的全局设置：
```bash
bin/flink run –p 2 –c com.atguigu.wc.SocketStreamWordCount ./FlinkTutorial-1.0-SNAPSHOT.jar
```
4. 配置文件中配置
我们还可以直接在集群的配置文件flink-conf.yaml中直接更改默认并行度：
```yaml
parallelism.default: 2
```
这个设置对于整个集群上提交的所有作业有效，初始值为1。无论在代码中设置、还是提交时的-p参数，都不是必须的；所以在没有指定并行度的时候，就会采用配置文件中的集群默认并行度。在开发环境中，没有配置文件，默认并行度就是当前机器的CPU核心数。

### 算子链 Operator Chain

#### 一对一 One To One forwarding

这种模式下，数据流维护着分区以及元素的顺序。比如图中的source和map算子，source算子读取数据之后，可以直接发送给map算子做处理，它们之间不需要重新分区，也不需要调整数据的顺序。这就意味着map 算子的子任务，看到的元素个数和顺序跟source 算子的子任务产生的完全一样，保证着“一对一”的关系。map、filter、flatMap等算子都是这种one-to-one的对应关系。这种关系类似于Spark中的窄依赖。

#### 重分区 Redistributing

在这种模式下，数据流的分区会发生改变。比如图中的map和后面的keyBy/window算子之间，以及keyBy/window算子和Sink算子之间，都是这样的关系。

每一个算子的子任务，会根据数据传输的策略，把数据发送到不同的下游目标任务。这些传输方式都会引起重分区的过程，这一过程类似于Spark中的shuffle。

#### 合并算子链

在Flink中，并行度相同的一对一（one to one）算子操作，可以直接链接在一起形成一个“大”的任务（task），这样原来的算子就成为了真正任务里的一部分，如下图所示。每个task会被一个线程执行。这样的技术被称为“算子链”（Operator Chain）。

上图中Source和map之间满足了算子链的要求，所以可以直接合并在一起，形成了一个任务；因为并行度为2，所以合并后的任务也有两个并行子任务。这样，这个数据流图所表示的作业最终会有5个任务，由5个线程并行执行。

将算子链接成task是非常有效的优化：可以减少线程之间的切换和基于缓存区的数据交换，在减少时延的同时提升吞吐量。
Flink默认会按照算子链的原则进行链接合并，如果我们想要禁止合并或者自行定义，也可以在代码中对算子做一些特定的设置：
```java
// 禁用算子链

.map(word -> Tuple2.of(word, 1L)).disableChaining();

// 从当前算子开始新链

.map(word -> Tuple2.of(word, 1L)).startNewChain()
```

### Task Slots

Flink中每一个TaskManager都是一个JVM进程，它可以启动多个独立的线程，来并行执行多个子任务（subtask）。

很显然，TaskManager的计算资源是有限的，并行的任务越多，每个线程的资源就会越少。那一个TaskManager到底能并行处理多少个任务呢？为了控制并发量，我们需要在TaskManager上对每个任务运行所占用的资源做出明确的划分，这就是所谓的任务槽（task slots）。

每个任务槽（task slot）其实表示了TaskManager拥有计算资源的一个固定大小的子集。这些资源就是用来独立执行一个子任务的。

#### 配置 Task Slots

在Flink的/opt/module/flink-1.17.0/conf/flink-conf.yaml配置文件中，可以设置TaskManager的slot数量，默认是1个slot。

```
taskmanager.numberOfTaskSlots: 8
```

需要注意的是，slot目前仅仅用来隔离内存，不会涉及CPU的隔离。在具体应用时，可以将slot数量配置为机器的CPU核心数，尽量避免不同任务之间对CPU的竞争。这也是开发环境默认并行度设为机器CPU数量的原因。

#### 不同的任务对 Task Slots 的共享

默认情况下，Flink是允许子任务共享slot的。如果我们保持sink任务并行度为1不变，而作业提交时设置全局并行度为6，那么前两个任务节点就会各自有6个并行子任务，整个流处理程序则有13个子任务。如上图所示，只要属于同一个作业，那么对于不同任务节点（算子）的并行子任务，就可以放到同一个slot上执行。所以对于第一个任务节点source→map，它的6个并行子任务必须分到不同的slot上，而第二个任务节点keyBy/window/apply的并行子任务却可以和第一个任务节点共享slot。

当我们将资源密集型和非密集型的任务同时放到一个slot中，它们就可以自行分配对资源占用的比例，从而保证最重的活平均分配给所有的TaskManager。

slot共享另一个好处就是允许我们保存完整的作业管道。这样一来，即使某个TaskManager出现故障宕机，其他节点也可以完全不受影响，作业的任务可以继续执行。

当然，Flink默认是允许slot共享的，如果希望某个算子对应的任务完全独占一个slot，或者只有某一部分算子共享slot，我们也可以通过设置“slot共享组”手动指定：

```java
.map(word -> Tuple2.of(word, 1L)).slotSharingGroup("1");
```

这样，只有属于同一个slot共享组的子任务，才会开启slot共享；不同组之间的任务是完全隔离的，必须分配到不同的slot上。在这种场景下，总共需要的slot数量，就是各个slot共享组最大并行度的总和。

### 任务槽和并行度的关系

任务槽和并行度都跟程序的并行执行有关，但两者是完全不同的概念。简单来说任务槽是静态的概念，是指TaskManager具有的并发执行能力，可以通过参数taskmanager.numberOfTaskSlots进行配置；而并行度是动态概念，也就是TaskManager运行程序时实际使用的并发能力，可以通过参数parallelism.default进行配置。

**举例说明：**假设一共有3个TaskManager，每一个TaskManager中的slot数量设置为3个，那么一共有9个task slot，表示集群最多能并行执行9个同一算子的子任务。

而我们定义word count程序的处理操作是四个转换算子：

source→ flatmap→ reduce→ sink

