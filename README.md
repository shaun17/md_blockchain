有研究微服务网关权限的，在网关zuul中对所有下游服务权限做控制，覆盖到所有接口，权限控制到角色、菜单、按钮、方法。基于zuul纯内存的方式，校验时性能无损耗。参考我另一个项目 https://gitee.com/tianyalei/zuulauth

有对多线程并发调度框架感兴趣的，参考我另一个项目 https://gitee.com/tianyalei/asyncTool  该并发框架支持任意的多线程并行、串行、阻塞、依赖、回调，可以任意组合各线程的执行顺序，还带全链路回调。

# md_blockchain
Java区块链平台，基于Springboot开发的区块链平台。区块链qq交流群737858576，一起学习区块链平台开发，当然也交流Springboot、springcloud、机器学习等知识。
### 起因

公司要开发区块链，原本是想着使用以太坊开发个合约或者是使用个第三方平台来做，后来发现都不符合业务需求。原因很简单，以太坊、超级账本等平台都是做共享账本的，有代币和挖矿等模块。而我们需要的就是数家公司组个联盟，来共同见证、记录一些不可篡改的交互信息，如A公司给B公司发了一个xxx请求，B公司响应了什么什么。其实要的就是一个分布式数据库，而且性能要好，不能像比特币那种10分钟才生成一个区块。我们要的更多的是数据库的性能，和区块链的一些特性。

### 经过

项目于18年3月初开始研发，历时一月发布了第一版。主要做了存储模块、加密模块、网络通信、PBFT共识算法、公钥私钥、区块内容解析落地入库等。已经初步具备了区块链的基本特征，但在merkle tree、智能合约以及其他的一些细节上，尚不到位。

希望高手不吝赐教，集思广益，提出见解或方案，来做一个区块链平台项目，适合更多的区块链场景，而不仅仅是账本和各种忽悠人的代币。

理想中的区块链平台：

