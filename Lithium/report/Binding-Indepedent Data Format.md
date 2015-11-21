Binding-Independent Data Format
===============================

#概述

在SAL中数据以一种树形结构建模，类似XML

QName - 节点类型标识，与XML QName类似
DOMNode - 节点树的基类，定义节点基本参数，例如QName，存在两种DOMNode类型：
  SimpleNode - 仅包含简单值的节点，如integer
  CompositeNode - 包含其他节点的节点
NodeModification - 应用到节点的修改的基类，存在两种NodeModification
  SimpleNodeModification - 表示SimpleNode变化，例如修改其中的值或节点删除
  CompositeNodeModification - 表示CompositeNode变化

#节点(Nodes)
Each node must have a QName. Multiple nodes with the same QName could exist in the same container node.
每个节点必须具有QName。具有相同QName的多个节点可以存在相同的容器节点

Simple Node
The Simple Node represents a leaf in the data tree that does not contain any nested nodes, but the value of node. In terms of XML The Simple Node is an element that contains only text data (CDATA or PCDATA).
简单节点代码了数据树中的一个叶子，除了具体值，它不含子节点。在XML语法中Simple Node是只包含了文本数据（CDATA或PCDATA）的元素
The Simple Node is a manifestation of the following YANG data schema constructs:
Simple Node代表了如下YANG数据：
leaf – simple node could represent YANG leafs of all types, except for the empty type, which in XML form is similar to an empty container.
leaf - Simple Node可以代表所有类型叶子，除了empty类型
item in a leaf-list
leaf-list中的条目

Composite Node
The Composite Node represents a branch in the data tree. It can contain nested composite nodes or leaf nodes. In terms of XML, the Composite Node is an element which does not contain text data directly (CDATA or PCDATA), only other nodes. The Composite Node is the manifestation of the following YANG data schema constructs:
Composite Node代表数据树上的分支。可以包含子composite node或leaf node。在XML语法中，表示只包含其他节点的元素，而不包含直接包含文本数据，它是如下YANG数据的表现：
container – the Composite Node represents the YANG container and could contain all children schema nodes of that container
container - 表示YANG容器并可以包含所有子节点
item in the list – the composite node represents one item in the YANG list and could contain all children schema nodes of that list item
anyxml
leaf with empty type
带empty类型的叶子

节点修改(Node Modifications)
In use cases where state data changes, node modifications are modeled as a part of additional metadata in the data tree. Modification types are based on Netconf edit-config RPCs. In order to modify the configuration or state data tree a user must create a tree representing the modification of the data and apply the modification to the target tree.
表示数据改变，node modification作为数据树的附加元数据。Modification类型基于Netconf edit-config RPCs。用户创建树来代表数据的修改，并将修改应用到目标树上

Simple Node Modifications
The simple node supports the following modifications:
simple node支持下列修改:

replace - data identified by the node containing this modification replaces any related data in the target data tree; if no such data exists in the target data tree, it is created.
create - data identified by the node containing this modification is added to the target data tree if and only if the data does not already exist in target data store. If the data exists, an error is returned with an error value of "data-exists".
delete –data identified by the node containing this modification is deleted from the target data tree if and only if the data currently exists in the target data tree. If the data does not exist, an error is returned with an error value of "data-missing".
remove - The data identified by the node containing this attribute is deleted from the target data tree if the data currently exists in the target data tree. If the data does not exist, the modification is silently ignored.

Composite Node Modifications
The composite node supports the following modifications:

merge - data identified by the node containing this modification is merged with the data at the corresponding level in the data tree
replace - data identified by the node containing this modification replaces any related data in the target data tree; if no such data exists in the target data tree, it is created.
create - data identified by the node containing this modification is added to the target data tree if and only if the data does not already exist in target data store. If the data exists, an error is returned with an error value of "data-exists".
delete –data identified by the node containing this modification is deleted from the target data tree if and only if the data currently exists in the target data tree. If the data does not exist, an error is returned with an error value of "data-missing".
remove - data identified by the node containing this attribute is deleted from the target data tree if the data currently exists in the target data tree. If the data does not exist, the modification is silently ignored.

Binding-Independent Broker
The Binding Independent Broker is the communication hub between Providers and Consumers. It exposes the following functionality:
Binding Independent Broker是provider和consumer之间的通信中心，有如下功能

