#1.Overview

Hyperledger Fabric v1.0提供基本的API，使用gRPC上的协议缓冲区，用于应用程序与blockchain网络交互。 API涵盖事务处理，安全成员服务，区块链遍历和事件处理。有许多语言支持协议缓冲区，包括Go，C＃，Java，Javascript，Python和C ++。因此，期望提供用于应用开发者使用的本地软件开发工具（SDK）。

本文档指定SDK实施应提供的最小API集。 SDK的目标是提供这两种原语来访问区块链网络中的面向客户端的特征，并且在合理的抽象层次之上，使得开发人员的生活更容易，以应用程序写入的本地语言发展努力。但是，使用SDK不会阻止应用程序直接调用gRPC。

请注意，当前的REST API已被弃用，SDK不应使用REST API构建。使用gRPC通过REST的原因是1）能够用全双工流控制流（大多数API调用是异步的），以及2）更好的性能。此外，对等体不需要为HTTP / HTTPS打开更多的端口，因为在内部我们也使用gRPC进行组件间通信。
一个例外是新的成员服务实现（COP）暴露了一个REST API。

作为参考，Fabric v1.0架构和交互模型可以在本工作项中的附加文档中找到：https://jira.hyperledger.org/browse/FAB-37

***
#2.Goals

1. 应用程序开发
Fabric SDK应该使开发人员能够编写能够以各种方式与网络交互的应用程序。 应用程序可以调用/查询链码，监听网络生成的事件，检索有关存储在分类帐中的块和事务的信息等。
2. 链码开发（不是本规范v1.0的重点）
Fabric SDK应该使开发人员能够编写和单元测试chaincode。 开发人员应该能够快速测试链码而不需要将它们部署到网络中。

***
#3.原理

1. 记录良好的api，数据模型和示例代码
SDK应提供有关可用apis，数据模型和示例说明如何使用apis的清楚的书面文档。
2. 使用方便
Chaincode开发人员和应用程序开发人员关心编写业务逻辑。即使开发人员熟悉Fabric项目的内部结构是有利的，它不应该是一个先决条件。因此，SDK不应该对Fabric项目有任何编译时依赖性（除了定义各种合同的proto文件）。 SDK包/ jars /库应该通过知名的仓库提供，以便开发人员可以轻松地安装它们，并立即开始编写chaincode开发应用程序。
3. 性能
SDK必须仔细设计，以促进高吞吐量，水平可扩展性和低延迟。它应该理想地实现为无状态组件，或者允许通过数据库在应用程序实例之间共享状态。
4. 版本控制
建议由不同语言实现的SDK遵循相同的版本化，并且相同版本化的各种实现应当保持彼此的功能性的兼容性。
例如，fabric-sdk-node 1.0和fabric-sdk-py 1.0应支持Fabric 1.0版本，并具有相同的公开功能，如成员资​​格操作和事务处理。
5. 适用性
SDK应该便于插入可维护性支持，即日志记录。它应该允许消费应用程序设置记录器的实例。这是非常有用的，因为应用程序可能想为代码的所有部分（在SDK内部和外部）使用一个公共日志记录器。并且通常IT组织将具有用于监视和分析目的的日志追踪设置，使得期望“标准”日志格式。 SDK应该具有内置的记录器，以便开发人员在默认情况下获得日志记录。但它必须允许使用一组标准的日志API设置外部记录器。

***
#4.方案
有很多可能的情况，但让我们集中在几个例子说明SDK的整体功能。
####会员注册和注册
为了与Fabric交互，应用程序必须具有适当的身份，它被编码为我们称为ECert（注册证书）的成员身份证书的一部分。该身份可以由外部CA（证书颁发机构）或结构成员服务（Fabric-COP）提供，只要在处理期间由Fabric组件验证ECert以建立信任链。

如果使用标准CA来管理ECert的分发，则将使用ECert签署事务。这使得每个事务可识别为客户端身份。

