# 优化检查点和大状态

## 概述

要使Flink应用程序可以大规模可靠地运行，必须满足两个条件：

* 应用程序需要能够可靠地获取检查点
* 在发生故障之后，资源需要足以赶上输入数据流

第一部分讨论如何大规模地获得良好的检查点。最后一节介绍了有关规划使用多少资源的一些最佳实践。

## 监控状态和检查点

监视检查点行为的最简单方法是通过UI的检查点部分。[检查点监视](https://ci.apache.org/projects/flink/flink-docs-release-1.7/monitoring/checkpoint_monitoring.html)的文档展示了如何访问可用的检查点指标。

扩大检查点时特别需要注意的两个数字是：

* 操作员启动检查点的时间：此时间目前尚未直接公开，但对应于：

  `checkpoint_start_delay = end_to_end_duration - synchronous_duration - asynchronous_duration`

  当触发检查点的时间一直很高时，这意味着_检查站障碍_需要很长时间才能从源头传送到操作符。这通常表明系统在恒定的背压下运行。

* 在对齐期间缓冲的数据量。对于精确一次的语义，Flink 在接收多个输入流的操作符处_对齐_流，为该对齐缓冲一些数据。缓冲的数据量理想地较低 - 较高的数量意味着在不同的输入流的非常不同的时间接收检查点障碍。

请注意，当存在瞬态背压，数据偏斜或网络问题时，此处指示的数字偶尔会很高。但是，如果数字一直很高，则意味着Flink将许多资源用于检查点。

## 调整检查点

应用程序可以定期触发检查点。当检查点比检查点间隔花费更长时间时，在正在进行的检查点完成之前不会触发下一个检查点。默认情况下，一旦正在进行的检查点完成，将立即触发下一个检查点。

当检查点经常花费比基本间隔更长的时间时\(例如，由于状态增长比计划的更大，或者检查点存储的存储空间暂时变慢\)，系统就会不断地接受检查点\(新的检查点在进行中一旦完成就会立即启动\)。这可能意味着太多的资源在检查点中被不断地占用，而操作符取得的进展太少。这种行为对使用异步检查点状态的流应用程序影响较小，但仍然可能对应用程序的整体性能产生影响。

为防止出现这种情况，应用程序可以定义_检查点之间_的_最短持续时间_：

`StreamExecutionEnvironment.getCheckpointConfig().setMinPauseBetweenCheckpoints(milliseconds)`

此持续时间是在最新检查点结束和下一个检查点开始之间必须经过的最小时间间隔。下图说明了这对检查点的影响。

![](../../.gitbook/assets/image%20%2812%29.png)

_注意：_可以配置应用程序（通过`CheckpointConfig`）以允许多个检查点同时进行。对于Flink中具有大状态的应用程序，这通常会将太多资源绑定到检查点。当手动触发保存点时，它可能正在与正在进行的检查点同时进行。

## 调整网络缓冲区

在Flink 1.3之前，增加的网络缓冲区数量也导致检查点时间增加，因为保留更多的运行数据意味着检查点障碍被延迟。从Flink 1.3开始，每个传出/传入通道使用的网络缓冲区数量是有限的，因此可以配置网络缓冲区而不会影响检查点时间（请参阅[网络缓冲区配置](https://ci.apache.org/projects/flink/flink-docs-release-1.7/ops/config.html#configuring-the-network-buffers)）。

## 尽可能使状态检查点异步化

当状态是_异步_快照时，检查点的伸缩性比状态被同步快照时好。特别是在具有多个连接，协同功能或窗口的更复杂的流应用程序中，这可能会产生深远的影响。

要异步创建状态，应用程序必须做两件事：

1. 使用[由Flink](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/stream/state/state.html)管理的状态：托管状态表示Flink提供存储状态的数据结构。目前，这是真正的_键控状态_，这就好比接口背后抽象`ValueState`，`ListState`，`ReducingState`，...
2. 使用支持异步快照的状态后端。在Flink 1.2中，只有RocksDB状态后端使用完全异步快照。从Flink 1.3开始，基于堆的状态后端也支持异步快照。

以上两点意味着大状态通常应保持为键控状态，而不是操作符状态。

## 调整RocksDB

许多大型Flink流应用程序的状态存储工作负载是RocksDB状态后端。后端可扩展到主内存之外，并可靠地存储大型键控状态。

不幸的是，RocksDB的性能可能会随着配置的不同而变化，而且几乎没有关于如何正确调优RocksDB的文档。例如，默认配置是针对ssd定制的，并且在旋转磁盘上执行次优配置。

### **增量检查点**

\*\*\*\*

```text
    RocksDBStateBackend backend =
        new RocksDBStateBackend(filebackend, true);
```

### **RocksDB计时器**

### **将选项传递给RocksDB**

```java
RocksDBStateBackend.setOptions(new MyOptions());

public class MyOptions implements OptionsFactory {

    @Override
    public DBOptions createDBOptions(DBOptions currentOptions) {
    	return currentOptions.setIncreaseParallelism(4)
    		   .setUseFsync(false);
    }
    		
    @Override
    public ColumnFamilyOptions createColumnOptions(ColumnFamilyOptions currentOptions) {
    	return currentOptions.setTableFormatConfig(
    		new BlockBasedTableConfig()
    			.setBlockCacheSize(256 * 1024 * 1024)  // 256 MB
    			.setBlockSize(128 * 1024));            // 128 KB
    }
}
```

### **预定义选项**

Flink为不同设置的RocksDB提供了一些预定义的选项集合，例如可以通过`RocksDBStateBackend.setPredefinedOptions(PredefinedOptions.SPINNING_DISK_OPTIMIZED_HIGH_MEM)`来设置这些集合。

我们希望随着时间的推移积累更多这样的配置文件。当您发现一组对某些工作负载工作得很好并且具有代表性的选项时，可免费贡献这些预定义的选项配置文件。

{% hint style="info" %}
**注意：**RocksDB是一个本机库，它直接从进程分配内存，而不是从JVM分配内存。分配给RocksDB的任何内存都必须考虑在内，通常是通过将任务管理器的JVM堆大小减少相同的数量。如果不这样做，可能会导致Yarn/Mesos/等终止JVM进程，以分配比配置更多的内存。
{% endhint %}

## 容量规划

本节讨论如何确定应该使用多少资源来使Flink作业可靠地运行。容量规划的基本经验法则是：

* 正常运行应具有足够的容量，以便在恒定的_背压下_不运行。有关如何检查应用程序是否在背压下运行的详细信息，请参阅[背压监控](https://ci.apache.org/projects/flink/flink-docs-release-1.7/monitoring/back_pressure.html)。
* 在无故障时间内无需背压即可运行程序所需的资源之上提供一些额外资源。需要这些资源来“赶上”在应用程序恢复期间累积的输入数据。应该取决于恢复操作通常需要多长时间（这取决于故障转移时需要加载到新TaskManagers中的状态的大小）以及该方案需要多快才能恢复。

  _重要提示_：应该在激活检查点的情况下建立基线，因为检查点会占用一些资源（例如网络带宽）。

* 临时背压通常是正常的，在负载峰值期间、追赶阶段或外部系统（写入水槽中）出现临时减速时，执行流控制的一个重要部分。
* 某些操作（如大窗口）会导致下游操作员出现尖峰负载：对于窗口，下游操作符在构建窗口时可能没什么可做的，并且在窗口发出时有负载。下游并行度的规划需要考虑窗口发射的程度以及需要处理这种尖峰的速度。

**要点：**为了以后允许添加资源，请确保将数据流程序的_最大并行_度设置为合理的数字。最大并行度定义了在重新缩放程序时（通过保存点）设置程序并行度的高度。

Flink的内部bookkeeping以最大并行度的粒度跟踪多个键组的并行状态。Flink的设计力求使最大并行度值非常高的程序效率更高，即使执行的程序并行度很低。

## 压缩

Flink为所有检查点和保存点提供可选压缩（默认：关闭）。目前，压缩始终使用[snappy压缩算法（版本1.1.4），](https://github.com/xerial/snappy-java)但我们计划在将来支持自定义压缩算法。压缩适用于键控状态下的键组的粒度，即每个键组可以单独解压缩，这对于重新缩放很重要。

压缩可以通过以下方式激活`ExecutionConfig`：

```text
		ExecutionConfig executionConfig = new ExecutionConfig();
		executionConfig.setUseSnapshotCompression(true);
```

{% hint style="info" %}
**注意：**压缩选项对增量快照没有影响，因为它们使用的是RocksDB的内部格式，它始终使用开箱即用的快速压缩。
{% endhint %}

## 任务本地恢复

### 动机

在Flink的检查点中，每个任务都会生成其状态的快照，然后将其写入分布式存储。每个任务通过发送描述分布式存储中状态位置的句柄来确认成功将状态写入作业管理器。反过来，作业管理器从所有任务中收集句柄并将它们捆绑到一个检查点对象中。

在恢复的情况下，作业管理器打开最新的检查点对象并将句柄发送回相应的任务，然后可以从分布式存储中恢复其状态。使用分布式存储来存储状态有两个重要的优点。首先，存储是容错的，其次，分布式存储中的所有状态都可被所有节点访问，并且可以容易地重新分配（例如，用于重新分级）。

但是，使用远程分布式存储也有一个很大的缺点：所有任务必须通过网络从远程位置读取其状态。在许多情况下，恢复可以将失败的任务重新安排到与上一次运行相同的任务管理器（当然还有例如机器故障），但我们仍然必须读取远程状态。这可能导致_大型状态的恢复时间很长_，即使单个机器上只有很小的故障。

### 途径

任务本地状态恢复完全针对这个长恢复时间的问题，主要思想如下：对于每个检查点，每个任务不仅将任务状态写入分布式存储，而且还保留_状态快照的辅助副本，任务本地的存储_（例如，在本地磁盘或内存中）。请注意，快照的主存储必须仍然是分布式存储，因为本地存储不能确保节点故障下的持久性，也不能为其他节点提供访问以重新分发状态，此功能仍需要主副本。

但是，对于可以重新安排到先前位置进行恢复的每个任务，我们可以从辅助本地副本恢复状态，并避免远程读取状态的成本。鉴于_许多故障不是节点故障，并且节点故障通常一次只影响一个节点或极少数节点，_很可能在恢复中大多数任务可以返回到其先前的位置并且完整地找到它们的本地状态。这使得本地恢复有效地缩短了恢复时间。

请注意，根据所选的状态后端和检查点策略，每个检查点可能会产生一些额外费用，用于创建和存储辅助本地状态副本。例如，在大多数情况下，实现将简单地将对分布式存储的写入复制到本地文件。

![](../../.gitbook/assets/image%20%283%29.png)

### 主（分布式存储）和辅助（任务 - 本地）状态快照的关系

任务本地状态通常被认为是一个辅助副本，检查点状态的基本事实是分布式存储中的主副本。这对检查点和恢复期间的本地状态问题有影响:

* 对于检查点，_主副本必须成功，_并且生成_辅助本地副本的失败不会_使检查点_失败_。如果无法创建主副本，则检查点将失败，即使已成功创建辅助副本也是如此。
* 只有主副本由作业管理器确认和管理，辅助副本由任务管理器拥有，其生命周期可以独立于其主副本。例如，可以将3个最新检查点的历史记录保留为主副本，并仅保留最新检查点的任务本地状态。
* 对于恢复，如果匹配的辅助副本可用，Flink将始终先_尝试从任务本地状态恢复_。如果从辅助副本恢复期间出现任何问题，Flink将_透明地重试从主副本恢复任务_。如果主副本和（可选）辅助副本失败，则恢复仅失败。在这种情况下，根据配置，Flink仍然可以回退到较旧的检查点。
* 任务本地副本可能仅包含完整任务状态的一部分（例如，在写入一个本地文件时出现异常）。在这种情况下，Flink将首先尝试在本地恢复本地部分，从主副本恢复非本地状态。主状态必须始终完整，并且是_任务本地状态_的_超集_。
* 任务本地状态可以具有与主状态不同的格式，它们不需要完全相同的字节。例如，任务本地状态甚至可能是由堆对象组成的内存，而不是存储在任何文件中。
* 如果任务管理器丢失，则其所有任务中的本地状态将丢失。

### 配置任务本地恢复

默认情况下，任务本地恢复被停用，可以通过Flink的配置使用`checkpointOptions.local_recovery`中指定的key `state.backend.local-recovery`来激活。此设置的值可以为_true_，也可以为_false_（默认值）以禁用本地恢复。

### 有关不同状态后端的任务本地恢复的详细信息

_**限制**：目前，任务本地恢复仅涵盖键控状态后端。键控状态通常是状态最大的部分。在不久的将来，我们还将涵盖操作符的状态和计时器。_

以下状态后端可以支持任务本地恢复。

* FsStateBackend：键控状态支持任务本地恢复。实现将状态复制到本地文件。这可能会引入额外的写入成本并占用本地磁盘空间。将来，我们还可能提供一种实现，将任务本地状态保存在内存中。
* RocksDBStateBackend：键控状态支持任务本地恢复。对于_完整检查点_，状态将复制到本地文件。这可能会引入额外的写入成本并占用本地磁盘空间。对于_增量快照_，本地状态基于RocksDB的本地检查点机制。此机制也用作创建主副本的第一步，这意味着在这种情况下，不会引入额外的成本来创建辅助副本。我们只是保留原生检查点目录，而不是在上传到分布式商店后删除它。此本地副本可以与RocksDB的工作目录共享活动文件（通过硬链接），因此对于活动文件，也不会为使用增量快照的任务本地恢复消耗额外的磁盘空间。使用硬链接还意味着RocksDB目录必须与可用于存储本地状态的所有配置本地恢复目录位于同一物理设备上，否则建立硬链接可能会失败（请参阅FLINK-10954）。 目前，当RocksDB目录配置为位于多个物理设备上时，这也会阻止使用本地恢复。

### 分配保留调度

任务本地恢复假定在失败时保留分配任务调度，其工作如下。每个任务都会记住其先前的分配，并_请求完全相同的插槽_在恢复时重新启动。如果此插槽不可用，则任务将从资源管理器请求_新的新插槽_。这样，如果任务管理器不再可用，则无法返回其先前位置的_任务将不会驱动其先前插槽中的其他恢复任务_。我们的理由是，当任务管理器不再可用时，前一个插槽只能消失，在这种情况下是_一些_任务必须请求新的插槽。通过我们的调度策略，我们可以为最大数量的任务提供从本地状态恢复的机会，并避免任务将其先前的插槽彼此窃取的级联效应。