![输入图片说明](https://gitee.com/uploads/images/2018/0419/170921_7808ffdc_303698.png "1.png")

### 项目说明
主要有存储模块、网络模块、PBFT共识算法、加密模块、区块解析入库等。

该项目属于"链"，非"币"。不涉及虚拟币和挖矿。

### 存储模块
Block内存储的是类Sql语句。联盟间预先设定好符合业务场景需要的数据库表结构，然后设定好各个节点对表的操作权限（ADD，UPDATE，DELETE），将来各个节点就可以按照自己被允许的权限，进行Sql语句的编写，并打包至Block中，再全网广播，等待全网校验签名、权限等信息的合法性。如果Block合法，则进入PBFT共识算法机制，各节点开始按照PrePrepare、Prepare、Commit等状态依次执行，直到2f+1个commit后，开始进行本地生成新区块。新区块生成后，各节点进行区块内容解析，并落地入库的操作。

场景就比较广泛了，可以设定不同的表结构，或者多个表，进而能完成各自类型信息的存储。譬如商品溯源，从生产商、运输、经销商、消费者等，每个环节都可以对某个商品进行ADD信息的操作。

存储采用的是key-value数据库rocksDB，了解比特币的知道，比特币用的是levelDB，都是类似的东西。可以通过修改yml中db.levelDB为true，db.RocksDB为false来动态切换使用哪个数据库。

结构类似于sql的语句，如ADD（增删改） tableName（表名）ID（主键） JSON（该记录的json）。这里设置了回滚的逻辑，也就是当你做了一个ADD操作时，会同时存储一条Delete语句，以用于将来可能的回滚操作。



### 网络模块
网络层，采用的是各节点互相长连接、断线重连，然后维持心跳包。网络框架使用的是t-io，也是oschina的知名开源项目。t-io采用了AIO的方式，在大量长连接情况下性能优异，资源占用也很少，并且具备group功能，特别适合于做多个联盟链的SaaS平台。并且包含了心跳包、断线重连、retry等优秀功能。

在项目中，每个节点即是server，又是client，作为server则被其他的N-1个节点连接，作为client则去连接其他N-1个节点的server。同一个联盟，设定一个Group，每次发消息，直接调用sendGroup方法即可。

但仍需要注意的是，由于项目采用了pbft共识算法，在达到共识的过程中，会产生N的3次方数量的网络通信，当节点数量较多，如已达到100时，每次共识将会给网络带来沉重的负担。这是算法本身的限制。

### 共识模块PBFT

分布式共识算法是分布式系统的核心，常见的有Paxos、pbft、bft、raft、pow等。区块链中常见的是POW、POS、DPOS、pbft等。

比特币采用了POW工作量证明，需要耗费大量的资源进行hash运算（挖矿），由矿工来完成生成Block的权利。其他多是采用选举投票的方式来决定谁来生成Block。共同的特点就是只能特定的节点来生成区块，然后广播给其他人。

区块链分如下三类：

私有链：这是指在企业内部部署的区块链应用，所有节点都是可以信任的，不存在恶意节点；

联盟链：半封闭生态的交易网络，存在不对等信任的节点，可能存在恶意节点；

公有链：开放生态的交易网络，为联盟链和私有链等提供全球交易网络。

由于私有链是封闭生态的存储系统，因此采用Paxos类共识算法（过半同意）可以达到最优的性能；联盟链有半公开半开放特性，因此拜占庭容错是适合选择之一，例如IBM超级账本项目；对于公有链来说，这种共识算法的要求已经超出了普通分布式系统构建的范畴，再加上交易的特性，因此需要引入更多的安全考虑。所以比特币的POW是个非常好的选择。

我们这里可选的是raft和pbft，分别做私链和联盟链，项目中我使用了修改过的pbft共识算法。

先来简单了解pbft：

（1）从全网节点选举出一个主节点（Leader），新区块由主节点负责生成。

（2）每个节点把客户端发来的交易向全网广播，主节点将从网络收集到需放在新区块内的多个交易排序后存入列表，并将该列表向全网广播。

（3）每个节点接收到交易列表后，根据排序模拟执行这些交易。所有交易执行完后，基于交易结果计算新区块的哈希摘要，并向全网广播。

（4）如果一个节点收到的2f（f为可容忍的拜占庭节点数）个其它节点发来的摘要都和自己相等，就向全网广播一条commit消息。

（5）如果一个节点收到2f+1条（包括自己）commit消息，即可提交新区块到本地的区块链和状态数据库。

（6）客户端收到f + 1个成功（即便有f个失败、再f个恶意返回的错误信息，f + 1个正确的也是多数派）的返回，即可认为该次写入请求是成功的。

可以看到，传统的pbft是需要先选举出leader的，然后由leader来搜集交易，并打包，然后广播出去。然后各个节点开始对新Block进行校验、投票、累积commit数量，最后落地。

而我这里对pbft做了修改，这是一个联盟，各个节点是平等的，而且性能要高。所以我不想让每个节点都生成一个指令后，发给其他节点，再大家选举出一个节点来搜集网络上的指令组合再生成Block，太复杂了，而且又存在了leader节点的故障隐患。

我对pbft的修改是，不需要选择leader，任何节点都可以构建Block，然后全网广播。其他节点收到该Block请求时即进入Pre-Prepare状态，校验格式、hash、签名、和table的权限，校验通过后，进入Prepare状态，并全网广播状态。待自己累积的各节点Prepare的数量大于2f+1时，进入commit状态，并全网广播该状态。待自己累积的各节点Commit的数量大于2f+1时，认为已达成共识，将Block加入区块链中，然后执行Block中sql语句。

很明显，和有leader时相比，缺少了顺序的概念。有leader时能保证Block的顺序，当有并发生成Block的需求时，leader能按照顺序进行广播。譬如大家都已经到number=5的区块了，然后需要再生成2个，有leader时，则会按照6、7的顺序来生成。而没有leader时，则可能发生多节点同时生成6的情况。为了避免分叉，我做了一些处理，具体的可以在代码里看实现逻辑。

### 区块信息查询

各节点通过执行相同的sql来实现一个同步的sqlite数据库（或mysql等其他关系型数据库），将来对数据的查询都是直接查询sqlite，性能高于传统的区块链项目。

由于各个节点都能生成Block，在高并发下会出现区块不一致的情况。如果因为某些原因导致链分叉了，也提供了回滚机制，sql可以回滚。原理也很简单，你ADD一个数据时，我会在区块里同时记录两个指令，一个是ADD，一个是回滚用的DELETE。同理，UPDATE时也会保存原来的旧数据。区块里的sql落地，譬如顺序执行1-10个指令，回滚时就是从10-1执行回滚指令。

每个节点都会记录自己已经同步了的区块的值，以便随时进行sql落地入库。

对区块链信息的查询，那就简单了，直接做数据库查询即可。相比于比特币需要检索整个区块链的索引树，速度和方便性就大不同了。

### 简单使用说明

使用方法：先下载[md_blockchain_manager项目](https://gitee.com/tianyalei/md_blockchain_manager)，然后导入工程里的sql数据库文件，修改application.yml数据库配置，最后启动manager项目。

然后修改md_blockchain中application.yml里的name、appid和manager项目数据库里的某个值对应，作为一个节点。如果有多个节点，则某个节点都和数据库里对应，填写各节点的ip。managerUrl就是manager项目的url，让该项目能访问到manager项目。

在md_blockchian项目启动时，在ClientStarter类中可见，启动时会从manager项目拉取所有节点的数据，并进行连接。如果自己的ip和appId等不在manager数据库中，则无法启动。

可以通过访问localhost:8080/block?content=1来生成一个区块。正常使用时至少要启动4个节点才行，否则无法达成共识，PBFT要求2f+1个节点同意才能生成Block。为了方便测试，可以直接修改pbftSize的返回值为0，这样就能自己一个节点玩起来了。如果有多个节点，在生成Block后就会发现别的节点也会自动同步自己新生成的Block。目前代码里默认设置了一张表message，里面也只有一个字段content，相当于一个简单的区块链记事本。当有4个节点时，可以通过并发访问其中的几个来同时生成Block进行测试，看是否会分叉。还可以关停其中的一个，看其他的三个是否能达成共识（拜占庭最多容许f个节点故障，4个节点允许1个故障），恢复故障的那个，看是否能够同步其他正常节点的Block。可以进行各种测试，欢迎提bug。

可以通过localhost:8080/block/sqlite来查看sqlite里存的数据，就是根据Block里的sql语句执行后的结果。

我把项目部署到docker里了，共启动4个节点，如图：
![输入图片说明](https://gitee.com/uploads/images/2018/0404/105151_c8931604_303698.png "1.png")

manager就是md_blockchain_manager项目，主要功能就是提供联盟链内各节点ip和各节点的权限信息
![输入图片说明](https://gitee.com/uploads/images/2018/0426/185644_23b10899_303698.png "2.png")

四个节点ip都写死了，都启动后，它们会相互全部连接起来，并维持住长连接和心跳包，相互交换最新的Block信息。
![输入图片说明](https://gitee.com/uploads/images/2018/0426/190528_3f93792e_303698.png "2.png")


我调用一下block项目的生成区块接口，http://ip:port/block?content=1，可以看到各节点投票及pbft的各状态
![输入图片说明](https://gitee.com/uploads/images/2018/0426/185819_06f95c59_303698.png "2.png")

别的节点会是这样，收到block项目请求生成区块的请求、并开始校验，然后进入pbft的投票状态
![输入图片说明](https://gitee.com/uploads/images/2018/0426/190006_0bb1f8d4_303698.png "2.png")

如果某节点断线了、或者是新加入的节点，从别的正常节点拉取新区块
![输入图片说明](https://gitee.com/uploads/images/2018/0426/190201_bdfd8d19_303698.png "2.png")

此外还有高并发情况下，各节点同时生成Block，系统处理共识、保证区块链不分叉的一些测试。

这个生成区块的接口是写好用来测试的，正常走的流程是调用instuction接口，先生产符合自己需求的指令，然后组合多个指令，调用BlockController里的生成区块接口。