另一方面，为结构设计的成员服务提供商（MSP）使用从ECert的密钥对导出密钥对的加密算法，从而为每个事务（也称为TCert）产生假名证书。 TCert不表示身份，但可以通过成员服务追溯到原始ECert。这种技术对于在许多业务场景中隐藏事务标识很有用。

Fabric提供了MSP接口的实现，称为“COP”（名称的灵感来自于类似警察的功能，而不是首字母缩略词）。它在github.com的hyperledger组织下的“fabric-cop”存储库中。

如果将基于结构的网络配置为使用外部CA，则用户注册表也将在外部维护。外部用户注册表将负责认证用户。经认证的用户然后可以请求与CA的注册，以便获得ECert。

就SDK而言，应该有两种类型的API：任何CA（内置或外部）的通用API，以及Fabric的内置成员服务专用的API。公共API负责允许消费应用执行与用户身份相关的最基本的操作，即“enroll（）”，这是获得已认证用户的ECert的过程;以及“getTCerts（）”，交易证书以允许用户提交交易。

可选成员服务实现（aka COP）的客户机的设计在单独的文档中描述。
####链结构
Hyperledger Fabric通过通道和分类帐设计支持隐私和机密性，统称为“链”。通信（线路上的数据）被限制在参与者之间的信道内，而块（静止数据）被保存在仅在那些参与者之间分发的私人分类帐上。网络上不是链的一部分的对等端不知道通道和私有分类帐中的任何内容，也不能访问它们。

一旦构建了链，应用程序就可以以私有方式向链上的对等体发送事务，并将已验证的事务提交给专用分类帐。

创建链的责任在于应用程序。通过SDK，应用程序启动与组织组（表示网络成员）和订购者服务的链的形成。

在SDK中，渠道及其关联的分类帐的组合由Chain类表示。应用程序与指定的订购者一起工作以首先准备新的频道并获得包含关于新链的关键信息的发生块，包括参与者信息（URL和证书），订购者信息（URL和证书）。应用然后通过以配置系统链码为目标的配置事务来协调参与的对等体对新通道的邀请。
![Alt text](1482912736936.png)
####事务支持
事务涉及2个显式步骤：赞同和提交。支持是请求对等体运行和签署（签署）事务的结果;承诺是要求对交易及其背书的有效性达成共识。

下图说明了客户端SDK与事务上的对等体之间的交互。有关更多详细信息，请参阅架构文档。

首先，SDK使用应用程序，使用应用程序（或经过身份验证的用户的）私钥（所有外部消息都需要签名认证）签署消息。然后根据认证策略（通过验证系统链代码或VSCC实现）将消息发送给1个或多个对等体。它异步获取响应，并决定是否继续创建事务并将其提交到一致性服务。继续交易提交的决定是基于基于带外知识的支持策略谓词（例如2中的2）。一旦提交，事务的处理是异步的，因此SDK侦听提交事件以通知应用程序完成或拒绝事务处理。
![Alt text](1482912809629.jpeg)
这是非常高级别的事务流描述。 详细涉及SDK上的网络和环境配置，获得签名密钥的安全成员资格管理，处理事务和事件流以及（根据应用）多个共识信道。
***
#5.客户端 - 服务器API参考
以下链接指向与Fabric（Peer，订购者和成员服务）的grpc通信的消息和服务定义：

提议（部署或事务调用的提议）
ProposalResponse（通用建议响应）

Chaincode（chaincode spec，invocation spec，deployment spec）
ChaincodeProposal（链码特定的建议标题和有效载荷）

事务（事务调用）
ChaincodeTransaction（链码特定事务动作有效载荷）

（新的COP API是基于json的，需要添加新的成员服务设计）

