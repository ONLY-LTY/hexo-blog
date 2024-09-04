---
title: Flink执行流程二
date: 2024-09-03 19:38:58
tags:
  - Flink
---
## 1.理解flink的图结构
第一部分讲到，我们的主函数最后一项任务就是生成StreamGraph，然后生成JobGraph，然后以此开始调度任务运行，所以接下来我们从这里入手，继续探索flink。
## 1.1 flink的三层图结构
事实上，flink总共提供了三种图的抽象，我们前面已经提到了StreamGraph和JobGraph，还有一种是ExecutionGraph，是用于调度的基本数据结构。

![image_1caf1oll019fp1odv1bh9idosr79.png-486.3kB][4]

上面这张图清晰的给出了flink各个图的工作原理和转换过程。其中最后一个物理执行图并非flink的数据结构，而是程序开始执行后，各个task分布在不同的节点上，所形成的物理上的关系表示。
 - 从JobGraph的图里可以看到，数据从上一个operator流到下一个operator的过程中，上游作为生产者提供了IntermediateDataSet，而下游作为消费者需要JobEdge。事实上，JobEdge是一个通信管道，连接了上游生产的dataset和下游的JobVertex节点。
 - 在JobGraph转换到ExecutionGraph的过程中，主要发生了以下转变：
  -  加入了并行度的概念，成为真正可调度的图结构
  -  生成了与JobVertex对应的ExecutionJobVertex，ExecutionVertex，与IntermediateDataSet对应的IntermediateResult和IntermediateResultPartition等，并行将通过这些类实现
 - ExecutionGraph已经可以用于调度任务。我们可以看到，flink根据该图生成了一一对应的Task，每个task对应一个ExecutionGraph的一个Execution。Task用InputGate、InputChannel和ResultPartition对应了上面图中的IntermediateResult和ExecutionEdge。

那么，flink抽象出这三层图结构，四层执行逻辑的意义是什么呢？
StreamGraph是对用户逻辑的映射。JobGraph在此基础上进行了一些优化，比如把一部分操作串成chain以提高效率。ExecutionGraph是为了调度存在的，加入了并行处理的概念。而在此基础上真正执行的是Task及其相关结构。
## 2.StreamGraph的生成
在第一节的算子注册部分，我们可以看到，flink把每一个算子transform成一个对流的转换（比如上文中返回的SingleOutputStreamOperator是一个DataStream的子类），并且注册到执行环境中，用于生成StreamGraph。实际生成StreamGraph的入口是

```java
//其中的transformations是一个list，里面记录的就是我们在transform方法中放进来的算子。
StreamGraphGenerator.generate(env, transformations)
```
StreamTransformation代表了从一个或多个DataStream生成新DataStream的操作。顺便，DataStream类在内部组合了一个StreamTransformation类，实际的转换操作均通过该类完成。
![image_1caf64b7c1gjnv2eebi1v9e1cvum.png-129.4kB][5]
我们可以看到，从source到各种map,union再到sink操作全部被映射成了StreamTransformation。
其映射过程如下所示：
![image_1caf6ak4rkqsc1u1hci93fe0d13.png-36.6kB][6]

以MapFunction为例：

 - 首先，用户代码里定义的UDF会被当作其基类对待，然后交给StreamMap这个operator做进一步包装。事实上，每一个Transformation都对应了一个StreamOperator。
 - 由于map这个操作只接受一个输入，所以再被进一步包装为OneInputTransformation。
 - 最后，将该transformation注册到执行环境中，当执行上文提到的generate方法时，生成StreamGraph图结构。

另外，并不是每一个StreamTransformation都会转换成runtime层中的物理操作。有一些只是逻辑概念，比如union、split/select、partition等。如下图所示的转换树，在运行时会优化成下方的操作图

![image_1caf71h79s0s3fodem1aeb1j3m1g.png-83.8kB][7]

## 3.StreamGraph生成函数分析
我们从StreamGraphGenerator.generate()方法往下看：

