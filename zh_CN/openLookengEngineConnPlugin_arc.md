## 如何实现openlookeng的Linkis引擎插件
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;本文介绍如何实现OpenLooKeng的引擎插件，本文还是采用on Yarn的模式进行介绍，如果是Standalone模式可以参考Presto引擎插件的实现。

## 1. 为什么采用On yarn模式

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;因为On Yarn模式可以做到实时调度启动一个Presto集群，可以很好做到对资源进行伸缩，对于存在Yarn集群的用户更加友好。
实现思路主要如下：
1. 通过Linkis调度启动一个openLookeng引擎插件进程
2. 并在openLookeng引擎插件的默认EngineConn启动的时候调用openLookeng on yarn的pyton脚本
3. 在初始化后获取openLookeng集群的url，并汇报给linkismanager启动成功
4. 任务提交到openLookeng 引擎后会首先对代码进行解析封装后在executor中通过调用openlookeng的java sdk进行任务提交和结果获取（类似于presto引擎插件的实现）
5. openLookeng引擎插件在空闲一段时间后，可以自动进行退出，或者由用户通过linkis管理台进行控制

## 2. 具体实现逻辑
1. 新建openLookeng插件模块：
因为openLookeng引擎需要作为交互式引擎的方式存在，所以这里直接引入linkis-computation-engineconn模块即可
```
<dependency>
<groupId>com.webank.wedatasphere.linkis</groupId>
<artifactId>linkis-computation-engineconn</artifactId>
<version>${linkis.version}</version>
</dependency>
```
2. 实现ECP的主要接口：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;a)OpenLookengEngineConnPlugin，这个是引擎插件上下文类，用户对插件相关的接口进行管理。启动EngineConn时，会先找到对应的EngineConnPlugin类，以此为入口，获取其它核心接口的实现，是必须实现的主要接口。
    
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;b)OpenLookengEngineConnFactory，实现如何启动一个引擎连接器，和如何启动一个引擎执行器的逻辑，是必须实现的接口，这里用于调用Python 脚本并获取启动后的URL。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;b.a 实现createEngineConn方法：返回一个EngineConn对象，其中，getEngine返回一个封装了与底层引擎连接信息的对象，这里主要返回包含了OpenLookeng集群URL的对象。
    
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;b.b 继承ComputationSingleExecutorEngineConnFactory，实现createExecutor，返回对应的Executor，Executor是真正的执行器。
        
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;c)OpenLookengEngineConnResourceFactory，用于限定启动一个引擎所需要的资源，引擎启动前，将以此为依 据 向 Linkis Manager 申 请 资 源。非必须，默认可以使用GenericEngineResourceFactory只做本机资源管控，建议实现类似spark引擎插件的方式完成对Yarn资源的管控。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;d)EngineLaunchBuilder，用于封装EngineConnManager可以解析成启动命令的必要信息。非必须，这里可以直接继承JavaProcessEngineConnLaunchBuilder，因为默认启动一个简单的java进程。

3. 实现Executor。Executor为执行器，作为真正的计算场景执行器，是实际的计算逻辑执行单元，也对引擎各种具体能力的抽象，提供加锁、访问状态、获取日志等多种不同的服务。根据实际的使用需要，Linkis默认提供以下的派生Executor基类，其类名和主要作用如下：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;a) SensibleExecutor：
       
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; i. Executor存在多种状态，允许Executor切换状态
         
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ii. Executor切换状态后，允许做通知等操作
         
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;b) YarnExecutor：指Yarn类型的引擎，能够获取得到applicationId和applicationURL和队列。
       
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;c) ResourceExecutor: 指引擎具备资源动态变化的能力，配合提供requestExpectedResource方法，用于每次希望更改资源时，先向RM申请新的资源；而resourceUpdate方法，用于每次引擎实际使用资源发生变化时，向RM汇报资源情况。
       
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;d) AccessibleExecutor：是一个非常重要的Executor基类。如果用户的Executor继承了该基类，则表示该Engine是可以被访问的。这里需区分SensibleExecutor的state()和 AccessibleExecutor 的 getEngineStatus()方法：state()用于获取引擎状态，getEngineStatus()会获取引擎的状态、负载、并发等基础指标Metric数据。
       
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;e) 同时，如果继承了 AccessibleExecutor，会触发Engine 进程实例化多个EngineReceiver方法。EngineReceiver用于处理Entrance、EM和LinkisMaster的RPC请求，使得该引擎变成了一个可被访问的引擎，用户如果有特殊的RPC需求，可以通过实现RPCService接口，进而实现与AccessibleExecutor通信。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;f) ExecutableExecutor：是一个常驻型的Executor基类，常驻型的Executor包含：生产中心的Streaming应用、提交给Schedulis后指定要以独立模式运行的脚步、业务用户的业务应用等。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;g) StreamingExecutor：Streaming为流式应用，继承自ExecutableExecutor，需具备诊断、do checkpoint、采集作业信息、监控告警的能力。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;h) ComputationExecutor：是常用的交互式引擎Executor，处理交互式执行任务，并且具备状态查询、任务kill等交互式能力。
OpenLookeng默认实现ComputationExecutor接口即可，这里需要讨论的时候是否支持openLookeng引擎插件的并发，如果需要支持多用户并发的话实现ConcurrentComputationExecutor接口 
不管哪种方式，主要实现executorLine方法既可以，用于执行一行sql代码，通过调用OpenLookeng的sdk并通过集群url进行任务提交，并获取执行进度和结果进行存储。

## 待确认点：
1. 配置文件是否放到引擎插件的conf目录还是在机器上面安装对应的配置文件
2. 如何支持多hadoop集群