消息定义应该是设计SDK API的灵感来源。 API显然不必忠实地反映消息定义，因为SDK可以使用智能默认值和状态信息来最小化所需参数的数量。
***
#6.规格
在这里我们讨论设计原则和结构考虑。
总之，我们有几个不同级别的模块（更小的数字意味着更高的级别）：
<table>
<tr><td colspan ="3"> 软件包：Hyperledger Fabric客户端 </td></tr>
<tr><td>模块</td><td>Level</td><td>功能</td></tr>
<tr><td>客户端</td><td>0</td><td>它必须允许用户构造任何所需的对象以直接对网络执行所有支持的操作，例如chaincode部署，事务调用，各种查询。此外，每个语言实现还可以基于编码风格和流行的社区实践来决定是否添加方便的方法，如sendTransaction（chain，tx）;</td></tr>
<tr><td>链</td><td>1</td><td>链表示为了启动可以独立处理事务的一致性信道而特别形成的对等体的网络。网络可以具有1个或多个链。 链中的对等体维护包含在链上传递的事务的单独分类帐，包括关于成员资格的任何配置。所有事务都在链上发送，应用程序可以在多个链上操作。</td></tr>
<tr><td>Peer</td><td>2</td><td>表示网络上的计算节点。 Peer有署名者和/或提交者的角色，维护分类账。应用可以连接到它可以访问的多个对等体。 </td></tr>
<tr><td>Orderer</td><td>2</td><td>类似于“Peer”，但是表示排序服务的端点，其可以是单个节点（在开发期间的本地设置）或者到订户网络的代理节点。基于结构的区块链网络将具有由一组定序器节点组成的单个排序服务。应用可以选择信任订户中的特定一个，或一组订购者，或者设置代理以将交易广播传播给订购者。</td></tr>
<tr><td>用户</td><td>2</td><td>表示将在网络上进行处理的用户。此类的实例可以基于注册证书来实例化。证书可以从成员服务或从外部CA获得。

理论上，这个类也可以表示网络中的对等节点成员资格。但是，这不是一个应用程序的问题（它更多的是一个网络管理问题），所以在这个设计中没有暴露。</td></tr>
<tr><td>Proposal</td><td>3</td><td>注册的用户可以向对等体列表发出交易建议，以支持交易。一旦接收到背书响应，应用可以确定是否已经获得足够的背书签名以及它是否应该继续向共识服务提交交易。

这是一个围绕提案的原始GRPC消息的包装类，提供了方便的方法来构造它。</td></tr>
<tr><td>ProposalResponse </td><td>3</td><td>从提案调用到批注者的响应，包含背书结果（yay或nay），签名等。

这是一个关于ProposalResponse的原始GRPC消息的包装类，提供了访问其内容（签注，签名等）的便利方法。</td></tr>
<tr><td>事务</td><td>3</td><td>注册的用户可以在收集了签署后提交事务。事务请求包含签署签名和MVCC +后映像，并且定向订购服务API。事务可以有两种类型：部署和调用。
这是一个关于Transaction的原始GRPC消息的包装类，提供了方便的方法来构造它。</td></tr>
<tr><td>CryptoSuite</td><td>3</td><td>密码套件封装了用于数字签名和使用非对称密钥对的加密算法，使用对称密钥的消息加密以及安全散列和MAC。</td></tr>
<tr><td colspan ="3"> 软件包：会员服务</td></tr>
<tr><td>模块</td><td>Level</td><td>功能</td></tr>
<tr><td>MemberService </td><td>0</td><td>这是结构的可选组件的客户端，成员服务。此类的主要目的是从成员服务获取用户注册证书。
此外，这个类本身或其扩展也应该提供在Fabric的默认成员服务实现中可用的附加功能，即用户注册。
</td></tr>
</table>
为了帮助说明上述模型类之间的关系，创建了以下UML图：
![Alt text](1482914233911.png)
###6.1. 客户

与最终用户的主要交互处理程序。客户端实例提供处理程序以与对等体，订户和可选的成员服务的网络交互。使用SDK的应用程序可能需要与多个网络交互，每个网络通过客户端的单独实例。

