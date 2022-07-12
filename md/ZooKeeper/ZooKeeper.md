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

在 ZooKeeper 中， 接口类`Watcher`用于表示一个标准的事件处理器， 其定义了事件通知相关的逻辑。

在`Watcher`的内部接口类`Event`中，包含`KeeperState`和`EventType`两个枚举类， 分别代表了通知状态和事件类型

`Watcher`中还定义了事件的回调方法：`process (WatchedEvent event)`。

### Watcher事件

[Watcher通知状态与事件类型一览]()

同一个事件类型在不同的通知状态中代表的含义有所不同。

### 回调方法process ()

当 ZooKeeper 向客户端发送一个 Watcher 事件通知时，客户端就会回调相应的`process`方法对服务端事件进行处理。

在这个服务端传递到客户端的过程中，ZooKeeper 使用了`WatchedEvent`对象来封装服务端事件并传递给客户端，`WatchedEvent`也是 ZooKeeper 整个 Watcher 通知机制的最小通知单元。

但是`WatchedEvent`实际上只是一个逻辑事件， 是服务端和客户端程序执行过程中所需的逻辑对象，为了便于传输，ZooKeeper 使用了实现序列化接口的`WatcherEvent`对象来作为网络传输对象。

服务端在生成`WatchedEvent`事件之后，会调用`getWrapper`方法将自己包装成一个可序列化的`WatcherEvent`事件，之后通过网络传输到客户端。

客户端在接收到这个服务端的事件对象后， 首先会将`WatcherEvent`事件还原成一个`WatchedEvent`事件， 并传递给`process`方法处理，而回调方法`process`的入参也就是这个`WatchedEvent`事件。

通过这种方式我们就能够解析出完整的服务端事件了。

还需要注意的一点是， 无论是`WatcherEvent`还是`WatchedEvent`，其对 ZooKeeper 服务端事件的封装都是极其简单的，只包含了每一个事件的三个基本属性： 通知状态`keeperState`、事件类型`eventType`和节点路径`path`。

因此客户端无法直接从该事件中获取到对应数据节点的原始数据内容以及变更后的新数据内容， 而是需要客户端再次主动去重新获取数据，这也是 Watcher 机制的一个非常重要的特性。

### 工作机制

#### 1. 客户端注册Watcher

##### 创建一个 ZooKeeper 客户端对象实例时， 可以向构造方法中传入一个默认的 Watcher

这个 Watcher 将作为整个 ZooKeeper 会话期间的默认 Watcher，会一直被保存在客户端`ZKWatchManager`的`defaultWatcher`中。

##### 通过其他方法注册 Watcher

除了构造方法，ZooKeeper 客户端还可以通过`getData`、`getChildren`和`exist`这三个方法来向 ZooKeeper 服务器注册 Watcher，但无论使用哪种方式，注册 Watcher 的工作原理都是一致的。

下面**以`getData`方法为例**来说明：

`getData`方法用于获取指定节点的数据内容，其实它主要有两个方法：`public byte[] getData(String path, boolean watch, Stat stat)`、`public byte[] getData(final String path, Watcher watcher, Stat stat)`。前者通过一个 boolean 参数来标识是否使用默认 Watcher 来进行注册；两个方法的具体注册逻辑是一致的。

在向`getData`方法注册 Watcher 后， 客户端首先创建当前客户端请求`GetDataRequest`，并根据是否**使用Watcher监听**进行标记 ，同时会封装一个 Watcher 的注册信息`WatchRegistration` 对象， 用于暂时保存数据节点路径和 Watcher 的对应关系。

在 ZooKeeper 中，`Packet`可以被看作一个最小的通信协议单元，用于进行客户端与服务端之间的网络传输。

任何需要传输的对象都需要包装成一个`Packet`对象，因此在`ClientCnxn`中`WatchRegistration`会被封装到`Packet`中去，然后放入发送队列中等待客户端发送。

随后，ZooKeeper 客户端就会向服务端发送这个请求，同时等待请求的返回。

客户端完成请求发送后，会由`ClientCnxn`中的`SendThread`线程的`readResponse`方法负责接收来自服务端的响应，最后在`finishPacket`方法中，`Packet` 中 `WatchRegistration`调用`register`方法将对应的 Watcher 并注册到`ZKWatchManager`中去（因为是以`getData`方法为例，`WatchRegistration`的实现类`DataWatchRegistration`最终会将 Watcher 保存到`ZKWatchManager`的`dataWatches`中去）。

##### Watcher 实体的所有数据都会随着客户端请求被发送到服务端吗?

不会。

如果客户端注册的所有 Watcher 都被传递到服务端的话， 那么服务端肯定会出现内存紧张或其他性能问题了。

在 ZooKeeper 的设计中充分考虑到了这个问题：在上面的流程中， 我们虽然把 `WatchRegistration` 封装到了`Packet`对象中去， 但事实上， 在实际的底层网络传输序列化过程中， 并没有将`WatchRegistration`对象完全地序列化到底层字节数组中去。

在`Packet.createBB()`方法中， ZooKeeper 只会将`requestHeader`和`request`两个属性进行序列化。

#### 2. 服务端处理Watcher

#### 3. 客户端回调Watcher

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

# 会话

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
