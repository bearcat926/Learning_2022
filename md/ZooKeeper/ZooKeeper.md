# ZooKeeper

这是一个分布式协调服务框架，是Apache Hadoop 的一个子项目。

可以用来解决分布式应用中经常遇到的一些数据管理问题，如：统一命名服务、状态同步服务、集群管理、分布式应用配置项的管理等。

举个例子，分布式协调服务很难做好，特别容易出现竞争条件和死锁等错误，而使用 ZooKeeper 可以降低分布式应用从零开始实现协调服务的难度。

[ZooKeeper 官网文档](https://zookeeper.apache.org/doc/current/zookeeperOver.html)

## 保证

### 1. 顺序一致性

ZooKeeper 是主从模型，因此所有的写操作都会在「主节点」上执行，即由单机来维护客户端指令的顺序与他们发送至 ZooKeeper 的顺序一致一致。

### 2. 原子性

在 ZooKeeper 中，事务是指能够改变 ZooKeeper 服务器状态的操作。一般包括 Znode 的创建与删除、内容更新和 Session 创建与失效等操作。

类似事务的原子性，一条命令发送到 ZooKeeper 集群，要么执行成功要么失败，没有中间状态。

### 3. 单一系统映像

客户端将看到相同的服务视图，而不管它连接到集群中的哪一台 ZooKeeper 服务器。

即便开启会话时，

### 4. 可靠性（持久性）

一旦应用更新，数据将被持久化，直到数据被再次更新。

对于该保证有两个推论：

1、如果客户端得到了成功的返回码，说明写入成功，数据被持久化，如果出现了通信错误，超时等一些故障，客户端将不知道更新是否已应用。我们采取措施尽量减少失败，但唯一的保证是只有成功的返回码。 （这在Paxos中称为单调性条件。）

2、如果客户端已经读取到了数据或者写入成功了数据，都不会因为zk的失败而导致回滚。

### 5. 及时性（最终一致性）

客户端看到的服务视图保证在特定时间范围内是最新的。

# Watcher（事件监听器）

ZooKeeper 允许客户端注册一些 Watcher，去监听它关心的目录节点，而且当目录节点发生变化（如数据改变、被删除、子目录节点增加删除）时，ZooKeeper 会将事件通知客户端（通过节点的变化产生事件）

## Watcher通知机制

Watcher机制主要包括客户端线程、客户端 Watch Manager 和 ZooKeeper 服务器三部分。

工作流程可以简述为，客户端在向 ZooKeeper 服务器注册 Watcher 的同时，会将 Watcher 对象存储在客户端的 Watch Manager 中。

当 ZooKeeper 服务器端触发 Watcher 事件后，会向客户端发送通知，客户端线程从 Watch Manager 中取出对应的 Watcher 对象来执行回调逻辑。

### Watcher接口

在 ZooKeeper 中，接口类`Watcher`用于表示一个标准的事件处理器，其定义了事件通知相关的逻辑。

在`Watcher`的内部接口类`Event`中，包含`KeeperState`和`EventType`两个枚举类，分别代表了通知状态和事件类型

`Watcher`中还定义了事件的回调方法：`process (WatchedEvent event)`。

### Watcher事件

同一个事件类型在不同的通知状态中代表的含义有所不同。

### #回调方法process

当 ZooKeeper 向客户端发送一个 Watcher 事件通知时，客户端就会回调相应的`process`方法对服务端事件进行处理。

在这个服务端传递到客户端的过程中，ZooKeeper 使用了`WatchedEvent`对象来封装服务端事件并传递给客户端，`WatchedEvent`也是 ZooKeeper 整个 Watcher 通知机制的最小通知单元。

但是`WatchedEvent`实际上只是一个逻辑事件，是服务端和客户端程序执行过程中所需的逻辑对象，为了便于传输，ZooKeeper 使用了实现序列化接口的`WatcherEvent`对象来作为网络传输对象。

服务端在生成`WatchedEvent`事件之后，会调用`getWrapper`方法将自己包装成一个可序列化的`WatcherEvent`事件，之后通过网络传输到客户端。

客户端在接收到这个服务端的事件对象后，首先会将`WatcherEvent`事件还原成一个`WatchedEvent`事件，并传递给`process`方法处理，而回调方法`process`的入参也就是这个`WatchedEvent`事件。

通过这种方式我们就能够解析出完整的服务端事件了。

还需要注意的一点是，无论是`WatcherEvent`还是`WatchedEvent`，其对 ZooKeeper 服务端事件的封装都是极其简单的，只包含了每一个事件的三个基本属性： 通知状态`keeperState`、事件类型`eventType`和数据节点路径`path`。

因此客户端无法直接从该事件中获取到对应数据节点的原始数据内容以及变更后的新数据内容，而是需要客户端再次主动去重新获取数据，这也是 Watcher 机制的一个非常重要的特性。

### 工作机制

#### 1. 客户端注册Watcher

##### 创建一个 ZooKeeper 客户端对象实例时，可以向构造方法中传入一个默认的 Watcher

这个 Watcher 将作为整个 ZooKeeper 会话期间的默认 Watcher，会一直被保存在客户端`ZKWatchManager`的`defaultWatcher`中。

##### 通过其他方法注册 Watcher

除了构造方法，ZooKeeper 客户端还可以通过`getData`、`getChildren`和`exist`这三个方法来向 ZooKeeper 服务器注册 Watcher，但无论使用哪种方式，注册 Watcher 的工作原理都是一致的。

下面**以`getData`方法为例**来说明：

`getData`方法用于获取指定节点的数据内容，其实它主要有两个方法：`public byte[] getData(String path，boolean watch，Stat stat)`、`public byte[] getData(final String path，Watcher watcher，Stat stat)`。前者通过一个 boolean 参数来标识是否使用默认 Watcher 来进行注册；两个方法的具体注册逻辑是一致的。

在向`getData`方法注册 Watcher 后，客户端首先创建当前客户端请求`GetDataRequest`，并根据是否**使用Watcher监听**进行标记，同时会封装一个 Watcher 的注册信息`WatchRegistration` 对象，用于暂时保存数据节点路径和 Watcher 的对应关系。

在 ZooKeeper 中，`Packet`可以被看作一个最小的通信协议单元，用于进行客户端与服务端之间的网络传输。

任何需要传输的对象都需要包装成一个`Packet`对象，因此在`ClientCnxn`中`WatchRegistration`会被封装到`Packet`中去，然后放入发送队列中等待客户端发送。

随后，ZooKeeper 客户端就会向服务端发送这个请求，同时等待请求的返回。

客户端完成请求发送后，会由`ClientCnxn`中的`SendThread`的`readResponse`方法负责接收来自服务端的响应，最后在`finishPacket`方法中，`Packet` 中 `WatchRegistration`调用`register`方法将对应的 Watcher 并注册到`ZKWatchManager`中去（因为是以`getData`方法为例，`WatchRegistration`的实现类`DataWatchRegistration`最终会将 Watcher 保存到`ZKWatchManager`的`dataWatches`中去）。

##### Watcher 实体的所有数据都会随着客户端请求被发送到服务端吗?

不会。

如果客户端注册的所有 Watcher 都被传递到服务端的话，那么服务端肯定会出现内存紧张或其他性能问题了。

在 ZooKeeper 的设计中充分考虑到了这个问题：在上面的流程中，我们虽然把 `WatchRegistration` 封装到了`Packet`对象中去，但事实上，在实际的底层网络传输序列化过程中，并没有将`WatchRegistration`对象完全地序列化到底层字节数组中去。

在`Packet.createBB()`方法中，ZooKeeper 只会将`requestHeader`和`request`两个属性进行序列化。

#### 2. 服务端处理Watcher

##### 将 ServerCnxn 存储到 WatchManager

服务端收到来自客户端的请求之后，在 `FinalRequestProcessor`的`processRequest()`中会判断当前请求是否需要注册 Watcher。

下面**以对`getData`请求的处理为例**来说明：

从`getData`请求的处理逻辑中，我们可以看到，当`getDataRequest.getWatch()`为 true 的时候，ZooKeeper 就认为当前客户端请求需要进行 Watcher 注册，于是就会将数据节点路径`path`、新创建的`Stat`对象以及当前的`ServerCnxn`对象一起传入`zks.getZKDatabase().getData()`方法中去。最终，它们会被存储在`WatchManager`的`watchTable`和`watch2Paths`中。

`ServerCnxn`是一个 ZooKeeper 客户端和服务器之间连接的抽象类，代表了一个客户端和服务器的连接。`ServerCnxn`类的默认实现是`NIOServerCnxn`，而且从3.4.0 版本开始，引入了基于Netty的实现：`NettyServerCnxn`。但无论采用哪种实现方式，都实现了 Watcher 的`process`方法，因此我们可以把`ServerCnxn`看作是一个 Watcher 对象。

`WatchManager`是 ZooKeeper 服务端 Watcher 的管理者，其内部管理的`watchTable`和`watch2Paths`两个存储结构，分别从两个维度对 Watcher 进行存储。前者是从数据节点路径的粒度来托管 Watcher，后者是从 Watcher 的粒度来控制事件需要触发的数据节点。同时，`WatchManager`还负责 Watcher 事件的触发，并移除那些已经被触发的 Watcher。

在服务端，`DataTree`中会托管两个`WatchManager`，分别是`dataWatches`和`childWatches`，分别对应数据变更 Watcher 和子节点变更 Watcher。在本例中，因为是`getData`方法，因此最终会被存储在`dataWatches`中。

##### Watcher 触发

无论是`dataWatches`还是`childWatches`，Watcher的触发逻辑都是一致的。

###### 1.封装`WatchedEvent`

首先将通知状态`KeeperState`、事件类型`EventType`以及节点路径`Path`封装成一个`WatchedEvent`对象。

###### 2.查询 Watcher

根据数据节点路径，从`watchTable`中取出对应的 Watcher。

如果没有找到 Watcher，说明没有任何客户端在该数据节点上注册过 Watcher，直接退出；而如果找到了这个 Watcher，服务端会将其提取出来，同时会直接从`watchTable`和`watch2Paths`中将其删除。

从这里我们也可以看出，Watcher 在服务端是一次性的，即触发一次就失效了。

###### 3. 调用 process 方法来触发 Watcher 

在这一步中，会逐个依次地调用从上一步中找出的、所有 Watcher 的process方法。

因为 ZooKeeper 会在注册 Watcher 时，将请求中的`ServerCnxn`作为一个 Watcher 进行存储，所以这里调用的`process`方法，事实上就是`ServerCnxn`的`process`方法。

以`NIOServerCnxn`的`process`方法为例，它先创建了一个`ReplyHeader`对象，将其请求头中标记-1，表明当前是一个通知；再将`WatchedEvent`包装成`WatcherEvent`对象，以便进行网络传输序列化；最后向客户端发送该通知。

上述方法的逻辑非常简单，本质上是借助当前客户端连接的 `ServerCnxn` 对象来实现向客户端传递`WatchedEvent`对象，而真正的客户端 Watcher 回调与业务逻辑都在客户端执行。

#### 3. 客户端回调Watcher

##### SendThread 接收事件通知

对于一个来自服务端的响应，客户端都是由`SendThread`的`readResponse(ByteBuffer incomingBuffer)`方法来进行统一处理的，大体上可以将处理逻辑分为以下4个步骤：

###### 1. 反序列化

ZooKeeper 客户端接到请求后，首先会将字节流转换成`WatcherEvent`对象。

###### 2. 处理 chrootPath

首先根据响应头`ReplyHeader`中的XID判断究竟是哪种类型的响应。如果标识了`XID = -1`，就表明这是一个通知类型的响应。

如果客户端设置了`chrootPath`属性，那么需要对服务端传过来的完整的节点路径进行`chrootPath`处理，生成客户端的一个相对节点路径。例如，客户端设置了`chrootPath`为`/app1`，那么针对服务端传过来的响应包含的节点路径为`/app/locks`。

 经过`chrootPath`处理后，就会变成一个相对路径`/locks`。

###### 3. 还原 WatchedEvent

在#回调方法process 中提到，process方法的参数定义是`WatchedEvent`，因此这里需要将`WatcherEvent`对象转换成`WatchedEvent`对象。

###### 4. 回调 Watcher

最后调用`eventThread.queueEvent(we)`将`WatchedEvent`对象交给`EventThread`，在下一个轮询周期中进行 Watcher 回调。

##### EventThread 处理事件通知

`EventThread`是 ZooKeeper 客户端中专门用来处理服务端通知事件的线程，处理逻辑如下：

###### 1. 创建 `WatcherSetEventPair`

在`SendThread `中调用的`queueEvent`方法首先会调用`ZooKeeper.materialize`方法，并通过其返回值创建一个对象`WatcherSetEventPair`。

###### 2. 取出并移除对应的 Watcher

在`materialize`方法中，客户端会根据通知事件类型`EventType`，从相应的 Watcher 存储中去除对应的 Watcher。

注意，**此处使用的是`remove`方法**，因此也表明了客户端的 Watcher 机制同样也是一次性的，即一旦被触发后，该 Watcher 就失效了。

###### 3. 将 Watcher 放入`waitingEvents`队列

获取到相关的所有 Watcher 之后，客户端会通过它们创建`WatcherSetEventPair`对象，并放入`waitingEvents`这个队列中去。

`WaitingEvents`是一个存储待处理 Watcher 的队列，`EventThread`的`run`方法会不断对该队列进行处理。

###### 4. `EventThread`处理 Watcher

`EventThread`每次都会从`waitingEvents`队列中取出一个 Watcher，并调用`processEvent`方法进行串行同步处理。

此处的`processEvent`方法中的 Watcher 才是之前客户端真正注册的 Watcher，在其中调用`process`方法就可以实现对 Watcher 的回调了。

## 特性

### 一次性

无论是服务端还是客户端，一个 Watcher 一旦被触发，ZooKeeper 都会想起从相应的储存中移除

这样的设计有效地减轻了服务器的压力

开发人员需要注意在 Watcher 的使用上要反复注册

### 客户端串行执行

客户端 Watcher 回调的过程是一个串行同步的过程，这保证了顺序性。

开发人员需要注意千万不要因为一个 Watcher 的处理逻辑影响了整个客户端的 Watcher 回调

### 轻量级

WatchedEvent 是 ZooKeeper 整个 Watcher 通知机制的最小通知单元，整个数据结构中只包含了三部分：通知状态、事件类型和节点路径

也就是说 Watcher 通知只会告诉客户端在哪里发生了什么事件，而不会说明事件的具体内容，这需要客户端主动去重新获取数据

另外，客户端向服务端注册 Watcher 时，并不会把客户端真实的 Watcher 对象传递到服务端，仅仅是在客户端请求中使用 boolean 类型属性进行了标记，同时服务端也仅仅只是保存了当前连接的 ServerCnxn 对象

# ACL（访问控制列表，Access Control Lists ）策略

## UGO

针对一个文件或目录，对创建者`User`、创建者所在的组`Group`和其他所有用户`Other`分别配置不同的权限。

是一种在 Unix/Linux 文件系统中使用的粗粒度文件系统权限控制模式，也是目前应用最广泛的权限控制方式。

## ACL

即访问控制列表，是一种相对来说比较新颖且更细粒度的权限管理方式，可以针对任意用户和组进行细粒度的权限控制。

目前绝大部分 Unix 系统都已经支持了ACL方式的权限控制，Linux 也从2.6版本的内核开始支持这个特性。

### 权限模式：Scheme

IP：通过 IP 地址粒度来进行权限控制。

Digest：最常用的权限控制模式，以类似于 **username:password** 形式的标识来进行权限配置，便于区分不同应用来进行权限控制。

World：最开放的权限控制模式，这种权限控制方式几乎没有任何作用，所有用户都可以在不进行任何权限校验的情况下操作 ZooKeeper 上的数据。

Super：超级用户可以对任意ZooKeeper上的数据节点进行任何操作。

### 授权对象：ID

授权对象指权限赋予的用户或一个指定实体，例如 IP地址或是机器等。在不同的权限模式下，授权对象是不同的。

### 权限： Permission

**CREATE**：**子节点**的创建权限

**READ**：**数据节点**的读取权限和**子节点**的列表读取权限

**WRITE**：**数据节点**的更新权限

**DELETE**：**子节点**的删除权限

**ADMIN**：**数据节点**的管理权限，设置节点 ACL 的权限

### ACL管理

#### 设置ACL

##### 1. 在数据节点创建的同时进行 ACL 设置：`create [-s] [-e] path data acl`

##### 2. 对已经存在的数据节点进行 ACL 设置：`setAcl path acl`

#### Super模式的用法

##### 如何开启Super模式

在 ZooKeeper 服务器启动的时候，添加如下命令参数`-Dzookeeper.DigestAuthenticationProvider.superDigest=foo:kWN6aNSbjcKWPqjiV7cg8N24raU=`

`foo`代表了一个超级管理员的用户名；`kWN6aNSbjcKWPqjiV7cg8N24raU=`是可变的，由 ZooKeeper 的系统管理员来进行自主配置

##### 如果一个持久数据节点包含了 ACL 权限控制，但其创建者客户端已经退出或已不再使用，那么这些数据节点该如何清理呢？

 可以在 ACL 的 Super 模式下，使用超级管理员权限清理该数据节点

# 客户端

## 核心组件

### ZooKeeper实例

客户端的入口

### ClientWatchManager

客户端 Watcher 管理器

### HostProvider

客户端地址列表管理器

### ClientCnxn

客户端网络连接器，其内部又包含两个线程，即`SendThread`和`EventThread`。

前者是一个IO线程，主要负责 ZooKeeper 客户端和服务端之间的网络IO通信；后者是一个事件线程，主要负责对服务端事件进行处理。

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

# 会话

## 会话状态

会话在整个生命周期中， 会在不同的状态之间进行切换， 这些状态一般可以分为`CONNECTING`、`CONNECTED`、`RECONNECTING`、`RECONNECTED`和`CLOSE`等。

## 会话创建

## 会话管理

## 会话清理





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