每个客户端在初始创建时都应使用来自一致性服务的配置数据进行初始化，该配置数据包括受信任根列表，订户证书和IP地址以及它可以访问的对等证书和IP地址的列表。作为引导应用程序环境的一部分，这必须在带外完成。它也是应用程序的责任，维护客户端的配置，因为SDK不持久化此对象。

每个客户端实例可以维护表示通道和相关子分类帐的多个链。
####new_chain
使用给定的名称初始化链实例。这实际上代表了“Channel”（如上所述），并且该调用返回一个空对象。为了初始化频道，必须首先在返回的对象上配置参与批准者和订阅者对等体的列表。

* 参数
name（str）：链的名称，建议使用命名空间避免碰撞
* 返回
（链实例）：未初始化的链实例。

#### get_chain
从状态存储获取链实例。这允许保存现有链实例以供稍后检索，并在应用程序的实例之间共享。请注意，应用程序/ SDK负责记录链信息。如果应用程序不能从存储器中查找链信息，则它可以调用另一个API来查询一个或多个对等体的该信息。
* 参数
name（str）：链的名称
* 返回
（链实例或无）：链实例的名称。
 错误：
状态存储尚未设置
在该名称下不存在链

#### query_chain_info
这是到指定的对等体的网络调用以发现链信息。目标对等体必须是所讨论链的一部分，以便能够返回所请求的信息。

* 参数
 name（str）：链的名称
peer（对等实例的数组）：target要查询的Peer
* 返回
（链实例或无）：链实例的名称。
错误：
目标对等方不知道关于链的任何东西


#### set_state_store
SDK应该有一个内置的键值存储实现（建议基于文件的实现，以便在开发过程中轻松设置）。但生产系统需要一个由数据库支持的存储，以实现更强大的存储和集群，以便多个应用程序实例可以通过数据库共享应用程序状态（注意，这不一定使应用程序有状态）。此API使得此可插入，以便应用程序可以选择不同的存储实现。

* 参数
store（KeyValueStore）：消费应用程序提供的替代KeyValueStore实现的实例。
* 返回
没有

#### get_state_store
一种获取用于此客户端的状态存储对象的方便方法。

* 参数
没有
* 返回
（KeyValueStore实例）：在此客户端内设置的KeyValueStore实现对象，如果不存在，则为null

####set_crypto_suite
设置CryptoSuite接口实现的实例。密码套件封装了数字签名和使用非对称密钥对的加密算法，使用对称密钥的消息加密以及安全散列和MAC。

* 参数
套件（对象）：一个加密套件实现的实例

####get_crypto_suite
一种方便的方法，用于获取用于此客户端的CryptoSuite对象。

* 参数
没有
* 返回
（CryptoSuite实例）：此客户端中设置的CryptoSuite实现对象，如果不存在，则为null

####set_user_context
将User类的实例设置为此客户端实例的安全上下文。此用户的凭证（ECert）将用于与区块链网络进行事务和查询。在设置用户上下文时，如果在客户端实例上设置了“状态存储”，则SDK将该对象保存在持久性高速缓存中。如果没有设置状态存储，则不会建立此缓存，并且应用程序负责在应用程序崩溃并恢复时再次设置用户上下文。

* 参数
user（User）：封装了经过验证的用户签名资料（私钥和注册证书）的User类的实例，

####get_user_context
如上所述，客户端实例可以具有可选的状态存储。 SDK将已注册的用户保存在可由应用程序的授权用户访问的存储中（认证由SDK外部的应用程序完成）。此函数尝试通过名称从本地存储（通过KeyValueStore接口）加载用户。加载的用户对象必须表示具有由受信任的CA（例如COP服务器）签名的有效注册证书的已注册用户。

* 参数
name（str）：的名称
* 返回
（用户实例）：与名称对应的用户对象，如果用户不存在或者状态存储尚未设置，则为null

###6.2.链（chain）

