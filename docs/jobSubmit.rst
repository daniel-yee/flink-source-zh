任务提交
===============
Flink任务在被提交到Yarn上后会经过如下流程,具体如下:

 .. image:: image/flink-submit-to-yarn.png


 #. Client从客户端代码生成的StreamGraph提取出JobGraph;
 #. 上传JobGraph和对应的jar包;
 #. 启动App Master;
 #. 启动JobManager;
 #. 启动ResourceManager;
 #. JobManager向ResourceManager申请slots;
 #. ResourceManager向Yarn ResourceManager申请Container;
 #. 启动申请到的Container;
 #. 启动的Container作为TaskManager向ResourceManager注册;
 #. ResourceManger向TaskManager请求slot;
 #. TaskManager提供slot给JobManager,让其分配任务执行.

 上面的流程主要包含Client,JobManager,ResourceManager,TaskManager共四个部分.接下来就对每个部分进行详细的分析.

生成StreamGraph
----------------
在用户编写一个Flink任务之后是怎么样一步步转换成Flink的第一层抽象StreamGraph的呢?本节将会对此进行详细的介绍.

StreamGraph生成的主要流程如下:

 * 用户对DataStream声明的每个操作都会将该操作对应的Transformation添加到Transformations列表:List
 * 用户程序中调用env.execute后(batch调用print方法类似),Flink将从List的Sink开始自底向上进行遍历,这也是为何Flink一定要写Sink的原因,没有Sink就无法生成StreamGraph.
 * 如果上游Transformation还没有进行处理,会先对上游的Transformation进行处理,处理即包装成一个StreamNode,再通过Edge建立上下游StreamNode的联系.
 * StreamGraphGenerator.generate()方法会最终生成一个完整的StreamGraph

 其中,addSink的大致流程为:生成Operator -> 生成Transformation -> 加入Transformations中.具体操作如下:

 #. 对用户函数进行序列化,并转化成Operator
 #. clean进行闭包操作,如使用了哪些外部变量,会对所有字段进行遍历,并将它们的引用存储在闭包中
 #. 完成Operator到SinkTransformation的转换,由DataStream和Operator共同构建一个SinkTransformation
 #. 将SinkTransformation加入到transformations中

其实Transformation包含许多种类型,除了上面的SinkTransformation,还有SourceTransformation,OneInputTransformation,TwoInputTransformaion,PartitionTransformaion,
SelectTransformation等等.具体的使用场景如下:

 * PartitionTransformation:如果用户想要对DataStream进行keyby操作,得到一个KeyedStream,即需要对数据重新分区.首先,用户需要设置根据什么key进行
   分区,即KeySelector.然后在生成KeyedStream的过程中,会得到一个PartitionTransformation.在PartitionTransformation中会对这条记录通过key进行计算,
   判断应该发往下游哪个节点,KeyGroup可以由maxParallism进行调整.
 * TwoInputTransformaion:指包含两个输入流,如inputStream1和inputStream2,加上这个Transformation的输出,及Operator即可得到一个完整的TwoInputTransformation.

以上过程得到了transformations的List,接下来就可以通过StreamGraphGenerator生成完整的StreamGraph.
生成StreamGraph时会遍历Transformation树,逐个对Transformation进行转化,具体的转化由transform()方法完成.transform最终都会调用transformXXX对
具体的StreamTransformation进行转换.transformPartition则是创建VirtualNode而不是StreamNode.


Client
-------------
Client模块的入口为CliFrontend,用于接收和处理各种命令与请求,如Run和Cancel代表运行和取消任务,CliFrontend在收到对应命令后,根据参数来具体执行命令.
Run命令中必须执行Jar和Class,也可指定SavePoint目录来恢复任务.

Client会根据Jar来提取出Plan,即DataFlow.然后在此Plan的基础上生成JobGraph.其主要操作是对StreamGraph进行优化,将能chain在一起的Operator进行Chain在一起的操作.
在得到JobGraph后就会提交JobGraph等内容,为任务的运行做准备.
Operator能chain在一起的条件:
 #. 上下游Operator的并行度一致
 #. 下游节点的入度为1
 #. 上下游节点都在同一个SlotSharingGroup中(默认为default)
 #. 下游节点的chain策略是ALWAYS(可以与上下游链接,map/flatmap/filter等默认是ALWAYS)
 #. 上游节点的chain策略是ALWAYS或HEAD(只能与下游链接,不能与上游链接,source默认是HEAD)
 #. 两个Operator间的数据分区方式是fowward
 #. 用户没有禁用chain