Provider and Consumer registration provider和consumer注册
Notification Hub 通知中心
RPC Routing RPC路由
System state access & modification 系统状态访问及修改
Communication between the Broker, Providers and Consumers is session-based, so that the Broker can uniquely identify registered Consumers and Providers and their respective functionalities / metadata. There are two session types:
Broker，Provider和Consumer之间的通信是基于会话，因此broker可以唯一标识consumer和provider，以及他们各自的功能与原数据，有两种会话类型：

ConsumerSession – uniquely identifies a Consumer's registration 唯一标识consumer注册
ProviderSession – uniquely identifies a Provider's registration 唯一标准provider注册
Provider and Consumer Registration
Providers and Consumers need to register with the Binding-Independent SAL layer before they can use it. Only registered Providers can expose their functionality to Consumers via the SAL layer.
Provider及Consumer在使用前需要向Binding-Independent SAL层注册，只有注册的provider能像consumer提供功能

Registration Contract
Registration of a Consumer
Session = registerConsumer(Consumer)
Where:

Session: ConsumerSession between the Broker and a Consumer, uniquely identifies the Consumer’s registration
Consumer: A consumer component, an object implementing the Consumer contract.
During registration the Broker gets the functionality from the Consumer. It uses the getFunctionality() method from the Consumer contract, to register that functionality into the system.
在注册过程中，Broker可以获取Consumer的功能，利用getFunctionality()方法获取功能，并注册至系统中

The Consumer is required to use the returned session for all communication with the Broker or any Broker service. The session is injected into the Consumer by invoking the method injectConsumerSession() of the Consumer contract.
Consumer需要使用注册返回的会话与Broker进行通信，会话通过Consumer中injectConsumerSession()方法注入到Consumer中

Provider Registration
Session = registerProvider(Provider)
Where:

Session: ProviderSession between the Broker and a Provider, uniquely identifies the provider registration
Provider: A consumer component, an object implementing the Provider contract.
At registration time the Broker uses the getFunctionality() method from the Provider's contract to register the Provider's functionality with the system.

The Provider is required to use the returned session to communicate with the Broker and/or to use the Broker's services. The session is injected into the Provider by invoking the method injectProviderSession of the Provider contract.

Notification Hub

The Binding Independent Broker exposes the implementation of the NotificationService contract to Providers and Consumers.
Binding Independent Broker暴露NotificationService实现

NotificationService Contract
Publishing a Notification 发布通告
A Provider publishes a notification by invoking the following method on the implementation of the NotificationService contract:
Provider通过调用下面方法来发布通告，该方法为实现NotificationService

Synopsis:

Success = notify(Session, DataNode Notification)     
Where:

Success: set to OK by the Binding-Independent Broker if the notification has been successfully created
Session: ProviderSession between BI Broker and a Provider
Notification: a data node in the BI structure representing contents of the notification.
Notification Listener Registration
The Broker provides functionality to register and unregister Notification Listeners by exposing and implementing the following methods:
Broker提供下列方法来注册或解除Notification Listener

Synopsis:

Success = addNotificationListener(Session, NotificationType, NotificationListener listener)
Where:

Success: set to OK by binding-independent Broker if the Notification listener has been successfully registered
Session: ConsumerSession between the BI Broker and a Consumer
NotificationType: a model independent identifier of the notification type.
NotificationListener: an object implementing the NotificationListener contract, which is called if the notification occurred.
Success = removeNotificationListener(Session, NotificationType, NotificationListener listener)
Where:

Success: set to OK by binding-independent Broker if the notification listener has been successfully registered
Session: ConsumerSession between BI Broker and consumer
NotificationType: an QName identifying the notification type.
NotificationListener: an object implementing NotificationListener contract, which is to be unregistered.
RPC Routing

The Binding Independent Broker exposes the RpcService contract to Consumers (applications and other providers) and to Providers.
Binding Independent Broker暴露RpcService

RpcService Contract
RPC Calls
Synopsis:

Result = rpc(Session, RpcIdentifier, Input) 
Where:

