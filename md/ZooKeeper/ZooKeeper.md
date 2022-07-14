# 会话

## 会话状态

会话在整个生命周期中，会在不同的状态之间进行切换，这些状态一般可以分为`CONNECTING`、`CONNECTED`、`RECONNECTING`、`RECONNECTED`和`CLOSE`等。

一且客户端开始创建 ZooKeeper 对象，那么客户端状态就会变成`CONNECTING`；同时，客户端开始从服务器地址列表中逐个选取 IP 地址来尝试进行网络连接，直到成功连接上服务器，然后将客户端状态变更为`CONNECTED`。

通常情况下，在 ZooKeeper 运行期间，客户端的状态会因为断开重连，而经常在`CONNECTING`和`CONNECTED`两个状态间进行切换。

如果出现诸如会话超时、权限检查失败或是客户端主动退出程序等情况，那么客户端的状态就会直接变更为`CLOSE`。

## 会话创建

### Session

Session 是 ZooKeeper 中的会话实体，代表了一个客户端会话，其包含以下4个基本属性：

#### sessionId（会话lD）

用来唯一标识一个会话，当每次客户端向服务端发起「会话创建」请求时，服务端都会为其分配一个全局唯一的`sessionId`。

##### #实现初始化sessionId

在`SessionTracker`接口的实现类`SessionTrackerImpl`初始化的时候，会调用`initializeNextSession`方法来生成一个初始化的`sessionId`。

1. 获取当前时间的毫秒级时间戳`System.currentTimeMillis()`，并进行左移24位`<< 24`，再无符号右移8位`>>> 8`的操作
2. 获取机器标识 SlD 左移56位的值`sid << 56`
3. 将上两步中得到的两个值进行「按位或」操作

可以将上述算法概括为：高8位确定了所在机器，后56位使用当前时间的毫秒级时间戳进行随机。

**使用无符号右移的原因**是防止左移后的数值为负数而导致的有符号右移补1，这样可以避免高位数值对SID的干扰。

#### timeout（会话超时时间）

客户端在构造 ZooKeeper 实例的时候，会配置一个`sessionTimeout`参数用干指定会话的超时时间。

ZooKeeper 客户端向服务器发送这个超时时间后，服务器会根据自己的超时时间限制最终确定会话的超时时间。

#### tickTime（下次会话超时时间点）

为了便于 ZooKeeper 对会话实行「分桶策略」管理，同时也是为了高效低耗地实现会话的超时检查与清理，ZooKeeper 会为每个会话标记一个下次会话超时时间点。默认情况下`tickTime`为2000，即每隔2000毫秒进行一次会话超时检查。

#### isClosing（会话关闭标记）

当服务端检测到一个会话已经超时失效的时候，会将该会话的`isClosing`属性标记为「己关闭」，这样就能确保不再处理来自该会话的新请求了。

### SessionTracker

`SessionTracker`是 ZooKeeper 服务端的会话管理器，负责会话的创建、管理和清理等工作。

每一个会话在实现类`SessionTrackerImpl`内部都保留了三份：

#### sessionsById

根据`sessionId`来管理 Session 实体的`HashMap<Long, Sessionimpl>`类型数据结构

#### sessionsWithTimeout

根据`sessionId`来管理会话超时时间`timeout`的`ConcurrentHashMap<Long, Integer>`类型数据结构

#### sessionSets

根据下次会话超时时间点`tickTime`来归档会话，便于进行会话管理和超时检查的`HashMap<Long, SessionSet>`类型数据结构


## 一次会话的创建过程

### 1. 客户端初始化阶段

#### 1. 初始化ZooKeeper对象

通过调用 ZooKeeper 的构造方法来实例化一个 ZooKeeper 对象，在初始化过程中，会创建一个客户端的 Watcher 管理器`ZKWatchManager`（`ClientWatchManager`接口的实现类）。

#### 2. 设置默认 Watcher

如果在 ZooKeeper 的构造方法中传入一个 Watcher 对象的话，那么客户端会将这个对象作为作为整个客户端会话期间的默认 Watcher保存在`ZKWatchManager`的`defaultWatcher`中。

#### 3. 构造 HostProvider。

对于构造方法中传入的服务器地址，客户端会将其存放在服务器地址列表管理器`HostProvider`中。

#### 4. 创建并初始化 ClientCnxn