“链”对象捕获通道的设置，该通道由订户创建以隔离到参与通道的对等体的事务传递。链必须在配置了对等端和订户列表之后进行初始化。初始化向订户发送CONFIGURATION事务以创建指定的通道，并要求对等体加入该通道。
####add_peer
将对等端点添加到链对象，这是仅本地操作

* 参数
peer（Peer）：已使用URL，TLC证书和注册证书初始化的Peer类的实例

####remove_peer
从链对象中删除对等端点，这是一个仅本地操作

* 参数
peer（Peer）：对等类的实例

####get_peers
从本地信息中获取链的同行。

* 参数
	没有
* 返回
（对等列表）：链上的对等列表

####add_orderer
将orderer端点添加到链对象，这是一个仅限本地操作。链实例可以选择使用单个订购者节点，其将向订购者网络的其余部分广播请求。或者如果应用程序不信任订阅者节点，它可以通过将它们添加到链实例中来选择使用多个。并且所有关于订货方的API将同时向所有订货方广播。

* 参数
orderer（Orderer）：Orderer类的实例

####remove_orderer
从链对象中删除orderer端点，这是一个仅限本地操作。

* 参数
orderer（Orderer）：Orderer类的实例

####get_orderers
获取链的订单。这是仅本地操作。

* 参数
	没有
* 返回
（订货人列表）：链上的订货人列表

####initialize_chain
调用订阅者开始构建新链，这是打开新消息流和连接参与对等的列表的组合。这是一个长期运行的过程。只有一个应用程序实例需要调用此方法。一旦链成功创建，其他应用程序实例只需要调用get_chain（）来获取有关此链的信息。

* 参数
	没有
* 返回
（bool）：链初始化过程是否成功

####update_chain
调用订阅者更新现有链。这允许向现有链添加和删除对等节点，以及在证书续订时更新对等证书信息。

* 参数
	没有
* 返回
（bool）：链更新过程是否成功


####is_readonly
获取链状态以查看底层渠道是否已终止，使其成为只读链，可以查询信息（事务和状态），但不能提交新的事务。

* 参数
	没有
* 返回
（bool）：是ready-only（true）或不是

####query_info
查询关于链状态的各种有用信息（高度，已知对等体）

* 参数
	没有
* 返回
（ChainInfo）with height，当前唯一有用的信息

####query_block
按块编号查询分类帐

* 参数
blockNumber（number）
* 返回
包含块的对象

####query_transaction
按号码查询交易分类帐

* 参数
transactionID
* 返回
包含事务的TransactionInfo

####create_deploy_proposal
创建交易提案。这涉及将提议与数据（chaincodeID，chaincode调用规范等）组合并使用与要签名的ECert相对应的私钥来对其进行签名。

* 参数
1. chaincodePath（string）：要部署的chaincode的路径
2. chaincodeName（object）：用于标识链上的chaincode的自定义名称
3. fcn（string）：部署后启动状态时要调用的chaincode函数的名称
4. args（string []）：用于调用由“fcn”指定的init函数的* 参数
sign（Bool）：是否对事务签名，默认为True
* 返回
（提案）：创建的投标实例或无。

####create_transaction_proposal
创建交易提案。这涉及将提案与数据（chaincodeName，要调用的函数，参数等）组合并使用与ECert对应的私钥来签名。

* 参数
1. chaincodeName（string）：给予要调用的目标chaincode的名称
2. args（string []）：用于在chaincode上调用“invoke”方法的* 参数
3. Sign（Bool）：是否对事务签名，默认为True
* 返回
（Transaction_Proposal instance）：创建的Transaction_Proposal实例或None。

####send_transaction_proposal
将创建的提案发送给同行以进行背书。

* 参数
1. transaction_proposal（Transaction_Proposal）：事务建议数据
2. chain：将向其发送提议的对等体的目标链
3. retry（Number）：失败时重试的次数，默认为0
* 返回
（Transaction_Proposal_Response响应）：发送提议请求的响应。