Result: An Object containing BI structure if the RPC was successful, otherwise reason of the error
Session: ConsumerSession between the BI Broker and a Consumer
RpcIdentifier: QName identifying the RPC type
Input: Composite Node in BI structure representing contents of the input to the RPC.
RPC Call functionality enables the invocation of functionality exposed by Providers.
RPC调用功能将调用由Provider实现的功能

Registration of RPC Implementation
Exposes an API to register for RPC processing and returning results.

Status = addRpcImplementation(Session, RpcIdentifier, RpcImplementation)
Where:

Status: OK if the RPC implementation was registered successfully.
Session: ProviderSession between the Broker and a Provider
RpcIdentifier: QName identifying the RPC type
RpcImplementation: An object implementing RPCImplementation contract, which is called if the RPC occurred.
Status = removeRpcImplementation(Session, RpcIdentifier, RpcImplementation)
Where:

Status: OK if the rpc implementation was unregistered registered successfully.
Session: ProviderSession between broker and provider
RpcIdentifier: QName identifying the RPC type
RpcImplementation: An object implementing RPCImplementation contract, which is to be unregistered.
RPC Validation Functionality
Exposes an API to register for RPC processing and returning results.

Status = addRpcValidator(Session, RpcIdentifier, RpcValidator)
Where:

Status: OK if the RPC was registered successfully.
Session: ProviderSession between the BI Broker and a Consumer
RpcIdentifier: Binding-independent type identifier of RPC
RpcValidator: An object implementing RpcValidator contract, which is called if the RPC occurred.
Status = removeRpcValidator(Session, RpcIdentifier, RpcValidator)
Where:

Status: OK if the RPC was registered successfully.
Session: ProviderSession between the BI Broker and a Consumer
RpcIdentifier: Binding-independent type identifier of RPC
RpcValidator: An object implementing RpcValidator contract, which is to be removed.

System State Access & Modification
The Binding Independent Broker provides uniform access to the system's state data. The BI Broker does not implement state functionality directly, but uses BI Data Repositories as operators on operational and configuration state.
Binding Indendent Broker提供对于系统状态数据的统一访问。BI Broker并未直接实现状态功能，但可以使用BI Data Repositories来操作operational和configuration状态

Consumers and Providers are responsible for pushing their state into the state repository.
Consumer和Provider负责将状态推送到state repository

The system state access & state modification is provided by implementing the DataBroker contract.
System state access & state modification是通过实现DataBroker契约来提供的

See Section BI Data Repository for more information.

Two Phase Commit of Data Change 数据变化的两步提交
Commit of state is blocking operation, which commits the changes described in candidate state to current state.
状态提交是阻塞性操作

In order to perform a successful commit and apply its changes to the current state data, all affected providers must successfully validate the commit.
为了提交成功并将变化应用到当前数据上，所有受影响的provider必须验证提交

The commit operation of state data follows the two-phase commit protocol:
数据提交遵循以下两部提交协议：

Initiator: Any Consumer or Provider, which invoked commit method from the DataBroker contract. 发起者：consumer或provider，调用commit方法
Coordinator: Data Repository instance implementing the DataStore contract in cooperation with Brokers 协调者：Data Repository实例
Cohorts: All affected Providers that registered their respective implementations of the DataCommitHandler contract. 群组：所有受影响的provider，即注册实现DataCommitHandler

The basic commit algorithm is as follows:
基本提交算法如下：

Commit Request Phase is invoked by a Consumer by issuing the commit operation.
提交请求步骤
The coordinator sends a commit-request request to all cohorts and waits for their replies.
协调者发送commit-request请求给所有群组并等待响应
The cohorts execute the transaction to the point where they will be asked to commit. Each one of the cohorts prepare an rollback scenario.
群组执行事务，群组每个成员均准备回滚
Each cohort replies with success reply or error message if the cohort experiences a failure that will prevent commit.
每个群组成员响应，如任何一个成员失败，则提交停止

Commit phase
提交步骤
Success - If the coordinator received a success reply from all cohorts:
The coordinator sends a finish-commit RPC to all cohorts.
Each cohort completes the operation
Each cohort sends a success reply to the coordinator.
The coordinator completes the transaction when all success replies were retrieved from cohorts and returns a success reply to the initiator.
Failure – If any cohort returned error during the commit request phase (or the transaction timed-out):
The coordinator sends a commit-rollback RPC to all cohorts.
Each cohort rolls back the transaction using its own pre-defined rollback strategy.
Each cohort sends an acknowledgement reply to the coordinator.
The coordinator rollbacks the transaction and returns an error to the initiator.
The error reported to the invoker contains list of all validation errors issued during the Commit Request phase.