客户端创建`ClientCnxn`，用来管理客户端与服务器的网络交互。

另外，在创建`ClientCnxn`的同时，客户端还会初始化客户端两个核心队列`outgoingQueue`和`pendingQueue`，分别作为客户端请求的发送队列和服务端响应的接收队列。

最后，客户端还会同时创建`ClientCnxnSocket`，这是`ClientCnxn`的底层IO处理器。

#### 5. 初始化 SendThread 和 EventThread

客户端创建两个核心线程`SendThread`和`EventThread`，再将`ClientCnxnSocket`分配给`SendThread`作为底层网络IO处理器，并初始化`EventThread`的`waitingEvents`队列，用于存放所有等待被客户端处理的事件。

### 2. 会话创建阶段

#### 6. 启动 SendThread 和 EventThread

`SendThread`首先会判断当前客户端的状态，进行一系列清理性工作，为客户端发送**会话创建**请求做准备。

#### 7. 获取一个服务器地址
在开始创建TCP连接之前，`SendThread`首先需要从`HostProvider`中随机获取出一个 ZooKeeper 服务器的目标地址，然后委托给`ClientCnxnSocket`。

#### 8. 创建 TCP 连接。
获取到一个服务器地址后，`ClientCnxnSocket`负责创建一个和服务器进行交互的TCP长连接。

#### 9. 构造 ConnectRequest 请求。
上一步只是纯粹地在网络层创建了客户端与服务端之间的 Socket 连接，但远未完成 ZooKeeper 客户端会话的创建。

`SendThread`会根据当前客户端的实际设置，构造出一个`ConnectRequest`请求，该请求代表了客户端试图与服务器创建一个会话。

同时，ZooKeeper 客户端还会进一步将该请求包装成网络IO层的`Packet`对象，放入请求发送队列`outgoingQueue`中去。

#### 10. 发送请求
当客户端请求准备完毕后，就可以开始向服务端发送请求了。

`ClientCnxnSocket`负责从`outgoingQueue`中取出一个待发送的`Packet`对象，将其序列化成`ByteBuffer`后，向服务端进行发送。

### 3. 响应处理阶段

#### 11. 接收服务端响应

`ClientCnxnSocket`接收到服务端的响应后，会首先判断当前的客户端状态是否是「已初始化状态」。

如果尚未完成初始化，那么就认为该响应一定是会话创建请求的响应，直接交由`readConnectResult`方法来处理该响应。

#### 12. 处理Response

`ClientCnxnSocket`会对接收到的服务端响应进行反序列化，得到`ConnectResponse`对象，并从中获取到 ZooKeeper 服务端分配的`sessionId`。

#### 13. 连接成功
连接成功后，客户端一方面需要通知`SendThread`线程，进一步对客户端进行会话参数的设置，包括`readTimeout`和`connectTimeout`等，并更新客户端状态； 另
一方面，需要通知地址管理器`HostProvider`当前成功连接的服务器地址。

#### 14. 生成事件 SyncConnected-None
为了能够让上层应用感知到会话的成功创建，`SendThread`会生成一个事件「SyncConnected-None」，代表客户端与服务器会话创建成功，并将该事件传递
给`EventThread`线程。

#### 15. 查询 Watcher
`EventThread`线程收到事件后，会针对「SyncConnected-None」事件，从`ZKWatchManager`管理器中查询出对应的Watcher，然后将其放到`EventThread`的`waitingEvents`队列中去。

#### 16. 处理事件
`EventThread`不断地从`waitingEvents`队列中取出待处理的Watcher对象，然后直接调用该对象的`process`方法，以达到触发Watcher的目的。

### 创建会话的过程

#### 0. 客户端发起「会话创建」请求

#### 1. 请求接收

#####1.  IO层接收来自客户端的请求

在 ZooKeeper 中，`NIOServerCnxnFactory`线程会接收来自客户端请求，并将`NIOServerCnxn`对象从底层网络IO`attachment`出来。

`NIOServerCnxn`实例维护了一个服务器与客户端之间的TCP长连接，ZooKeeper 通过调用它的`doIO`方法处理客户端与服务端的通信。

#####2.  判断是否是客户端「会话创建」请求。

对于当前请求，`NIOServerCnxn`会先检查这是一个「读IO事件」还是「IO写事件」，因为此处是一个「会话创建」请求，因此是「读IO事件」。