```java
	public static StreamGraph generate(StreamExecutionEnvironment env, List<StreamTransformation<?>> transformations) {
		return new StreamGraphGenerator(env).generateInternal(transformations);
	}

    //注意，StreamGraph的生成是从sink开始的
	private StreamGraph generateInternal(List<StreamTransformation<?>> transformations) {
		for (StreamTransformation<?> transformation: transformations) {
			transform(transformation);
		}
		return streamGraph;
	}

	//这个方法的核心逻辑就是判断传入的steamOperator是哪种类型，并执行相应的操作，详情见下面那一大堆if-else
	private Collection<Integer> transform(StreamTransformation<?> transform) {

		if (alreadyTransformed.containsKey(transform)) {
			return alreadyTransformed.get(transform);
		}

		LOG.debug("Transforming " + transform);

		if (transform.getMaxParallelism() <= 0) {

			// if the max parallelism hasn't been set, then first use the job wide max parallelism
			// from theExecutionConfig.
			int globalMaxParallelismFromConfig = env.getConfig().getMaxParallelism();
			if (globalMaxParallelismFromConfig > 0) {
				transform.setMaxParallelism(globalMaxParallelismFromConfig);
			}
		}

		// call at least once to trigger exceptions about MissingTypeInfo
		transform.getOutputType();

		Collection<Integer> transformedIds;
		//这里对操作符的类型进行判断，并以此调用相应的处理逻辑.简而言之，处理的核心无非是递归的将该节点和节点的上游节点加入图
		if (transform instanceof OneInputTransformation<?, ?>) {
			transformedIds = transformOneInputTransform((OneInputTransformation<?, ?>) transform);
		} else if (transform instanceof TwoInputTransformation<?, ?, ?>) {
			transformedIds = transformTwoInputTransform((TwoInputTransformation<?, ?, ?>) transform);
		} else if (transform instanceof SourceTransformation<?>) {
			transformedIds = transformSource((SourceTransformation<?>) transform);
		} else if (transform instanceof SinkTransformation<?>) {
			transformedIds = transformSink((SinkTransformation<?>) transform);
		} else if (transform instanceof UnionTransformation<?>) {
			transformedIds = transformUnion((UnionTransformation<?>) transform);
		} else if (transform instanceof SplitTransformation<?>) {
			transformedIds = transformSplit((SplitTransformation<?>) transform);
		} else if (transform instanceof SelectTransformation<?>) {
			transformedIds = transformSelect((SelectTransformation<?>) transform);
		} else if (transform instanceof FeedbackTransformation<?>) {
			transformedIds = transformFeedback((FeedbackTransformation<?>) transform);
		} else if (transform instanceof CoFeedbackTransformation<?>) {
			transformedIds = transformCoFeedback((CoFeedbackTransformation<?>) transform);
		} else if (transform instanceof PartitionTransformation<?>) {
			transformedIds = transformPartition((PartitionTransformation<?>) transform);
		} else if (transform instanceof SideOutputTransformation<?>) {
			transformedIds = transformSideOutput((SideOutputTransformation<?>) transform);
		} else {
			throw new IllegalStateException("Unknown transformation: " + transform);
		}

        //注意这里和函数开始时的方法相对应，在有向图中要注意避免循环的产生
		// need this check because the iterate transformation adds itself before
		// transforming the feedback edges
		if (!alreadyTransformed.containsKey(transform)) {
			alreadyTransformed.put(transform, transformedIds);
		}

		if (transform.getBufferTimeout() > 0) {
			streamGraph.setBufferTimeout(transform.getId(), transform.getBufferTimeout());
		}
		if (transform.getUid() != null) {
			streamGraph.setTransformationUID(transform.getId(), transform.getUid());
		}
		if (transform.getUserProvidedNodeHash() != null) {
			streamGraph.setTransformationUserHash(transform.getId(), transform.getUserProvidedNodeHash());
		}

		if (transform.getMinResources() != null && transform.getPreferredResources() != null) {
			streamGraph.setResources(transform.getId(), transform.getMinResources(), transform.getPreferredResources());
		}

		return transformedIds;
	}
```
因为map，filter等常用操作都是OneInputStreamOperator,我们就来看看*transformOneInputTransform*方法。
```java
private <IN, OUT> Collection<Integer> transformOneInputTransform(OneInputTransformation<IN, OUT> transform) {

	Collection<Integer> inputIds = transform(transform.getInput());

	// 在递归处理节点过程中，某个节点可能已经被其他子节点先处理过了，需要跳过
	if (alreadyTransformed.containsKey(transform)) {
		return alreadyTransformed.get(transform);
	}

  //这里是获取slotSharingGroup。这个group用来定义当前我们在处理的这个操作符可以跟什么操作符chain到一个slot里进行操作
  //因为有时候我们可能不满意flink替我们做的chain聚合
  //一个slot就是一个执行task的基本容器
	String slotSharingGroup = determineSlotSharingGroup(transform.getSlotSharingGroup(), inputIds);

  //把该operator加入图
	streamGraph.addOperator(transform.getId(),
			slotSharingGroup,
			transform.getOperator(),
			transform.getInputType(),
			transform.getOutputType(),
			transform.getName());

  //对于keyedStream，我们还要记录它的keySelector方法
  //flink并不真正为每个keyedStream保存一个key，而是每次需要用到key的时候都使用keySelector方法进行计算
  //因此，我们自定义的keySelector方法需要保证幂等性
  //到后面介绍keyGroup的时候我们还会再次提到这一点
	if (transform.getStateKeySelector() != null) {
		TypeSerializer<?> keySerializer = transform.getStateKeyType().createSerializer(env.getConfig());
		streamGraph.setOneInputStateKey(transform.getId(), transform.getStateKeySelector(), keySerializer);
	}

	streamGraph.setParallelism(transform.getId(), transform.getParallelism());
	streamGraph.setMaxParallelism(transform.getId(), transform.getMaxParallelism());

  //为当前节点和它的依赖节点建立边
  //这里可以看到之前提到的select union partition等逻辑节点被合并入edge的过程
	for (Integer inputId: inputIds) {
		streamGraph.addEdge(inputId, transform.getId(), 0);
	}

	return Collections.singleton(transform.getId());
}

public void addEdge(Integer upStreamVertexID, Integer downStreamVertexID, int typeNumber) {
	addEdgeInternal(upStreamVertexID,
			downStreamVertexID,
			typeNumber,
			null,
			new ArrayList<String>(),
			null);

}
  //addEdge的实现，会合并一些逻辑节点
private void addEdgeInternal(Integer upStreamVertexID,
		Integer downStreamVertexID,
		int typeNumber,
		StreamPartitioner<?> partitioner,
		List<String> outputNames,
		OutputTag outputTag) {
  //如果输入边是侧输出节点，则把side的输入边作为本节点的输入边，并递归调用
	if (virtualSideOutputNodes.containsKey(upStreamVertexID)) {
		int virtualId = upStreamVertexID;
		upStreamVertexID = virtualSideOutputNodes.get(virtualId).f0;
		if (outputTag == null) {
			outputTag = virtualSideOutputNodes.get(virtualId).f1;
		}
		addEdgeInternal(upStreamVertexID, downStreamVertexID, typeNumber, partitioner, null, outputTag);
		//如果输入边是select，则把select的输入边作为本节点的输入边
	} else if (virtualSelectNodes.containsKey(upStreamVertexID)) {
		int virtualId = upStreamVertexID;
		upStreamVertexID = virtualSelectNodes.get(virtualId).f0;
		if (outputNames.isEmpty()) {
			// selections that happen downstream override earlier selections
			outputNames = virtualSelectNodes.get(virtualId).f1;
		}
		addEdgeInternal(upStreamVertexID, downStreamVertexID, typeNumber, partitioner, outputNames, outputTag);
		//如果是partition节点
	} else if (virtualPartitionNodes.containsKey(upStreamVertexID)) {
		int virtualId = upStreamVertexID;
		upStreamVertexID = virtualPartitionNodes.get(virtualId).f0;
		if (partitioner == null) {
			partitioner = virtualPartitionNodes.get(virtualId).f1;
		}
		addEdgeInternal(upStreamVertexID, downStreamVertexID, typeNumber, partitioner, outputNames, outputTag);
	} else {
	//正常的edge处理逻辑
		StreamNode upstreamNode = getStreamNode(upStreamVertexID);
		StreamNode downstreamNode = getStreamNode(downStreamVertexID);

		// If no partitioner was specified and the parallelism of upstream and downstream
		// operator matches use forward partitioning, use rebalance otherwise.
		if (partitioner == null && upstreamNode.getParallelism() == downstreamNode.getParallelism()) {
			partitioner = new ForwardPartitioner<Object>();
		} else if (partitioner == null) {
			partitioner = new RebalancePartitioner<Object>();
		}

		if (partitioner instanceof ForwardPartitioner) {
			if (upstreamNode.getParallelism() != downstreamNode.getParallelism()) {
				throw new UnsupportedOperationException("Forward partitioning does not allow " +
						"change of parallelism. Upstream operation: " + upstreamNode + " parallelism: " + upstreamNode.getParallelism() +
						", downstream operation: " + downstreamNode + " parallelism: " + downstreamNode.getParallelism() +
						" You must use another partitioning strategy, such as broadcast, rebalance, shuffle or global.");
			}
		}

		StreamEdge edge = new StreamEdge(upstreamNode, downstreamNode, typeNumber, outputNames, partitioner, outputTag);

		getStreamNode(edge.getSourceId()).addOutEdge(edge);
		getStreamNode(edge.getTargetId()).addInEdge(edge);
	}
}
```
### 3.1 WordCount函数的StreamGraph
flink提供了一个StreamGraph可视化显示工具，[在这里](https://flink.apache.org/visualizer/)
我们可以把我们的程序的执行计划打印出来 *System.out.println(env.getExecutionPlan());* 复制到这个网站上，点击生成，如图所示：

![image_1cafgsliu1n2n1uj21p971b0h6m71t.png-25.7kB][8]
可以看到，我们源程序被转化成了4个operator。另外，在operator之间的连线上也显示出了flink添加的一些逻辑流程。由于我设定了每个操作符的并行度都是1，所以在每个操作符之间都是直接FORWARD，不存在shuffle的过程。
## 4.JobGraph的生成
flink会根据上一步生成的StreamGraph生成JobGraph，然后将JobGraph发送到server端进行ExecutionGraph的解析。
### 4.1 JobGraph生成源码
与StreamGraph类似，JobGraph的入口方法是

```java
StreamingJobGraphGenerator.createJobGraph()
```

```java
private JobGraph createJobGraph() {

	// 设置启动模式为所有节点均在一开始就启动
	jobGraph.setScheduleMode(ScheduleMode.EAGER);

	// 为每个节点生成hash id
	Map<Integer, byte[]> hashes = defaultStreamGraphHasher.traverseStreamGraphAndGenerateHashes(streamGraph);

	// 为了保持兼容性创建的hash
	List<Map<Integer, byte[]>> legacyHashes = new ArrayList<>(legacyStreamGraphHashers.size());
	for (StreamGraphHasher hasher : legacyStreamGraphHashers) {
		legacyHashes.add(hasher.traverseStreamGraphAndGenerateHashes(streamGraph));
	}

	Map<Integer, List<Tuple2<byte[], byte[]>>> chainedOperatorHashes = new HashMap<>();
      //生成jobvertex，串成chain等
      //这里的逻辑大致可以理解为，挨个遍历节点，如果该节点是一个chain的头节点，就生成一个JobVertex，如果不是头节点，就要把自身配置并入头节点，然后把头节点和自己的出边相连；对于不能chain的节点，当作只有头节点处理即可
	setChaining(hashes, legacyHashes, chainedOperatorHashes);
      //设置输入边edge
	setPhysicalEdges();
      //设置slot共享group
	setSlotSharing();
      //配置检查点
	configureCheckpointing();

	// 如果有之前的缓存文件的配置的话，重新读入
	for (Tuple2<String, DistributedCache.DistributedCacheEntry> e : streamGraph.getEnvironment().getCachedFiles()) {
		DistributedCache.writeFileInfoToConfig(e.f0, e.f1, jobGraph.getJobConfiguration());
	}

	// 传递执行环境配置
	try {
		jobGraph.setExecutionConfig(streamGraph.getExecutionConfig());
	}
	catch (IOException e) {
		throw new IllegalConfigurationException("Could not serialize the ExecutionConfig." +
				"This indicates that non-serializable types (like custom serializers) were registered");
	}

	return jobGraph;
}
```
### 4.2 operator chain的逻辑
 为了更高效地分布式执行，Flink会尽可能地将operator的subtask链接（chain）在一起形成task。每个task在一个线程中执行。将operators链接成task是非常有效的优化：它能减少线程之间的切换，减少消息的序列化/反序列化，减少数据在缓冲区的交换，减少了延迟的同时提高整体的吞吐量。

![image_1cafj7s6bittk5tt0bequlig2a.png-158.7kB][9]
上图中将KeyAggregation和Sink两个operator进行了合并，因为这两个合并后并不会改变整体的拓扑结构。但是，并不是任意两个 operator 就能 chain 一起的,其条件还是很苛刻的：
 - 上下游的并行度一致
 - 下游节点的入度为1 （也就是说下游节点没有来自其他节点的输入）
 - 上下游节点都在同一个 slot group 中（下面会解释 slot group）
 - 下游节点的 chain 策略为 ALWAYS（可以与上下游链接，map、flatmap、filter等默认是ALWAYS）
 - 上游节点的 chain 策略为 ALWAYS 或 HEAD（只能与下游链接，不能与上游链接，Source默认是HEAD）
 - 两个节点间数据分区方式是 forward（参考理解数据流的分区）
 - 用户没有禁用 chain

flink的chain逻辑是一种很常见的设计，比如spring的interceptor也是类似的实现方式。通过把操作符串成一个大操作符，flink避免了把数据序列化后通过网络发送给其他节点的开销，能够大大增强效率。
### 4.3 JobGraph的提交
前面已经提到，JobGraph的提交依赖于JobClient和JobManager之间的异步通信，如图所示：

![image_1cafn516r1p68kt31g7r196rcsv2n.png-40.1kB][10]
在submitJobAndWait方法中，其首先会创建一个JobClientActor的ActorRef,然后向其发起一个SubmitJobAndWait消息，该消息将JobGraph的实例提交给JobClientActor。发起模式是ask，它表示需要一个应答消息。

```java
Future<Object> future = Patterns.ask(jobClientActor, new JobClientMessages.SubmitJobAndWait(jobGraph), new Timeout(AkkaUtils.INF_TIMEOUT()));
answer = Await.result(future, AkkaUtils.INF_TIMEOUT());
```
该SubmitJobAndWait消息被JobClientActor接收后，最终通过调用tryToSubmitJob方法触发真正的提交动作。当JobManager的actor接收到来自client端的请求后，会执行一个submitJob方法，主要做以下事情：
 - 向BlobLibraryCacheManager注册该Job；
 - 构建ExecutionGraph对象；
 - 对JobGraph中的每个顶点进行初始化；
 - 将DAG拓扑中从source开始排序，排序后的顶点集合附加到Exec> - utionGraph对象；
 - 获取检查点相关的配置，并将其设置到ExecutionGraph对象；
 - 向ExecutionGraph注册相关的listener；
 - 执行恢复操作或者将JobGraph信息写入SubmittedJobGraphStore以在后续用于恢复目的；
 - 响应给客户端JobSubmitSuccess消息；
 - 对ExecutionGraph对象进行调度执行；

最后，JobManger会返回消息给JobClient，通知该任务是否提交成功。
## 5.ExecutionGraph的生成
与StreamGraph和JobGraph不同，ExecutionGraph并不是在我们的客户端程序生成，而是在服务端（JobManager处）生成的，顺便flink只维护一个JobManager。其入口代码是：

```java
ExecutionGraphBuilder.buildGraph（...）
```
该方法长200多行，其中一大半是checkpoiont的相关逻辑，我们暂且略过，直接看核心方法 *executionGraph.attachJobGraph* 因为ExecutionGraph事实上只是改动了JobGraph的每个节点，而没有对整个拓扑结构进行变动，所以代码里只是挨个遍历jobVertex并进行处理：

```java
for (JobVertex jobVertex : topologiallySorted) {
	if (jobVertex.isInputVertex() && !jobVertex.isStoppable()) {
		this.isStoppable = false;
	}

	//在这里生成ExecutionGraph的每个节点
	//首先是进行了一堆赋值，将任务信息交给要生成的图节点，以及设定并行度等等
	//然后是创建本节点的IntermediateResult，根据本节点的下游节点的个数确定创建几份
	//最后是根据设定好的并行度创建用于执行task的ExecutionVertex
	//如果job有设定inputsplit的话，这里还要指定inputsplits
	ExecutionJobVertex ejv = new ExecutionJobVertex(
		this,
		jobVertex,
		1,
		rpcCallTimeout,
		globalModVersion,
		createTimestamp);

    //这里要处理所有的JobEdge
    //对每个edge，获取对应的intermediateResult，并记录到本节点的输入上
    //最后，把每个ExecutorVertex和对应的IntermediateResult关联起来
	ejv.connectToPredecessors(this.intermediateResults);

	ExecutionJobVertex previousTask = this.tasks.putIfAbsent(jobVertex.getID(), ejv);
	if (previousTask != null) {
		throw new JobException(String.format("Encountered two job vertices with ID %s : previous=[%s] / new=[%s]",
				jobVertex.getID(), ejv, previousTask));
	}

	for (IntermediateResult res : ejv.getProducedDataSets()) {
		IntermediateResult previousDataSet = this.intermediateResults.putIfAbsent(res.getId(), res);
		if (previousDataSet != null) {
			throw new JobException(String.format("Encountered two intermediate data set with ID %s : previous=[%s] / new=[%s]",
					res.getId(), res, previousDataSet));
		}
	}

	this.verticesInCreationOrder.add(ejv);
	this.numVerticesTotal += ejv.getParallelism();
	newExecJobVertices.add(ejv);
}
```
至此，ExecutorGraph就创建完成了。

[4]: https://static.zybuluo.com/bethunebtj/nseitc0kyuq0n44s7qcp6ij9/image_1caf1oll019fp1odv1bh9idosr79.png
[5]: https://static.zybuluo.com/bethunebtj/69v9syr2p5k5om3c4jox9wh0/image_1caf64b7c1gjnv2eebi1v9e1cvum.png
[6]: https://static.zybuluo.com/bethunebtj/a8sjspg8agzl3utnncntds9q/image_1caf6ak4rkqsc1u1hci93fe0d13.png
[7]: https://static.zybuluo.com/bethunebtj/6zmlsivd9cjdm5nhsacuk3o1/image_1caf71h79s0s3fodem1aeb1j3m1g.png
[8]: https://static.zybuluo.com/bethunebtj/sfckex3xgu33m3srk2bc5hgk/image_1cafgsliu1n2n1uj21p971b0h6m71t.png
[9]: https://static.zybuluo.com/bethunebtj/jcjalvv130ex52vkglkt56r2/image_1cafj7s6bittk5tt0bequlig2a.png
[10]: https://static.zybuluo.com/bethunebtj/dj015uuqpnb4ct7810qfilhe/image_1cafn516r1p68kt31g7r196rcsv2n.png