The successful commit of configuration data DOES NOT imply that the configuration change is applied. It is the responsibility of each Provider to apply its respective configuration.
成功提交configuration数据并不意味着配置修改已经生效，provider有责任执行各自的配置

Broker as a Coordinator
Broker作为协调者
The Broker participates in two-phase commits as an aggregate commit handler. It implements the DataCommitHandler contract with the following functionality:
Broker实现DataCommitHandler如下功能：
Each invocation of a method from the DataCommitHandler contract is replicated to all other DataCommitHandler implementations that were registered as cohorts.
调用DataCommitHandler方法将被复制到所有的实现上
The operation is successful if and only if all invocations were successful
The lists of reported errors are merged into one, which is returned to the Data Repository
合并错误
The Broker’s implementation of the DataCommitHandler contract is visible only to Data Repositories.

DataBroker Contract
Retrieving Data
Synopsis:

Result = getData(Session)
Where:

Session: ConsumerSession between a Consumer and the Broker
Result: an RpcResult triplet in form (Success,Output,Error) where:
Success: Boolean indicating successful operation.
Output: CompositeNode representing the state data of the system (runtime or configuration).
Error: A list of errors which occurred during invocation.
An implementation of the DataBroker contract should use the provided session to learn the subtree of data that is visible to registered Consumers. It should use this information to request the relevant subset of data from the Data Repository.

Synopsis:

Result = getData(Session,Filter)
Where:

Session: ConsumerSession between a Consumer and the Broker
Filter: CompositeNode representing a subtree of the requested data CompositeNode表示请求数据的子树
Result:
Success: Boolean indicating successful operation.
Data: CompositeNode representing the state data of the system (runtime or configuration).
Error: A list of errors which occurred during invocation.
Editing Data
Synopsis:

Result = editData(Session,NodeModification)
Where:

Session: ConsumerSession between a Consumer and the Broker
NodeModification: The extended version of DOMNode, which represents the changes to be applied to data.
Result: an RpcResult triplet in form (Success,Output,Error) where:
Success: Boolean indicating successful application of NodeModification to the data tree.
Output: CompositeNode representing the modified data tree.
Error: A list of errors or warnings which occurred during the application of node modification.
Commiting Change
Synopsis:

Result = commit(Session)
Where:

Session: ConsumerSession between a Consumer and the Broker
Result: an RpcResult triplet in form (Success,Output,Error) where:
Success: Boolean indicating successful run of two-phase commit
Output: CompositeNode representing the modified data tree.
Error: A list of errors or warnings which prevented the commit to be successful.
The invocation of this operation initiates the two-phase commit.

The Broker invokes the Data Repository's commit operation, which starts the two-phase commit by issuing the requestCommit on the Broker’s implementation of the DataCommitHandler contract.

Registration of Data Commit Handlers
The following methods are used to register the Providers' implementations of the DataCommitHandler contract, which will be used as an cohorts during two-phase commit.

Synopsis:

Status = addDataCommitHandler(Session, DataCommitHandler)
Where:

Status: OK if the validator was registered successfully.
Session: ProviderSession between BI Broker and provider.
DataCommitHandler: An object implementing DataCommitHanlder contract, which is participant of two-phase commit as an cohort.
Synopsis:

Status = removeDataCommitHandler(Session, DataCommitHandler)
Where:

Status: OK if the RPC validator registered successfully.
Session: ProviderSession between BI Broker and provider.
DataValidator: An object implementing DataCommitHandler contract, which is to be removed.
Registration of Data Validators
Exposes an API to register for RPC processing and returning result.

Status = addDataValidator(Session, DataValidator)
Where:

Status: OK if the validator was registered successfully.
Session: ProviderSession between BI Broker and provider.
DataValidator: An object implementing DataValidator contract, which is called if the validation of data is required.
Status = removeDataValidator(Session, DataValidator)
Where:

Status: OK if the RPC validator registered successfully.
Session: ProviderSession between BI Broker and provider.
DataValidator: An object implementing DataValidator contract, which is to be removed.