####create_transaction
按照背书策略创建具有投标回应的交易。

* 参数
proposal_responses（[Transaction_Proposal_Response]）：在提案调用中接收的提案响应数组。
* 返回
（事务实例）：创建的事务对象实例。

####send_transaction
将事务发送到链的订阅服务（一个或多个订阅者端点）以达成共识并提交到分类帐。

此调用是异步的，并且通过BLOCK或CHAINCODE事件通知成功的事务提交。此方法必须为应用程序附加事件侦听器以处理“事务提交”，“事务完成”和“错误”事件提供一种机制。

请注意，在封底下，有两种不同类型的通信与结构后端触发不同的事件发送回应用程序的处理程序：
1. 具有订阅服务的grpc客户端以“广播”调用的请求/响应方式使用“常规”无状态HTTP连接。当在响应中接收到成功的确认时，方法实现应该发出“事务提交”，或者当接收到错误时，发出“错误”
 2. 方法实现还应当与链的事件源对等体作为内部事件中枢机制的一部分保持持久连接，以便支持结构事件“BLOCK”，“CHAINCODE”和“TRANSACTION”。这些事件应导致该方法向应用程序发出“完成”或“错误”事件。

* 参数
transaction（Transaction）：上面构造的事务对象
* 返回
result（EventEmitter）：允许应用程序在“提交”，“完成”和“错误”上附加事件处理程序的句柄。

###6.3.用户


User类表示已注册并由注册证书（ECert）和签名密钥表示的用户。 ECert必须已由区块链网络已配置为信任的CA之一签名。注册用户（具有签名密钥和ECert）可以与链进行链码部署，事务和查询。

可以预先从CA获得用户ECerts作为部署应用程序的一部分，或者可以通过其注册过程从可选的Fabric COP服务获得。

有时用户身份与对等身份混淆。用户身份代表签名能力，因为它可以访问私钥，而应用/ SDK上下文中的对等身份仅具有用于验证签名的证书。应用程序无法使用对等身份来签名，因为应用程序无权访问对等身份的私钥。
####get_name
获取成员名称。实例对象的必需属性。

* 返回（str）：
用户的名称

####get_roles
获取用户的角色。它是“客户端”和“审计器”中的可能值的数组。成员服务定义了为对等成员保留的另外两个角色：“对等体”和“验证者”，它们不暴露给应用程序。

* 返回（str []）：
此用户的角色

####get_enrollment_certificate
 返回表示此用户身份的底层ECert。

* 参数：无
* 返回：
由受信任的CA签名的PEM格式的证书

####set_name
设置用户的名称/ id。

