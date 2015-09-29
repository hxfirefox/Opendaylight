MDController.java 中的start方法，创建了SwitchConnectionHandlerImpl实例
```java
SwitchConnectionHandlerImpl switchConnectionHandler = new SwitchConnectionHandlerImpl();
```
在SwitchConnectionHandlerImpl从命名理解即为交换机连接处理，在其构造方法中创建了QueueProcessorLightImpl实例。随后在start方法中调用了init方法对SwitchConnectionHandlerImpl进行初始化，该过程中传递给OF协议消息处理的上下行处理器，同时调用了QueueProcessorLightImpl的init方法，该方法创建了3个线程池，分别是processorPool，harvesterPool，finisherPool，用来处理消息，传递消息及处理消息处理结果。
```java
public void init() {
        int ticketQueueCapacity = 1500;
        ticketQueue = new ArrayBlockingQueue<>(ticketQueueCapacity);
        /*
         * TODO FIXME - DOES THIS REALLY NEED TO BE CONCURRENT?  Can we figure out
         * a better lifecycle?  Why does this have to be a Set?
         */
        messageSources = new CopyOnWriteArraySet<>();

        processorPool = new ThreadPoolLoggingExecutor(processingPoolSize, processingPoolSize, 0,
                TimeUnit.MILLISECONDS,
                new ArrayBlockingQueue<Runnable>(ticketQueueCapacity),
                "OFmsgProcessor"); // 负责处理消息
        // force blocking when pool queue is full
        processorPool.setRejectedExecutionHandler(new RejectedExecutionHandler() {
            @Override
            public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
                try {
                    executor.getQueue().put(r);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    throw new IllegalStateException(e);
                }
            }
        });

        harvesterPool = new ThreadPoolLoggingExecutor(1, 1, 0,
                TimeUnit.MILLISECONDS, new ArrayBlockingQueue<Runnable>(1), "OFmsgHarvester"); // 负责消息传递
        finisherPool = new ThreadPoolLoggingExecutor(1, 1, 0,
                TimeUnit.MILLISECONDS, new ArrayBlockingQueue<Runnable>(1), "OFmsgFinisher"); // 处理消息转译完成
        finisher = new TicketFinisherImpl(
                ticketQueue, popListenersMapping); // OF消息处理结果将从ticketQueue中获取，目前由于为空，因此处于阻塞状态
        finisherPool.execute(finisher);

        harvester = new QueueKeeperHarvester<OfHeader>(this, messageSources);
        harvesterPool.execute(harvester);

        ticketProcessorFactory = new TicketProcessorFactoryImpl(); // OF消息处理工厂
        ticketProcessorFactory.setTranslatorMapping(translatorMapping);
        ticketProcessorFactory.setSpy(messageSpy);
        ticketProcessorFactory.setTicketFinisher(finisher);
 }
```
其中harvester是串联起前后消息传递的重要手段，如下
```java
harvester = new QueueKeeperHarvester<OfHeader>(this, messageSources);
```
创建给harvest时，传入的参数分别是QueueProcessorLightImpl实例本身与一个装载OF消息的集合，messageSources作为消息源来自ConnectionConductor的注册，如下：
```java
QueueKeeperFactory.plugQueue(queueProcessor, queue);
```
```java
public static <V> void plugQueue(
        MessageSourcePollRegistrator<QueueKeeper<V>> sourceRegistrator,
        QueueKeeper<V> queueKeeper) {
    AutoCloseable registration = sourceRegistrator
            .registerMessageSource(queueKeeper);
    queueKeeper.setPollRegistration(registration);
    sourceRegistrator.getHarvesterHandle().ping();
}
```
harsvert从集合中取出单个消息进入QueueProcessorLightImpl实例的ticket处理流程中。如下：
```java
 boolean starving = true;
 for (QueueKeeper<IN> source : messageSources) {
     QueueItem<IN> qItem = source.poll(); // queueZipper.poll()
     if (qItem != null) {
         starving = false;
         enqueuer.enqueueQueueItem(qItem); // 调用即为QueueProcessorLightImpl中的enqueueQueueItem方法
     }
 }
```
QueueProcessorLightImpl中enqueueQueueItem方法如下：
```java
@Override
public void enqueueQueueItem(QueueItem<OfHeader> queueItem) {
    messageSpy.spyMessage(queueItem.getMessage(), STATISTIC_GROUP.FROM_SWITCH_ENQUEUED);
    TicketImpl<OfHeader, DataObject> ticket = new TicketImpl<>(); // 输入为OF消息，输出为MD-SAL消息
    ticket.setConductor(queueItem.getConnectionConductor());
    ticket.setMessage(queueItem.getMessage());
    ticket.setQueueType(queueItem.getQueueType());

    LOG.trace("ticket scheduling: {}, ticket: {}",
            queueItem.getMessage().getImplementedInterface().getSimpleName(),
            System.identityHashCode(queueItem));
    scheduleTicket(ticket); // 进入线程处理
}
```
scheduleTicket方法将根据queue类型来选择线程，关于queue类型可参见《OpenDaylight OpenFlow Plugin 过载保护》，如下：
```java
private void scheduleTicket(Ticket<OfHeader, DataObject> ticket) {
    switch (ticket.getQueueType()) {
    case DEFAULT: // 处理非pktin消息
        Runnable ticketProcessor = ticketProcessorFactory.createProcessor(ticket); // 创建消息处理任务
        processorPool.execute(ticketProcessor); // 放入处理线程池
        try {
            ticketQueue.put(ticket); // 结果放入队列
        } catch (InterruptedException e) {
            LOG.warn("enqeueue of unordered message ticket failed", e);
        }
        break;
    case UNORDERED: // 处理pktin消息
        Runnable ticketProcessorSync = ticketProcessorFactory.createSyncProcessor(ticket);
        processorPool.execute(ticketProcessorSync);
        break;
    default:
        LOG.warn("unsupported enqueue type: {}", ticket.getQueueType());
    }
}
```
消息处理如下：
```java
Runnable ticketProcessor = new Runnable() {
    @Override
    public void run() {
        LOG.debug("message received, type: {}", ticket.getMessage().getImplementedInterface().getSimpleName());
        List<DataObject> translate;
        try {
            translate = translate(ticket); // 翻译OF消息
            ticket.getResult().set(translate); // 异步结果
            ticket.setDirectResult(translate); // 直接返回结果
            // spying on result
            if (spy != null) {
                spy.spyIn(ticket.getMessage());
                for (DataObject outMessage : translate) {
                    spy.spyOut(outMessage);
                }
            }
        } catch (Exception e) {
            LOG.warn("translation problem: {}", e.getMessage());
            ticket.getResult().setException(e);
        }
        LOG.debug("message processing done (type: {}, ticket: {})",
                ticket.getMessage().getImplementedInterface().getSimpleName(),
                System.identityHashCode(ticket));
    }
};
```
translate方法中遍历translator对OF消息进行处理，如下：
```java
for (IMDMessageTranslator<OfHeader, List<DataObject>> translator : translators) {
                SwitchConnectionDistinguisher cookie = null;
                // Pass cookie only for PACKT_OfHeader
                if (messageType.equals("PacketInMessage.class")) {
                    cookie = conductor.getAuxiliaryKey();
                }
                long start = System.nanoTime();
                List<DataObject> translatorOutput = translator.translate(cookie, conductor.getSessionContext(), message); // 仅当消息符合translator，List<DataObject>中才会有值，此处使用遍历是否效率较低？
                long end = System.nanoTime();
                LOG.trace("translator: {} elapsed time {} ns",translator,end-start);
                if(translatorOutput != null && !translatorOutput.isEmpty()) {
                    result.addAll(translatorOutput);
                }
            }
```