Requirements 需求
BI Broker must keep track of: BI Broker必须保存：
All providers and consumers 所有provider和consumer
providers and RPCs exposed by them provider及暴露的RPC
providers and notifications exposed by them provider及暴露的notification
consumers and their registration for receiving notifications consumer及他们注册接收的通告

Dependencies
Binding-Independent model
Schema repository
BI Data Repository
Open Questions
Behavior if the RPC schema is unknown.

Binding-Independent Data Repository
The Binding-independent Repository holds data (the state of the system) in a tree structure.

Provided Functionality
Repository for Running State BI Data Repository存储运行状态
The BI Data Repository serves as a Provider for running state that was published by Provider modules. This repository is binding-independent and can be accessed by any Consumer / Provider through RPC calls.

Repository for Configuration State BI Data Repository存储运行状态
The BI Data Repository serves as a Provider for configuration state that was published by producer modules and modified by consumers. This repository is binding-independent and can be accessed by any Consumer / Provider through RPC calls.

Candidate State Modification
State modification consists of following steps

Changes of state
Two-Phase Commit of state
Validation of new state
Commit request phase
Commit phase
Notification of state change
Consumer’s change of state data
State data changes are modeled after NETCONF edit-config operation.

The change operations are part of binding-indepenent DOM form of data, and multiple operations could be done in one change call.

Synopsis:

Status = editData(Target,ModificationDOMNode)
Where:

Target: the change target (if multiple roots / sets of data are available)[10].
ModificationDOMNode: The extendent version of DOMNode, which represents the changes to be applied to data.
Commit of new state data
Validation of state data
Before commitment, state data must be validated. The Data Repository requests a state data validation from the Binding-Independent Broker. The BI Broker triggers data validation on all registered Providers and returns a composite validation result.

Notification of state change
The Data Repository is responsible for triggering a notification reporting the change in state data. This notification is passed to the Binding-Independent Broker, which is responsible for routing the notification to all listening Consumers and Providers.
Data Repository负责触发状态变化通告，该通过被传递给BI Broker，由其负责通告给监听的consumer和provider

Model Schema Repository
Model schema repository is infrastructure component serving as a centralized storage of all known (registered) YANG schemas /modules, which could be used and processed by other components.
Model schema repository集中存储所有已知Yang schemas/modules

It is not responsible for parsing YANG schemas, neither the understanding the extensions.
不负责处理Yang schema，或者理解拓展

Provided Functionality
Unified access to YANG schemas 统一对YANG schema访问
Access to Unified YANG schema – a joint schema tree representing all the schemas known to the system with all augmentation applied. 连接所有schema及应用拓展
Access to context YANG schemas – a joint schema tree representing the model known to the concrete components. 
Dependencies
Binding-independent model
Java YANG parser
YANG Schema

Java YANG Parser
Java YANG parser is an infrastructure component responsible for parsing input in the YANG format and providing the parsed schema in the form of Java YANG schema model, which provides programmatic access to schema trees.
Java YANG解析器负责解析YANG格式输入，提供以Java YANG schema模型形式解析后的schema

Provided Functionality
Parsing YANG Files
The Java YANG parser is responsible for parsing input stream containing YANG schema into the Java representation of the YANG schema.

Dependencies
Java representation of YANG schema

Binding-Independent Consumer
Binding-Independent consumers (and consumers in general) usually do not provide implementations of specific contracts, but use contracts provided by the Binding-independent Broker to change the state of the system and/or to invoke functionality provided by Providers.
BI consumer通常不提供实现对特定接口的实现，但会使用BI Broker提供的接口来改变系统状态或调用功能

Notification Listener
When a Consumer implements the notification listener functionality defined in the NotificationListener contract,, it can expose and register the implementation with the Broker by using the:
当consumer实现了notification listener接口

pull form – returning the objects implementing the contract in result of getFunctionality method
push form – see NotificationService contract of Binding Independent broker

Binding-Independent Provider
Exposing Provided Functionality
A Provider exposes functionality by registering its implementations of various functionality contracts with the Broker.

The registration of functionality is supported in two forms:

Pull form – at registration time, the Broker requests (pulls) the provided functionality from the Provider
Push form – at runtime, a Provider can register additional functionality by invoking the registration mechanism specific to the functionality type (contract).
The functionality is usually an implementation of one or more contracts (Java interfaces) marked as ProviderFunctionality.

Initial Registration of Functionality
To be able to register with the Broker, a Provider must implement a method with the following synopsis:

Synopsis:

Functionality = getFunctionality()    
Where:

Functionality: A set of contract implementation instances that are exposed by a Provider to the Broker and other components. This set can be empty and there is no requirement for a Provider to include all its functionality in this set.
Exposing a RPC Implementation
When a Provider implements an RPC functionality defined in the RpcImplementation contract, it can register the implementation with the Broker by using the:

pull form – returning the objects implementing the contract in result of getFunctionality method
push form – see RpcService contract of Binding Independent broker
RpcImplementation Contract
The RpcImplementation contract defines how a Provider exposes its RPC functionality.

This contract allows for support of multiple RPC types to be available in one implementation by using QNames as function parameters.

The implementation of the contract can support the pull registration of a Provider's RPCs during the Provider's registration. It must provide a method with following synopsis:

Synopsis:

Supported = getSupportedRpcs()    
Where:

Supported: A set of QNames identifying the names of the RPCs which will be provided to the broker and other consumers.
The actual implementation of the RPC functionality is done by implementing the following method:

Synopsis:

RpcResult = invokeRpc(Qname,Input)    
Where:

Qname: QName identifying invoked RPC
Input: CompositeNode representing the input data in the binding-indepenendent form, which are input to the RPC.
RpcResult: an RpcResult triplet in form (Success,Output,Error) where:
Success: A boolean indicating success of the rpc execution.
Output: CompositeNode representing the output data of rpc in the binding-independent form.
Error: a list of errors or warnings encountered during the execution of rpc.
Two Phase Commit of Configuration
When a Provider supports the commit operation by implementing the functionality in the DataCommitHandler contract, it can expose and register the implementation with the Broker by using the:

pull form – returning the objects implementing the contract in result of getFunctionality method
push form – see DataBrokerService contract of Binding Independent broker
DataCommitHandler Contract
When a Provider serves as one of the cohorts in a two phase commit of runtime or configuration data, it uses the DataCommitHandler contract. (see the “Data Repository” section for two-phase commit definition).

Request Commit
The implementation of the request commit phase (voting on the commit) has the following synopsis:

Synopsis:

Result = requestCommit()     
Where:

Result: an RpcResult triplet in form (Success,Output,Error) where:
Success: a Boolean indicating if the commit request was successful in the scope of the provider. If the Success value is False, the whole commit will be rollbacked.
Output: Additional information, usually null.
Error: A list of all errors encountered during the execution of the commit request.
Success Commit Phase
The implementation of the successful finish of the commit (voting on the commit) has the following synopsis:

Synopsis:

Result = finishCommit() 
Where:

Result: an RpcResult triplet in form (Success,Output,Error) where:
Success: a Boolean indicating if the finish commit was successful in the scope of the provider.
Output: Additional information, usually null.
Error: A list of all errors encountered during the execution of the commit request.
Rollback Commit
Synopsis:

Result = rollbackCommit()    
Where:

Result: an RpcResult triplet in form (Success,Output,Error) where:
Success: a Boolean indicating if the rollback was successful in the scope of the provider.
Output: Additional information, usually null.
Error: A list of all errors encountered during the execution of the commit request.
Data Validation
When a Provider implements the data validation functionality defined in the contract DataValidator, it can expose and register the implementation with the Broker by using the:

pull form – returning the objects implementing the contract in result of getFunctionality method
push form – see DataBroker contract of Binding Independent Broker
Data Refreshing
When a Provider implements the data refresh functionality defined in the contract DataRefresher, it can expose and register the implementation with the Broker by using the:

pull form – returning the objects implementing the contract in result of getFunctionality method
push form – see DataBroker contract of Binding Independent broker
Exposing a Nested Subsystem
Data from nested subsystem are exposed exactly like any other data in the system - by modifying system state and adding data nodes under the attachment point of the nested subsystem.

This allows for Consumers to use the same system state access contracts to access and modify data in nested subsystems, without the knowledge that the data subtree may be residing in a different system.
