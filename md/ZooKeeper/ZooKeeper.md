# 会话

## Sync 流程

「Sync 流程」的核心就是使用`SyncRequestProcessor`处理器记录事务日志的过程。

`ProposalRequestProcessor`处理器在接收到一个上级处理器流转过来的请求后，首先会判断该请求是否是事务请求。针对每个事务请求，都会通过事务日志的形式将其记录下来。

Leader 服务器和 Follower 服务器的请求处理链路中都会有这个处理器，两者在事务日志的记录功能上是完全一致的。完成事务日志记录后，每个 Follower 服务器都会向 Leader 服务器发送 ACK 消息，表明自身完成了事务日志的记录，以便 Leader 服务器统计每个事务请求的投票情况。

## Proposal 流程

在 ZooKeeper 的实现中，每一个事务请求都需要集群中过半机器投票认可才能被真正应用到 ZooKeeper 的内存数据库中去，这个投票与统计过程被称「Proposal流程」

### 1. 发起投票

如果当前请求是事务请求，那么`ProposalRequestProcessor`处理器就会调用`zks.getLeader().propose(request);`让 Leader 服务器发起一轮事务投票。

在发起事务投票之前，首先会检查当前服务端的 ZXID 是否可用。若不可用，那么将会抛出`XidRolloverException`异常。

### 2. 生成提议 Proposal

如果当前服务端的ZXID可用，那么就可以开始事务投票了。

ZooKeeper 会将消息类型、ZXID、之前创建的事务中的请求头`TxnHeader`和请求体`CreateSessionTxn（创建会话中的请求体）`以及权限信息`authinfo`放入数据包对象`QuorumPacket`中，再将其和请求对象`request`序列化到提议对象`Proposal`中，`Proposal`对象代表了一次针对 ZooKeeper 服务器状态的变更申请。

### 3. 广播提议

生成提议后， Leader 服务器会以 ZXID 作为标识，将该提议放入投票箱队列`outstandingProposals`中，同时会将该提议广播给所有的 Follower 服务器。

### 4. 收集投票

Follower 服务器在`followLeader`方法中接收到 Leader 发来的这个提议后，会在`processPacket`方法中调用`fzk.logRequest(hdr, txn);`进入 Sync 流程来进行事务日志的记录。

一且日志记录完成后，就会发送 ACK 消息给 Leader 服务器，而 Leader 服务器会根据这些 ACK 消息来统计每个提议的投票情况。

当一个提议获得了集群中过半机器的投票，那么就认为该提议通过，接下去就可以进入提议的 Commit 阶段了。

### 5. 将请求放入 toBeApplied 队列（Leader）

在该提议被提交之前，ZooKeeper 会将其从`outstandingProposals`队列中移除，并放入`toBeApplied`队列中去。

### 6. 广播 COMMIT 消息

一且 ZooKeeper 确认一个提议已经可以被提交了，将会进入请求提交阶段，此时 Leader 服务器就会向 Follower 和 Observer 服务器发送 COMMIT 消息，以便所有服务器都能够提交该提议。

这里需要注意，由于 Observer 服务器并未参加之前的提议投票，因此 Observer 服务器尚未保存任何关于该提议的信息，所以在广播 COMMIT 消息的时候，需要区别对待，Leader 会调用`inform(p);`方法向其发送一个「INFORM 」消息，该消息体中包含了当前提议的内容。

而对于 Follower 服务器，由于已经保存了所有关于该提议的信息，因此 Leader 服务器只需要调用``commit(zxid);``向其发送 ZXID 即可。

最后 Leader 服务器调用`zk.commitProcessor.commit(p.request);`在本机上提交提议，即将该请求放入`committedRequests`队列中，同时#唤醒「Commit 流程」 （这一步非Leader服务器也会进行）

## Commit 流程

### 1. 将请求交付给  CommitProcessor  处理器

`ProposalRequestProcessor`处理器在接收到一个上级处理器流转过来的请求后，会先调用`nextProcessor.processRequest(request);`让`CommitProcessor`处理器将请求先放入`queuedRequests`队列中，等待之后处理。

### 2.  处理 queuedRequests 队列请求

`CommitProcessor`线程当检测到`queuedRequests`队列中已经有新的请求进来，就会逐个从队列中取出请求进行处理。

### 3. 标记 nextPending

