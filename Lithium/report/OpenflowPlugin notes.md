# 主入口

![msg_life_cycle](https://wiki.opendaylight.org/images/f/fb/MessageLifecycle.jpg)

ConfigurableOpenFlowProviderModule是OpenFlowPlugin中启动加载的入口，如下：

```java
@Override
public java.lang.AutoCloseable createInstance() {
    pluginProvider =  new OpenflowPluginProvider();
    pluginProvider.setDataBroker(getDataBrokerDependency());
    pluginProvider.setNotificationService(getNotificationServiceDependency());
    pluginProvider.setRpcRegistry(getRpcRegistryDependency());
    pluginProvider.setSwitchConnectionProviders(getOpenflowSwitchConnectionProviderDependency());
    pluginProvider.setRole(getRole()); // 此时获得的role为缺省值NOCHANGE，该值位于AbstractConfigurableOpenFlowProviderModule中
    pluginProvider.initialization();
    return pluginProvider;
}
```

其中创建了一个OpenflowPluginProvider，即OpenflowPlugin功能实际提供者，其中调用的initialization方法将服务进行了初始化，如下：

```java
public void initialization() {
    messageCountProvider = new MessageSpyCounterImpl();
    extensionConverterManager = new ExtensionConverterManagerImpl();
    roleManager = new OFRoleManager(OFSessionUtil.getSessionManager());

    LOG.debug("dependencies gathered..");
    registrationManager = new SalRegistrationManager();
    registrationManager.setDataService(dataBroker);
    registrationManager.setPublishService(notificationService);
    registrationManager.setRpcProviderRegistry(rpcRegistry);
    registrationManager.init();

    mdController = new MDController();
    mdController.setSwitchConnectionProviders(switchConnectionProviders);
    mdController.setMessageSpyCounter(messageCountProvider);
    mdController.setExtensionConverterProvider(extensionConverterManager);
    mdController.init();
    mdController.start();
}
```

# 下行通道


其中可大体分成两部分，一部分是SalRegistrationManager，另一部分则是MDController。
SalRegistrationManager可以理解为管理了控制器到交换机的下行通道。
在SalRegistrationManager中init方法，注册了SessionListener，当控制器与交换机连接会话发生变化时，会触发onSessionAdded和onSessionRemoved方法，如下：

```java
@Override
public void onSessionAdded(final SwitchSessionKeyOF sessionKey, final SessionContext context) {
    GetFeaturesOutput features = context.getFeatures(); // 获取OFPT_FEATURES_REPLY
    BigInteger datapathId = features.getDatapathId(); // 从OFPT_FEATURES_REPLY中获取dpid
    InstanceIdentifier<Node> identifier = identifierFromDatapathId(datapathId);
    NodeRef nodeRef = new NodeRef(identifier);
    NodeId nodeId = nodeIdFromDatapathId(datapathId);
    ModelDrivenSwitchImpl ofSwitch = new ModelDrivenSwitchImpl(nodeId, identifier, context); // 创建交换机实例
    CompositeObjectRegistration<ModelDrivenSwitch> registration =
            ofSwitch.register(rpcProviderRegistry); // 注册rpc调用
    context.setProviderRegistration(registration);

    LOG.debug("ModelDrivenSwitch for {} registered to MD-SAL.", datapathId);

    NotificationQueueWrapper wrappedNotification = new NotificationQueueWrapper(
            nodeAdded(ofSwitch, features, nodeRef),
            context.getFeatures().getVersion());
    context.getNotificationEnqueuer().enqueueNotification(wrappedNotification);
}

```

可以认为每一台交换机与控制器建立连接后，控制器都会为其创建一个ModelDrivenSwitchImpl实例，并为其注册相应的rpc，而ModelDrivenSwitchImpl则实现了多个rpc接口，继承如下：

```java
public interface ModelDrivenSwitch
        extends
        SalGroupService,
        SalFlowService,
        SalMeterService, SalTableService, SalPortService, PacketProcessingService, NodeConfigService,
        OpendaylightGroupStatisticsService, OpendaylightMeterStatisticsService, OpendaylightFlowStatisticsService,
        OpendaylightPortStatisticsService, OpendaylightFlowTableStatisticsService, OpendaylightQueueStatisticsService,
        Identifiable<InstanceIdentifier<Node>>
```

而AbstractModelDrivenSwitch则实现ModelDrivenSwitch，如下：

```java
public abstract class AbstractModelDrivenSwitch implements ModelDrivenSwitch
```

在AbstractModelDrivenSwitch中，注册了所有的rpc实现为AbstractModelDrivenSwitch，由于此类为抽象类，因此具体方法的实现将由实现类完成，即ModelDrivenSwitchImpl，如下：

```java
 @Override
public CompositeObjectRegistration<ModelDrivenSwitch> register(RpcProviderRegistry rpcProviderRegistry) {
    CompositeObjectRegistrationBuilder<ModelDrivenSwitch> builder = CompositeObjectRegistration
            .<ModelDrivenSwitch> builderFor(this);

    final RoutedRpcRegistration<SalFlowService> flowRegistration = rpcProviderRegistry.addRoutedRpcImplementation(SalFlowService.class, this);
    flowRegistration.registerPath(NodeContext.class, getIdentifier()); // 将rpc路由表中的数据进行更新，getIdentifier() 方法带入更新节点的path
    builder.add(flowRegistration);
    
    ...
}
```
上述rpc接口的实现均在ModelDrivenSwitchImpl中，以SalFlowService为例，如下：

```java
@Override
public Future<RpcResult<AddFlowOutput>> addFlow(final AddFlowInput input) {
    LOG.debug("Calling the FlowMod RPC method on MessageDispatchService");
    // use primary connection
    SwitchConnectionDistinguisher cookie = null;

    OFRpcTask<AddFlowInput, RpcResult<UpdateFlowOutput>> task =
            OFRpcTaskFactory.createAddFlowTask(rpcTaskContext, input, cookie);
    ListenableFuture<RpcResult<UpdateFlowOutput>> result = task.submit();

    return Futures.transform(result, OFRpcFutureResultTransformFactory.createForAddFlowOutput());
}
```

# 上行通道

MDController负责控制器与交换机的信令交互，即非流表、组表消息的交互，可以理解为控制器与交换机的上行通道管理。在init方法中，交换机与控制器消息处理实现被添加到映射中，如下：

```java
OpenflowPortsUtil.init(); // 完成协议中端口定义的映射
...
// 每个translator对应了Openflow协议中一种消息，负责将Of消息转换为MD-SAL中的各个notification
addMessageTranslator(ErrorMessage.class, OF10, new ErrorV10Translator());
addMessageTranslator(ErrorMessage.class, OF13, new ErrorTranslator());
addMessageTranslator(FlowRemovedMessage.class, OF10, new FlowRemovedTranslator());
addMessageTranslator(FlowRemovedMessage.class, OF13, new FlowRemovedTranslator());
...
// 制定了每种notification的通用pulisher
addMessagePopListener(NodeErrorNotification.class, notificationPopListener);
addMessagePopListener(BadActionErrorNotification.class, notificationPopListener);
addMessagePopListener(BadInstructionErrorNotification.class, notificationPopListener);
addMessagePopListener(BadMatchErrorNotification.class, notificationPopListener);
```

在init后，调用start方法创建SwitchConnectionHandlerImpl负责与处理交换机连接，start方法会启动一系列SwitchConnectionHandler，这些SwitchConnectionHandler会依次处理连接，以找到一个合适的，如下：

```java
List<ListenableFuture<Boolean>> starterChain = new ArrayList<>(switchConnectionProviders.size());
for (SwitchConnectionProvider switchConnectionPrv : switchConnectionProviders) {
    switchConnectionPrv.setSwitchConnectionHandler(switchConnectionHandler);
    ListenableFuture<Boolean> isOnlineFuture = switchConnectionPrv.startup();
    starterChain.add(isOnlineFuture);
}
```

# 角色管理

除去SalRegistrationManager与MDController，OpenflowPluginProvider还创建了一个OFRoleManager实例，在OpenflowPluginProvider中与Role相关的方法如下：

```java
/**
 * @param role of instance
 */
public void setRole(OfpRole role) {
    this.role = role;
}

/**
 * @param newRole
 */
public void fireRoleChange(OfpRole newRole) {
    if (!role.equals(newRole)) {
        LOG.debug("my role was chaged from {} to {}", role, newRole);
        role = newRole;
        switch (role) {
        case BECOMEMASTER:
            //TODO: implement appropriate action
            roleManager.manageRoleChange(role);
            break;
        case BECOMESLAVE:
            //TODO: implement appropriate action
            roleManager.manageRoleChange(role);
            break;
        case NOCHANGE:
            //TODO: implement appropriate action
            roleManager.manageRoleChange(role);
            break;
        default:
            LOG.warn("role not supported: {}", role);
            break;
        }
    }
}
```

从方法实现的角度看，似乎role在OpenflowPluginProvider中是一个集中控制的对象，并非与交换机节点绑定，即对于一定区域内的所有交换机而言，只能出现一个一个master，而不能出现某个交换机在某个控制器上为master的情况。从OFRoleManager提供的方法看似乎也验证了这一点，如下：

```java
/**
 * @param sessionManager
 */
public OFRoleManager(final SessionManager sessionManager) {
    Preconditions.checkNotNull("Session manager can not be empty.", sessionManager);
    this.sessionManager = sessionManager;
    workQueue = new PriorityBlockingQueue<>(500, new Comparator<RolePushTask>() { // 队列容量为500
        @Override
        public int compare(final RolePushTask o1, final RolePushTask o2) {
            return Integer.compare(o1.getPriority(), o2.getPriority()); // 按优先级排列的队列
        }
    });
    ThreadPoolLoggingExecutor delegate = new ThreadPoolLoggingExecutor(
            1, 1, 0, TimeUnit.MILLISECONDS, new ArrayBlockingQueue<Runnable>(1), "ofRoleBroadcast");
    broadcastPool = MoreExecutors.listeningDecorator(
            delegate);
}

/**
 * change role on each connected device
 *
 * @param role
 */
public void manageRoleChange(final OfpRole role) {
    for (final SessionContext session : sessionManager.getAllSessions()) { // 遍历所有的连接会话
        try {
            workQueue.put(new RolePushTask(role, session));
        } catch (InterruptedException e) {
            LOG.warn("Processing of role request failed while enqueueing role task: {}", e.getMessage());
        }
    }

    while (!workQueue.isEmpty()) {
        RolePushTask task = workQueue.poll();
        ListenableFuture<Boolean> rolePushResult = broadcastPool.submit(task); // 该方法会调用RolePushTask中的call方法
        CheckedFuture<Boolean, RolePushException> rolePushResultChecked =
                RoleUtil.makeCheckedRuleRequestFxResult(rolePushResult);
        try {
            Boolean succeeded = rolePushResultChecked.checkedGet(TIMEOUT, TIMEOUT_UNIT);
            if (!MoreObjects.firstNonNull(succeeded, Boolean.FALSE)) {
                if (task.getRetryCounter() < RETRY_LIMIT) {
                    workQueue.offer(task); // 修改失败role且失败次数小于重试次数的会话重新存入queue
                }
            }
        } catch (RolePushException | TimeoutException e) {
            LOG.warn("failed to process role request: {}", e);
        }
    }
}
```

RolePushTask中的call方法将发送RoleRequest，如下：

```java
generationId = RoleUtil.getNextGenerationId(generationId);

// try to possess role on device
Future<RpcResult<RoleRequestOutput>> roleReply = RoleUtil.sendRoleChangeRequest(session, role, generationId);
// flush election result with barrier
BarrierInput barrierInput = MessageFactory.createBarrier(
        session.getFeatures().getVersion(), session.getNextXid());
Future<RpcResult<BarrierOutput>> barrierResult = session.getPrimaryConductor().getConnectionAdapter().barrier(barrierInput);
```

改变Role的顶层调用在OpenflowPluginProvider的fireRoleChange方法中，如下，该方法只在ConfigurableOpenFlowProviderModule中reuseInstance被调用

```java
public void fireRoleChange(OfpRole newRole) {
    if (!role.equals(newRole)) {
        LOG.debug("my role was chaged from {} to {}", role, newRole);
        role = newRole;
        switch (role) {
        case BECOMEMASTER:
            //TODO: implement appropriate action
            roleManager.manageRoleChange(role);
            break;
        case BECOMESLAVE:
            //TODO: implement appropriate action
            roleManager.manageRoleChange(role);
            break;
        case NOCHANGE:
            //TODO: implement appropriate action
            roleManager.manageRoleChange(role);
            break;
        default:
            LOG.warn("role not supported: {}", role);
            break;
        }
    }
}
```

从代码看目前对于角色转换的功能是需要开发者加入的，并从外部调用实现交换机角色的切换，且更倾向于主备的实现方式。

# Openflow消息转译

MDController中注册了各种Openflow协议消息的处理器，这些处理器均继承自IMDMessageTranslator<I, O>，这是一个翻译器，所有到MD-SAL或往MD-SAL的消息都由它处理，它只有一个方法，如下：

```java
/**
 * This method is called in order to translate message to MD-SAL or from MD-SAL.
 *
 * @param cookie
 *            auxiliary connection identifier
 * @param sc
 *            The SessionContext which sent the OF message
 * @param msg
 *            The OF message
 *
 * @return translated message
 */
O translate(SwitchConnectionDistinguisher cookie, SessionContext sc, I msg);
```

其中cookie一般不会被使用，translate方法可以参照以下步骤实现：

```java
@Override
public List<DataObject> translate(SwitchConnectionDistinguisher cookie,
        SessionContext sc, OfHeader msg) {
    if(msg instanceof OF_MESSAGE.class) { // 判断消息是否属于要处理的消息类型
        // 按消息格式填充各个字段
        ...
        return list;
    } else {
        return Collections.emptyList(); // 处理出错则返回一个空的列表
    }
}
```

# MDController

调用