* 参数：
name（string [）：用户名/ id。

####set_roles
设置用户的角色。请参见上面的合法值。

* 参数：
角色（string []）：用户的角色列表

####set_enrollment_certificate
设置用户的注册证书。

* 参数：
证书：由受信任的CA签名的PEM格式的证书

####generate_tcerts
获取一批用于交易的TCerts。 TCert和事务之间有1对1的关系。 TCert可以由SDK使用用户的加密材料在本地生成。

* 参数
1. count（number）：批量中获取多少？
2. 属性（string []）：要包括在TCert中的属性列表
* 返回（TCert []）：
一个TCerts数组

###6.4.Peer

Peer类表示远程对等节点及其网络成员资格，也称为用于验证签名的ECert。对等成员表示组织，不同于表示个人的用户成员资格。

构建时，Peer实例可以指定为事件源，在这种情况下，应配置“eventSourceUrl”属性。这允许SDK自动将事务侦听器附加到事件流。

应当注意，对等事件流在对等级而不是在链和链码级处起作用。

####connectEventSource
由于实际上所有的对等体都是事件生成器，因此在构建对等实例时，应用程序可以将其指定为应用程序的事件源。通常只有一个链上的对等体之一需要是事件源，因为链上的所有对等体产生相同的事件。此方法告知SDK哪个对等端用作客户端应用程序的事件源。 SDK负责管理对等体的EventHub的连接生命周期。客户端应用程序的责任是理解并通知所选对等关于它想要接收哪些事件类型以及要使用的回叫功能。

* 参数：
	没有
* 结果：
Promise / Future：这为应用程序提供了附加“success”和“error”侦听器的句柄

####is_event_listened
发现是否至少有一个侦听器已连接到给定事件的目标对等端的网络调用。这有助于应用程序实例决定是否需要在崩溃恢复或多实例部署中连接到事件源。

注意：这需要对等事件生成器的增强。

* 参数：
1. eventName（string）：必需
2. chain（Chain）：可选
* 结果：
（boolean）：所述事件是否已被该链上的某个应用程序实例侦听

####addListener
对于连接到eventSource的对等体，addListener为一组事件类型注册EventCallBack。 addListener可以被多次调用，以支持不同的EventCallBack函数接收不同类型的事件。

请注意，下列参数在某些语言（如Java）中是可选的，它们构造侦听器接口的实例，并将该实例作为参数传递。

* 参数：
1. eventType：ie。块(Block)，Chaincode，事务
2. eventTypeData：Object特定于“Chaincode”事件类型当前需要的事件类型，指定与在目标对等体上执行的链码中设置的事件名称匹配的模式，以及“事务”事件类型，指定事务ID
3. eventCallback：客户端应用程序类注册回调。
* 返回：
 [event-listener-ref]对事件侦听器的引用，某些语言使用ID（javascript），其他语言使用对象引用（Java）

####removeListener
取消注册侦听器。

* 参数：
 [event-listener-ref]：由SDK为事件侦听器返回的引用
* 返回：
statusFlag：成功/失败

####get_name
获取对等体名称。实例对象的必需属性。

* 返回（str）：
 对等体的名称

####set_name
设置对等体名称/ id。

* 参数：
名称（字符串）：此对等体的唯一名称/ ID。

####get_roles
获取用户参与的用户角色。它是“客户端”和“审核员”中可能值的数组。成员服务定义了为对等成员保留的另外两个角色：“对等体”和“验证者”，它们不暴露给应用程序。
* 返回（str []）：
此用户的角色

####set_roles
设置用户参与的用户角色。有关合法值，请参见上文。

* 参数：
角色（string []）：用户的角色列表

####get_enrollment_certificate
 返回表示此用户身份的底层ECert。

* 参数：无
* 返回：
由受信任的CA签名的PEM格式的证书

####set_enrollment_certificate
设置对等体的注册证书。

* 参数：
证书：由受信任的CA签名的PEM格式的证书

###6.5.KeyValueStore（interface）
区块链应用程序需要保存状态，包括用户注册材料（私钥，CA签名的证书）。这些状态需要坚持。 “KeyValueStore”接口为SDK提供了一种简单的机制，可以代表应用程序自动保存状态。
如果应用程序使用CryptoSuite的基于软件的密钥生成实现，它需要一个密钥值存储。如果尚未由应用程序设置，SDK应默认为内置实现，例如基于本地文件系统的实现。
SDK还可以将键值存储中的用户注册材料保存为可选缓存。但是如果应用程序没有配置一个，SDK应该解释为应用程序选择始终为会话设置用户上下文，并且不尝试使用默认键值存储。
####get_value
检索给定键的值
* 参数
key（str）：键的名称
* 返回
结果（对象）：值

####set_value
设置值
* 参数
1. 键
2. 值
* 返回
确认值的成功存储

###6.6. CryptoSuite（接口）

密码套件封装了数字签名和使用非对称密钥对的加密算法，使用对称密钥的消息加密以及安全散列和MAC。这是由fabric crypto团队发布的BCCSP（区块链加密服务提供程序）接口的镜像设计。

目前计划为同行和COP的默认实现，并且必须通过SDK的默认实现也实现：
ECDSA：曲线“secp256r1”和“secp384r1”
AES：AES128，AES256，CBC / CTR / GCM模式密钥长度= 128位
SHA：SHA256，SHA384，SHA3_256，SHA3_384

####generate_key
基于选项生成密钥。输出可以是非对称算法中的私钥或公钥，或对称算法的秘密密钥。

* 参数
opts（Object）：封装两个属性“algorithm”和“ephemeral”的对象。
* 返回
结果（键）：键对象

####derivKey
使用opts从k导出密钥。

* 参数
1. k（键）
2. opts（Object）
* 返回
（键）导出键

####importKey
使用opts从其原始表示中导入密钥。

* 参数
1. k（键）
2. opts（Object）
* 返回
（Key）包含原始关键字节的Key类的实例

####getKey
* 返回此CSP与主题键标识符ski相关联的键。

* 参数
ski（byte []）
* 返回
（Key）与滑雪相对应的Key类的实例

####hash
使用选项隐藏消息msg。

* 参数
1. msg（byte []）
2. opts（Object）使用哈希算法（如“SHA2”或“SHA3”）的值封装属性“algorithm”的对象。
* 返回
（Key）与滑雪相对应的Key类的实例

####encrypt
加密纯文本。

* 参数
1. key（Key）公共加密密钥
2. plainText（byte []）
3. opts（Object）
* 返回
（byte []）密文

* decrypt
解密密文。

* 参数
1. 密钥（密钥）私有解密密钥
2. cipherText（byte []）
3. opts（Object）
* 返回
（byte []）纯文本

* sign
签名数据。
* 参数
1. 密钥（密钥）私有签名密钥
2. 要签名的目标消息的digest（byte []）固定长度摘要
3. opts（函数）散列函数要使用
* 返回
结果（对象）：签名对象

####verify
验证签名。

* 参数
1. 密钥（密钥）公共验证密钥
2. 签名（byte []）签名
3. digest（byte []）签名的原始摘要
* 返回
（bool）：验证成功与否

###7.处理网络错误
客户端SDK以两种不同的方式与fabric进行通信：无状态HTTP连接和持久HTTP连接。

“send_proposal”和“send_transaction”调用是以请求/响应方式的无状态调用。发生网络错误时，呼叫将超时。 SDK应该具有可配置的超时值，以便应用程序基于对客户端应用程序和结构节点之间的网络特性的理解来控制这些调用的行为。

此外，这些方法可能具有“重试”值，以允许SDK在超时错误时自动尝试重新尝试HTTP调用。在尝试重试并仍然发生超时后，该方法应该返回一个错误。

另一方面，SDK和事件源对等体之间的事件流连接是持久连接。具体来说，在由Fabric定义的事件流传输接口的情况下，连接是双向的，以允许消息在两个方向上发送。在网络故障的情况下，此连接断开，导致丢失事件和无法触发客户端应用程序注册的侦听器。

SDK应尝试代表应用程序重新建立连接。但是如果在“重试”尝试次数之后连接不能恢复，则它应该以合理的高严重性错误通知应用这种情况。

###8.	参考资料

1.	Next-Consensus-Architecture_Proposal: https://github.com/hyperledger/fabric/blob/master/proposals/r1/Next-Consensus-Architecture-Proposal.md

2.	Consensus endorsing, consenting, and committing model: https://jira.hyperledger.org/browse/FAB-37

3.	Node.js SDK
	https://github.com/hyperledger/fabric-sdk-node

    4) Fabric-Cop Design
        https://docs.google.com/document/d/1TRYHcaT8yMn8MZlDtreqzkDcXx0WI50AV2JpAcvAM5w

5) Next Hyperledger-Fabric Architecture Protocol messages
https://docs.google.com/document/d/1qD2zOTxmPoss3Xop_pDdkSXHozIR4N27NEsymosW4Gk