如果从`queuedRequests`队列中取出的请求是一个事务请求（`create`、`delete`、`setData`、`multi`、`setACL`、`createSession`、`closeSession`以及非Leader服务器的`sync`），那么就需要进行集群中各服务器之间的投票处理，同时需要将`nextPending`标记为当前请求。

标记`nextPending`的作用，一方面是确保事务请求的顺序性，另一方面也是便于`CommitProcessor`线程检测当前集群中是否正在进行事务请求的投票。

### 4. 等待 Proposal 投票结果、Leader 唤醒「Commit 流程」

在「Commit 流程」处理的同时，Leader 已经根据当前事务请求生成了一个提议 Proposal，并广播给了所有的 Follower 服务器。

因此，在这个时候，「Commit 流程」需要等待投票结束，Leader 服务器广播 COMMIT 消息，并 #唤醒「Commit 流程」

### 5. 提交请求

一旦发现`committedRequests`队列中已经有可以提交的请求了，那么「Commit 流程」就会开始提交请求。

当然在提交以前，为了保证事务请求的顺序执行，「Commit 流程」流程还会对比之前标记的`nextPending`和`committedRequests`队列中第一个请求是否一致。

如果检查通过，那么`CommitProcessor`就会将该请求放入`toProcess`队列中，然后在`CommitProcessor`线程下一次重新运行时，将`toProcess`队列中的所有请求交付给下一个请求处理器，并对`toProcess`队列进行清理。

对于 Leader 服务器上的`CommitProcessor`而言，它的下一个请求处理器是`toBeAppliedProcessor`，而 Follower 和 Observer 服务器的下一个请求处理器则是`FinalRequestProcessor`处理。

## 会话清理

当`SessionTracker`的会话超时检查线程整理出一些已经过期的会话后，就要开始进行「会话清理」了。

### 1. 标记会话状态为「己关闭」

由于整个「会话清理」过程需要一段的时间，因此为了保证在此期间不再处理来自该客户端的新请求，`SessionTracker`会首先通过调用`setSessionClosing(s.sessionId);`将该会话的`isClosing`属性标记为true。使会话即使在清理期间接收到该客户端的新请求，也无法继续处理了。

### 2. 发起「会话关闭」请求

为了使对该会话的关闭操作在整个服务端集群中都生效，ZooKeeper 使用了提交「会话关闭」请求的方式，由实现了`SessionExpirer`接口的`ZooKeeperServer`在自身实现的`expire`方法中调用了`close`方法，再通过`submitRequest`方法将「会话关闭」请求交付给`PrepRequestProcessor`处理器进行处理。

### 3. 收集需要清理的临时节点

在 ZooKeeper 中，一且某个会话失效后，那么和该会话相关的「临时节点」都需要被一并清除掉。因此，在清理「临时节点」之前，首先需要将服务器上所有和该会话相关的「临时节点」都整理出来。

在 ZooKeeper 的内存数据库中，为每个会话都单独保存了一份由该会话维护的所有「临时节点」集合，因此在「会话清理」阶段，只需要根据当前即将关闭的会话的`sessionId`，从内存数据库中获取到这份「临时节点」列表即可：`zks.getZKDatabase().getEphemerals(request.sessionId);`。

此处获取「临时节点」列表之后，需要对`outstandingChanges`加锁以保证没有新事件发生。

还有一个细节需要处理，即在 ZooKeeper 处理会话关闭请求之前，正好有以下两类请求到达了服务端并正在处理中：

1. 节点删除请求，且删除的目标节点正好是上述「临时节点」中的一个。
2. 临时节点创建请求，且创建的目标节点正好是上述「临时节点」中的一个。

对于这两类请求，其共同点都是事务处理尚未完成，因此还没有应用到内存数据库中，所以上述获取到的「临时节点」列表在遇上这两类事务请求的时候，会存在不
致的情况，以下为对该情况的处理方法：

1. 针对第一类请求，我们需要将请求对应的数据节点路径从`ephemerals`中移除，以避免重复删除：通过判断`c.stat == null`来检查该 Znode 节点是否存在
2. 针对第二类请求，我们需要将请求对应的数据节点路径添加到`ephemerals`中去，以删除这些即将会被创建但是尚未保存到内存数据库中去的「临时节点」：通过判断`c.stat.getEphemeralOwner() == request.sessionId`来检查该 Znode 节点是否属于当前会话