之后，`NIOServerCnxn`会调用`readPayload`方法并检查当前`NIOServerCnxn`实体是否已经完成初始化，即`initialized`是否为 true。

如果尚未被初始化，那么就可以确定该客户端请求是一个「会话创建」请求。

##### 3. 反序列化 ConnectRequest 请求

一且确定当前客户端请求是「会话创建」请求，那么`NIOServerCnxn`就会进入`readConnectRequest`方法，并调用`zkServer.processConnectRequest`让服务端对这个请求进行反序列化，并生成一个请求实体`ConnectRequest`。

##### 4. 判断是否是 ReadOnly 客户端

在 ZooKeeper 的设计实现中，如果当前 ZooKeeper 服务器是以 ReadOnly 模式启动的，那么所有来自非 ReadOnly 型客户端的请求将无法被处理。

因此，针对当前`ConnectRequest`，服务端会首先检查其是否是 ReadOnly 客户端，并以此来决定是否接受该「会话创建」请求。

##### 5. 检查客户端ZXID

在正常情况下，同一个ZooKeeper集群中，服务端的 ZXID 必定大于客户端的ZXID，如果不满足该条件，则服务端将不接受该客户端的「会话创建」请求。

##### 6. 协商 sessionTimeout

客户端在构造 ZooKeeper 实例的时候，会有一个`sessionTimeout`参数用于指定会话的超时时间。

客户端请求创建会话时，会携带这个超时时间（`ConnectRequest`中的`timeOut`属性），而服务器会根据自己的超时时间限制，与客户端 “协商” 决定该会话的超时时间。

默认情况下，ZooKeeper 服务端对超时时间的限制介于2个`tickTime`到20个`tickTime`之间，也可以通过`zoo.cfg`中的相关配置来调整超时时间限制。

##### 7. 判断是否需要重新创建会话

服务端根据客户端请求中是否包含`sessionId`来判断该客户端是否需要重新创建会话。

如果客户端请求中已经包含了`sessionId`，那么就认为该客户端正在进行会话重连。

在这种情况下，服务端只需要重新打开这个会话，否则需要重新创建。

#### 2. 会话创建

确定该客户端需要创建会话后，会进入`ZooKeeperServer`的`createSession`方法。

##### 8. 为客户端生成 sessionId

在为客户端创建会话之前，服务端首先会为每个客户端都分配一个`sessionId`，这步是在`SessionTrackerImpl`实体类的同步方法`createSession`中完成的。

###### 分配方式

每个 ZooKeeper 服务器在启动的时候，都会初始化一个会话管理器`SessionTracker`，同时初始化`sessionId`，这个 ID 被称为「基准 sessionId」。

在 ZooKeeper 中，是通过保证「基准 sessionId」的全局唯一性，并在这个基础上进行原子性递增，从而实现了服务器集群为会话分配分布式ID的工作。

因此「基准 sessionId」的初始化算法非常重要，详细可见 #实现初始化sessionId

##### 9. 注册会话

创建会话最重要的工作就是向`SessionTracker`中注册会话。

在「会话创建」初期，`SessionTrackerImpl`实体类通过调用`addSession`方法将该客户端会话的相关信息保存到`sessionsWithTimeout`和`sessionsById`中，方便后续会话管理器进行管理。

##### 10. 激活会话

向`SessionTracker`注册完会话后，接下来还需要对会话进行激活操作。

激活会话的核心作用是基于「分桶策略」为会话安排一个区块，并**确保在调用请求处理器前，该会话处于激活状态**，也便于会话清理线程`SessionTrackerImpl`能够快速高效地进行会话清理，这步通过`touchSession`方法完成。

##### 11. 生成会话密码

服务端在创建一个客户端会话的时候，会同时为客户端生成一个会话密码（类型为 byte 数组），连同`sessionId`一起发送给客户端，作为会话在集群中不同机器间转移的凭证。

会话密码的生成算法：`static final private long superSecret = 0XB3415C00L; Random r = new Random(sessionId ^ superSecret); r.nextBytes(passwd);`

#### 3. 预处理

ZooKeeper 对于每个客户端请求的处理模型采用了典型的责任链模式，每个客户端请求都会由几个不同的请求处理器依次进行处理。