### 4. 添加「节点删除」事务变更

完成该会话相关的「临时节点」收集后，ZooKeeper 会逐个将这些「临时节点」转换成节点删除请求，通过`addChangeRecord`方法放入事务变更队列 `outstandingChanges`中去。

### 5. 删除临时节点

上步中提交的请求最终会流转到`FinalRequestProcessor`处理器。

与「创建会话」的步骤比较相似，`FinalRequestProcessor`处理器也会检查`outstandingChanges`队列；但因为涉及到删除「临时节点」，还会检查`outstandingChangesForPath`队列。

之后又会逐步调用`zks.processTxn(hdr, txn);getZKDatabase().processTxn(hdr, txn);dataTree.processTxn(hdr, txn);...;killSession(header.getClientId(), header.getZxid());ephemerals.remove(session);...;deleteNode(path, zxid);`，以在内存数据库中删除该会话对应的所有临时节点。

### 6. 移除会话

除了上述删除步骤，对于「会话清理」这类事务请求，我们还需要将其从`SessionTracker`中移除，即让`ZooKeeperServer`通过`sessionTracker.removeSession(sessionId);`方法从`sessionsById`、`sessionsWithTimeout`和`sessionSets`中将该会话移除掉。

### 7. 关闭 NIOServerCnxn

`FinalRequestProcessor`处理器最终会调用`NIOServercnxnFactory`的`closeSession(request.sessionId);`方法唤醒`selector`，并在`closeSessionWithoutWakeup`方法中找到该会话对应的`NIOServerCnxn`，将其关闭。

## 重连

### 客户端再次连接上服务端时，可能的状态

#### CONNECTED

如果客户端在会话超时时间内重新连接上了ZooKeeper集群中任意一台机器，那么被视为重连成功。

#### EXPIRED

如果客户端是在会话超时时间以外重新连接上，那么服务端其实已经对该会话进行了会话清理操作，因此再次连接上的会话将被视为非法会话。

### 连接断开： CONNECTION_LOSS

`CONNECTION_LOSS`是指因为网络闪断，或是客户端当前连接的服务器出现问题而导致的客户端与服务器连接断开现象。

在这种情况下，如果客户端进行操作，就会立刻收到一个「Disconnected-None」事件的通知，同时会抛出异常`ConnectionLossException`。

#### 解决方法

应用需要做的事情就是捕获住这个异常，然后等待 ZooKeeper 的客户端自动完成重连。

一旦客户端成功连接上一台ZooKeeper机器后，那么客户端就会收到「SyncConnected-None」事件的通知，之后就可以重试刚刚出错的操作。

### 会话失效： SESSION_EXPIRED

`SESSION_EXPIRED`通常发生在`CONNECTION_LOSS`期间。

客户端和服务器连接断开之后，由于重连期间耗时过长，超过了会话超时时间`sessionTimeout`限制后还没有成功连接上服务器，那么服务器会认为这个会话已经结束了，就会开始进行会话清理。

但是另一方面，该客户端本身不知道会话已经失效，井且其客户端状态还是`DISCONNECTED`。因此当客户端重新连接上服务器时，将被服务器告知：该会话已经失效。

#### 解决方法

在这种情况下，用户需要重新实例化一个ZooKeeper对象，并且看应用的复杂情况，重新恢复临时数据。

### 会话转移： SESSION_MOVED

`SESSION_MOVED`是指客户端会话从一台服务器机器转移到了另一台服务器机器上。

假设客户端和服务器 *S1* 之间的连接已经断开了，如果客户端通过尝试重连后，成功连接上了新的服务器 *S2* 并且延续了有效会话，那么就可以说会话从 *S1* 转移到了 *S2* 上。

在3.2.0版本之后，ZooKeeper 明确提出了会话转移的概念，同时封装了`SessionMovedException`异常：在处理客户端请求的时候，会首先检查会话的所有者`Owner`，如果客户端请求的会话`Owner`不是当前服务器的话，那么就会直接抛出`SessionMovedException`异常。

`Owner`是在创建会话时，`PrepRequestProcessor`通过`pRequest2Txn`方法中调用`zks.setOwner(request.sessionId, request.getOwner());`指定的。

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

# 数据与存储

在 ZooKeeper 中，数据存储分为两部分：内存数据存储与磁盘数据存储。