[Zookeeper系列（二十四）Zookeeper原理解析之处理流程](https://blog.csdn.net/luckykapok918/article/details/71725957)

##### 12. 将请求交给 ZooKeeper 的 PrepRequestProcessor 处理器进行处理

服务端通过在`submitRequest`方法中调用`firstProcessor.processRequest(request);`将请求提交给第一个请求处理器`PrepRequestProcessor`。

随后`PrepRequestProcessor`线程会在执行体中调用`submittedRequests.take()`取出服务端提交的请求，之后调用`pRequest`方法对请求进行预处理。

##### 13. 创建请求事务头

对于事务请求，ZooKeeper 首先会为其创建请求事务头：`request.hdr = new TxnHeader(request.sessionId, request.cxid, zxid, zks.getTime(), type);`

请求事务头是每一个 ZooKeeper 事务请求中非常重要的一部分，服务端后续的请求处理器都是基于请求头来识别当前请求是否是事务请求。

[ZooKeeper请求事务头属性说明]()

##### 14. 创建请求事务体

对于事务请求，ZooKeeper 还会为其创建请求的事务体。

此处由于是「会话创建」请求，因此会创建事务体：`request.txn = new CreateSessionTxn(to);`

##### 15. 注册与激活会话（Leader 级）

此处进行会话注册与激活的目的是**处理由非 Leader 节点转发过来**的「会话创建」请求。

因为是非 Leader 节点转发过来的请求，因此，在此时 Leader 节点的`SessionTracker`中并没有进行会话的注册，所以需要在此处再进行一次注册与激活。

#### 4. 事务处理

##### 16. 将请求交给 ProposalRequestProcessor 处理器

完成对请求的预处理后，`PrepRequestProcessor`处理器会将请求通过`nextProcessor.processRequest(request);`交付给自己的下一级处理器 `ProposalRequestProcessor`。

`ProposalRequestProcessor`是一个与提案相关的处理器线程，而所谓的提案，就是 ZooKeeper 将针对事务请求所展开的一个投票流程进行包装而成的对象，其中封装了一系列对事务的操作。 

从`ProposalRequestProcessor`处理器开始，请求的处理将会进入三个子处理流程，分别是Sync流程、Proposal流程和Commit流程。

###### Sync 流程

###### Proposal 流程

###### Commit 流程

#### 5. 事务应用

##### 17. 交付给 FinalRequestProcessor 处理器

请求流转到`FinalRequestProcessor`处理器后，也就接近请求处理的尾声了。

`FinalRequestProcessor`处理器会首先检查`outstandingChanges`队列中请求的有效性，如果发现这些请求已经落后于当前正在处理的请求，那么直接
从`outstandingChanges`队列中移除。

之后调用`zks.processTxn(hdr, txn);`处理当前事务。

##### 18. 事务应用

在之前的请求处理逻辑中，我们仅仅是将该事务请求记录到了事务日志中去，而内存数据库中的状态尚未变更。因此，在这个环节，我们需要将事务变更应用到内存数据库中。

需要注意的一点是，对于「会话创建」这类事务请求，ZooKeeper 做了特殊处理：因为在 ZooKeeper 内存中，会话的管理都是由`SessionTracker`负责的，而在之前的步骤中，ZooKeeper 已经将会话信息注册到了`SessionTracker`中，因此此处无须对内存数据库做任何处理，**只需要再次向`SessionTracker`进行会话注册即可**。

##### 19. 将事务请求放入 commitProposal 队列 

一且完成事务请求的内存数据库应用，就可以调用`zks.getZKDatabase().addCommittedProposal(request);`将该请求放入`committedLog`队列中。

`committedLog`队列用来保存最近被提交的事务请求，以便集群间机器进行数据的快速同步（在数据同步过程中，`LearnerHandler`线程会调用`leader.zk.getZKDatabase().getCommittedLog();`获取最近提交的事务并进行同步处理）。

#### 6. 会话响应

至此，客户端的「会话创建」请求已经在 ZooKeeper 上所有负责请求处理链路的请求处理器间完成了流转。

##### 20. 统计处理

此处，ZooKeeper 会计算请求在服务端处理所花费的时间，同时还会统计客户端连接的一些基本信息，包括最新的 ZXlD`lastZxid`、最后一次和服务端的操作`lastOp`和 最后一次请求处理所花费的时间`lastLatency`等。

在完成这些操作后，`FinalRequestProcessor`会调用`zks.finishSessionInit(request.cnxn, true);`进行最后的响应处理。

#####21. 创建响应 ConnectResponse

`ConnectResponse`就是一个会话创建成功后的响应，包含了当前客户端与服务端之间的通信协议版本号`protocolVersion`、会话超时时间`timeOut`、`sessionId`和会话密码`passwd`。

##### 22.  序列化 ConnectResponse

##### 23. IO层发送响应给客户端

## 会话管理

### 分桶策略

指将类似的会话分配到同一区块中进行管理，以便于 ZooKeeper 对不同区块的会话进行隔离处理，同一区块的会话进行统一处理。分配的原则是每个会话的下次超时时间`nextExpirationTime` 。

`nextExpirationTime`是指该会话最近一次可能超时的时间，对于一个新创建的会话来说，其会话创建完毕后，ZooKeeper 就会为其计算`nextExpirationTime`，计算方式为：`nextExpirationTime = (time / expirationInterval + 1) * expirationInterval;`

其中，`time`的值在初始化`nextExpirationTime`时，为`System.currentTimeMillis()`；而进行会话激活时，则为`System.currentTimeMillis() + timeout`。 `expirationInterval`的值为`tickTime`的默认值，单位是毫秒。

### 会话激活

为了保持客户端会话的有效性，在 ZooKeeper 的运行过程中，客户端会在会话超时时间过期范围内向服务端发送PING请求来保持会话的有效性，这就是俗称的「心跳检测」。与此同时，服务端需要不断地接收来自客户端的这个「心跳检测」，井且需要重新激活对应的客户端会话，我们将这个重新激活的过程称为`touchSession`。

通过会话激活`touchSession`，服务端能检测到对应客户端的存活性，同时客户端也能自己保持连接状态。

#### Leader 服务器激活客户端会话的流程

##### 1. 检验该会话是否已经被关闭

Leader 会检查该会话是否已经被关闭，如果该会话已经被关闭，那么不再继续激活该会话。

##### 2. 计算该会话新的超时时间`expireTime`

如果该会话尚未关闭，则开始激活会话。

首先需要计算出该会话下一次超时时间点，并与旧的下次会话超时时间点`tickTime`进行比较，若`expireTime`早于（小于）`tickTime`，则不进行任何操作。

##### 3. 迁移会话

如果`expireTime`迟与（大于等于）`tickTime`，则将该会话从老区块（通过`tickTime`定位老区块）中取出并删除，放入`expireTime`对应的新区块中。

#### 发起方式

心跳检测由客户端主动发起，以请求的形式向服务端发送。

1. 只要客户端向服务端发送请求，包括读或写请求，那么就会触发一次会话激活。

2. 如果客户端发现在`sessionTimeout / 3`的时间内，未和服务器进行过任何通信，即没有向服务端发送任何请求，那么就会主动发起一个PING请求，服务端收到该请求后，就会触发一次会话激活。

#### 会话超时检查

`SessionTrackerImpl`类本身就是一个专门进行「会话超时检查」的线程，其工作机制就是：定时对会话桶中剩下的、未被迁移的会话进行逐个检查并清理。

##### 超时检查线程是如何做到定时检查的呢？

基于「分桶策略」，`SessionTrackerImpl`线程在以`expirationInterval`为倍数的时间点上进行检查，既提高了会话检查的效率，而且采用批量清理的方式使检查操作具有较好的性能。

## 会话清理

当`SessionTracker`的会话超时检查线程整理出一些已经过期的会话后，就要开始进行会话清理了。

### 1. 标记会话状态为「己关闭」

### 2. 发起「会话关闭」请求

### 3. 收集需要清理的临时节点

### 4. 添加「节点删除」事务变更

### 5. 删除临时节点

### 6. 移除会话

### 7. 关闭 NIOServerCnxn

## 重连

### 连接断开： CONNECTION_LOSS

### 会话失效： SESSION_EXPIRED

### 会话转移： SESSION_MOVED

## 场景

### 统一配置管理

### 分组管理

path结构

### 统一命名

sequential

### 同步

临时节点

#### 分布式锁

锁依托一个父节点，且具备 -s 

代表父节点下可以有多把锁

##### 队列式事务的锁

#### HA 选主
