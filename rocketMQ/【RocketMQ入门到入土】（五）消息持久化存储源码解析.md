# 一、原理

## 1、消息存在哪了？

消息持久化的地方其实是磁盘上，在如下目录里的commitlog文件夹里。

```
/root/store/commitlog
```

源码如下：

```
// {@link org.apache.rocketmq.store.config.MessageStoreConfig}

// 数据存储根目录
private String storePathRootDir = System.getProperty("user.home") + File.separator + "store";
// commitlog目录
private String storePathCommitLog = System.getProperty("user.home") + File.separator + "store" + File.separator + "commitlog";
// 每个commitlog文件大小为1GB，超过1GB则创建新的commitlog文件
private int mappedFileSizeCommitLog = 1024 * 1024 * 1024;
```

比如验证下：

```
[root@iZ2ze84zygpzjw5bfcmh2hZ commitlog]# pwd
/root/store/commitlog
[root@iZ2ze84zygpzjw5bfcmh2hZ commitlog]# ll -h
total 400K
-rw-r--r-- 1 root root 1.0G Jun 30 18:21 00000000000000000000
[root@iZ2ze84zygpzjw5bfcmh2hZ commitlog]#
```

> 可以清晰的看到文件大小是1.0G，超过1.0G再写入消息的话会自动创建新的commitlog文件。

## 2、关键类解释

### 2.1、MappedFile

对应的是commitlog文件，比如上面的`00000000000000000000`文件。

### 2.2、MappedFileQueue

是`MappedFile` 所在的文件夹，对 `MappedFile` 进行封装成文件队列。

### 2.3、CommitLog

针对 `MappedFileQueue` 的封装使用。

# 二、Broker接收消息

## 1、调用链

```
BrokerStartup.start() -》 BrokerController.start() -》 NettyRemotingServer.start() -》 NettyRemotingServer.prepareSharableHandlers() -》 new NettyServerHandler() -》 NettyRemotingAbstract.processMessageReceived() 
-》 NettyRemotingAbstract.processRequestCommand() -》 SendMessageProcessor.processRequest()
```

## 2、processRequest

> SendMessageProcessor.processRequest()

```
@Override
public RemotingCommand processRequest(ChannelHandlerContext ctx,
                                      RemotingCommand request) throws RemotingCommandException {
    RemotingCommand response = null;
    try {
        // 调用asyncProcessRequest
        response = asyncProcessRequest(ctx, request).get();
    } catch (InterruptedException | ExecutionException e) {
        log.error("process SendMessage error, request : " + request.toString(), e);
    }
    return response;
}
```

## 3、asyncProcessRequest

```
public CompletableFuture<RemotingCommand> asyncProcessRequest(ChannelHandlerContext ctx,
                                                                  RemotingCommand request) throws RemotingCommandException {
    final SendMessageContext mqtraceContext;
    switch (request.getCode()) {
        // 表示消费者发送的消息，发送者消费失败会重新发回队列进行消息重试
        case RequestCode.CONSUMER_SEND_MSG_BACK:
            return this.asyncConsumerSendMsgBack(ctx, request);
        default:
            // 解析header，也就是我们Producer发送过来的消息都在request里，给他解析到SendMessageRequestHeader对象里去。
            SendMessageRequestHeader requestHeader = parseRequestHeader(request);
            if (requestHeader == null) {
                return CompletableFuture.completedFuture(null);
            }
            mqtraceContext = buildMsgContext(ctx, requestHeader);
            // 将解析好的参数放到SendMessageContext对象里
            this.executeSendMessageHookBefore(ctx, request, mqtraceContext);
            if (requestHeader.isBatch()) {
                // 批处理消息用
                return this.asyncSendBatchMessage(ctx, request, mqtraceContext, requestHeader);
            } else {
                // 非批处理，我们这里介绍的核心。
                return this.asyncSendMessage(ctx, request, mqtraceContext, requestHeader);
            }
    }
}
```

## 4、asyncSendMessage

```
private CompletableFuture<RemotingCommand> asyncSendMessage(ChannelHandlerContext ctx, RemotingCommand request,
                                                                SendMessageContext mqtraceContext,
                                                                SendMessageRequestHeader requestHeader) {
    final byte[] body = request.getBody();

    int queueIdInt = requestHeader.getQueueId();
    TopicConfig topicConfig = this.brokerController.getTopicConfigManager().selectTopicConfig(requestHeader.getTopic());

    // 拼凑message对象
    MessageExtBrokerInner msgInner = new MessageExtBrokerInner();
    msgInner.setTopic(requestHeader.getTopic());
    msgInner.setQueueId(queueIdInt);
    msgInner.setBody(body);
    msgInner.setFlag(requestHeader.getFlag());
    MessageAccessor.setProperties(msgInner, MessageDecoder.string2messageProperties(requestHeader.getProperties()));
    msgInner.setPropertiesString(requestHeader.getProperties());
    msgInner.setBornTimestamp(requestHeader.getBornTimestamp());
    msgInner.setBornHost(ctx.channel().remoteAddress());
    msgInner.setStoreHost(this.getStoreHost());
    msgInner.setReconsumeTimes(requestHeader.getReconsumeTimes() == null ? 0 : requestHeader.getReconsumeTimes());
    
    CompletableFuture<PutMessageResult> putMessageResult = null;
    Map<String, String> origProps = MessageDecoder.string2messageProperties(requestHeader.getProperties());
    // 真正接收消息的方法
    putMessageResult = this.brokerController.getMessageStore().asyncPutMessage(msgInner);
    return handlePutMessageResultFuture(putMessageResult, response, request, msgInner, responseHeader, mqtraceContext, ctx, queueIdInt);
}
```

> 至此我们的消息接收完成了，都封装到了MessageExtBrokerInner对象里。

# 三、Broker消息存储（持久化）

## 1、asyncPutMessage

> 接着上步骤的asyncSendMessage继续看

```
@Override
public CompletableFuture<PutMessageResult> asyncPutMessage(MessageExtBrokerInner msg) {
    CompletableFuture<PutMessageResult> putResultFuture = this.commitLog.asyncPutMessage(msg);
    putResultFuture.thenAccept((result) -> {
        ......
    });
    return putResultFuture;
}
```

## 2、commitLog.asyncPutMessage

```
public CompletableFuture<PutMessageResult> asyncPutMessage(final MessageExtBrokerInner msg) {
    // 获取最后一个文件，MappedFile就是commitlog目录下的那个0000000000文件
    MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();
    try {
        // 追加数据到commitlog
        result = mappedFile.appendMessage(msg, this.appendMessageCallback);
        switch (result.getStatus()) {
            ......
        }
        // 将内存的数据持久化到磁盘
        CompletableFuture<PutMessageStatus> flushResultFuture = submitFlushRequest(result, putMessageResult, msg);
    }
}
```

## 3、appendMessagesInner

```
public AppendMessageResult appendMessagesInner(final MessageExt messageExt, final AppendMessageCallback cb) {
    // 将消息写到内存
    return cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBrokerInner) messageExt);
}
```

## 4、doAppend

```
@Override
public AppendMessageResult doAppend(final long fileFromOffset, final ByteBuffer byteBuffer, final int maxBlank,
                                    final MessageExtBrokerInner msgInner) {
    // Initialization of storage space
    this.resetByteBuffer(msgStoreItemMemory, msgLen);
    // 1 TOTALSIZE
    this.msgStoreItemMemory.putInt(msgLen);
    // 2 MAGICCODE
    this.msgStoreItemMemory.putInt(CommitLog.MESSAGE_MAGIC_CODE);
    // 3 BODYCRC
    this.msgStoreItemMemory.putInt(msgInner.getBodyCRC());
    // 4 QUEUEID
    this.msgStoreItemMemory.putInt(msgInner.getQueueId());
    // 5 FLAG
    this.msgStoreItemMemory.putInt(msgInner.getFlag());
    // 6 QUEUEOFFSET
    this.msgStoreItemMemory.putLong(queueOffset);
    // 7 PHYSICALOFFSET
    this.msgStoreItemMemory.putLong(fileFromOffset + byteBuffer.position());
    // 8 SYSFLAG
    this.msgStoreItemMemory.putInt(msgInner.getSysFlag());
    // 9 BORNTIMESTAMP
    this.msgStoreItemMemory.putLong(msgInner.getBornTimestamp());
    // 10 BORNHOST
    this.resetByteBuffer(bornHostHolder, bornHostLength);
    this.msgStoreItemMemory.put(msgInner.getBornHostBytes(bornHostHolder));
    // 11 STORETIMESTAMP
    this.msgStoreItemMemory.putLong(msgInner.getStoreTimestamp());
    // 12 STOREHOSTADDRESS
    this.resetByteBuffer(storeHostHolder, storeHostLength);
    this.msgStoreItemMemory.put(msgInner.getStoreHostBytes(storeHostHolder));
    // 13 RECONSUMETIMES
    this.msgStoreItemMemory.putInt(msgInner.getReconsumeTimes());
    // 14 Prepared Transaction Offset
    this.msgStoreItemMemory.putLong(msgInner.getPreparedTransactionOffset());
    // 15 BODY
    this.msgStoreItemMemory.putInt(bodyLength);
    if (bodyLength > 0)
        this.msgStoreItemMemory.put(msgInner.getBody());
    // 16 TOPIC
    this.msgStoreItemMemory.put((byte) topicLength);
    this.msgStoreItemMemory.put(topicData);
    // 17 PROPERTIES
    this.msgStoreItemMemory.putShort((short) propertiesLength);
    if (propertiesLength > 0)
        this.msgStoreItemMemory.put(propertiesData);

    final long beginTimeMills = CommitLog.this.defaultMessageStore.now();
    // Write messages to the queue buffer
    byteBuffer.put(this.msgStoreItemMemory.array(), 0, msgLen);
    return result;
}
```

> 这一步其实就已经把消息保存到缓冲区里了，也就是msgStoreItemMemory，这里采取的NIO。
>
> ```
> private final ByteBuffer msgStoreItemMemory;
> ```

## 5、submitFlushRequest

> 再次回到【2、commitLog.asyncPutMessage】的submitFlushRequest方法，因为之前的方法是将数据已经写到ByteBuffer缓冲区里了，下一步也就是我们现在这一步就要刷盘了。

```
public CompletableFuture<PutMessageStatus> submitFlushRequest(AppendMessageResult result, PutMessageResult putMessageResult,
                                                              MessageExt messageExt) {
    // 同步刷盘
    if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
        final GroupCommitService service = (GroupCommitService) this.flushCommitLogService;
        if (messageExt.isWaitStoreMsgOK()) {
            GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes(),
                                                                this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
            service.putRequest(request);
            return request.future();
        } else {
            service.wakeup();
            return CompletableFuture.completedFuture(PutMessageStatus.PUT_OK);
        }
    }
    // 异步刷盘
    else {
        if (!this.defaultMessageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
            
            flushCommitLogService.wakeup();
        } else  {
            commitLogService.wakeup();
        }
        return CompletableFuture.completedFuture(PutMessageStatus.PUT_OK);
    }
}
```

## 6、异步刷盘

```
class FlushRealTimeService extends FlushCommitLogService {
    @Override
    public void run() {
        while (!this.isStopped()) {
            try {
    // 每隔500ms刷一次盘
                if (flushCommitLogTimed) {
                    Thread.sleep(500);
                } else {
                    this.waitForRunning(500);
                }
                // 调用mappedFileQueue的flush方法
                CommitLog.this.mappedFileQueue.flush(flushPhysicQueueLeastPages);
            } catch (Throwable e) {
            }
        }
    }
}
```

> 可看出默认是每隔500毫秒刷一次盘

## 7、mappedFileQueue.flush

```
public boolean flush(final int flushLeastPages) {
    MappedFile mappedFile = this.findMappedFileByOffset(this.flushedWhere, this.flushedWhere == 0);
    if (mappedFile != null) {
        // 真正的刷盘操作
        int offset = mappedFile.flush(flushLeastPages);
    }
}
```

## 8、mappedFile.flush

```
public int flush(final int flushLeastPages) {
    if (this.isAbleToFlush(flushLeastPages)) {
        try {
            if (writeBuffer != null || this.fileChannel.position() != 0) {
                // 刷盘   NIO
                this.fileChannel.force(false);
            } else {
    // 刷盘  NIO
                this.mappedByteBuffer.force();
            }
        } catch (Throwable e) {
            log.error("Error occurred when force data to disk.", e);
        }
    }
    return this.getFlushedPosition();
}
```

> 至此已经全部结束。

# 四、总结

Broker收到消息后怎么持久化的？

有两种方式：同步和异步。一般选择异步，同步效率低，但是更可靠。消息存储大致原理是：

核心类MappedFile对应的是每个commitlog文件，MappedFileQueue相当于文件夹，管理所有的文件，还有一个管理者CommitLog对象，他负责提供一些操作。具体的是Broker端拿到消息后先将消息、topic、queue等内容存到ByteBuffer里，然后去持久化到commitlog文件中。commitlog文件大小为1G，超出大小会新创建commitlog文件来存储，采取的nio方式。

# 五、补充：同步/异步刷盘

## 1、关键类

|         类名          |                 描述                 | 刷盘性能 |
| :-------------------: | :----------------------------------: | :------: |
| CommitRealTimeService |      异步刷盘 &&开启字节缓冲区       |   最高   |
| FlushRealTimeService  |     异步刷盘&&关闭内存字节缓冲区     |   较高   |
|  GroupCommitService   | 同步刷盘，刷完盘才会返回消息写入成功 |   最低   |

## 2、图解

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucic0Cq1ibiaRqHIo3JH148Okicpa0MzicAyWnYSQ2Jx8rZhzbK9UffBgZKy3DHkjoPDxGbo5y2uczWvibQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 3、同步刷盘

### 3.1、源码

```
// {@link org.apache.rocketmq.store.CommitLog#submitFlushRequest()}
// Synchronization flush
if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
    // 同步刷盘service -> GroupCommitService
    final GroupCommitService service = (GroupCommitService) this.flushCommitLogService;
    if (messageExt.isWaitStoreMsgOK()) {
        // 数据准备
        GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes(),
                                                 this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
        // 将数据对象放到requestsWrite里
        service.putRequest(request);
        return request.future();
    } else {
        service.wakeup();
        return CompletableFuture.completedFuture(PutMessageStatus.PUT_OK);
    }
}
```

*putRequest*

```
public synchronized void putRequest(final GroupCommitRequest request) {
    synchronized (this.requestsWrite) {
        this.requestsWrite.add(request);
    }
    // 这里很关键！！！，给他设置成true。然后计数器-1。下面run方法的时候才会进行交换数据且return
    if (hasNotified.compareAndSet(false, true)) {
        waitPoint.countDown(); // notify
    }
}
```

*run*

```
public void run() {
    while (!this.isStopped()) {
        try {
            // 是同步还是异步的关键方法，也就是说组不阻塞全看这里。
            this.waitForRunning(10);
            // 真正的刷盘逻辑
            this.doCommit();
        } catch (Exception e) {
            CommitLog.log.warn(this.getServiceName() + " service has exception. ", e);
        }
    }
}
```

*waitForRunning*

```
protected volatile AtomicBoolean hasNotified = new AtomicBoolean(false);
// 其实就是CountDownLatch
protected final CountDownLatch2 waitPoint = new CountDownLatch2(1);

protected void waitForRunning(long interval) {
    // 如果是true，且给他改成false成功的话，则onWaitEnd()且return，但是默认是false，也就是默认情况下这个if不会进。
    if (hasNotified.compareAndSet(true, false)) {
        this.onWaitEnd();
        return;
    }

    //entry to wait
    waitPoint.reset();

    try {
        // 等待，默认值是1，也就是waitPoint.countDown()一次后就会激活这里。
        waitPoint.await(interval, TimeUnit.MILLISECONDS);
    } catch (InterruptedException e) {
        log.error("Interrupted", e);
    } finally {
        // 给状态值设置成false
        hasNotified.set(false);
        this.onWaitEnd();
    }
}
```

### 3.2、总结

总结下同步刷盘的主要流程：

> 核心类是GroupCommitService，核心方法 是waitForRunning。

- 先调用putRequest方法将hasNotified变为true，且进行notify，也就是`waitPoint.countDown()`。
- 其次是run方法里的`waitForRunning()`，`waitForRunning()`判断hasNotified是不是true，是true则交换数据然后return掉，也就是不进行await阻塞，直接return。
- 最后上一步return了，没有阻塞，那么顺理成章的调用doCommit进行真正意义的刷盘。

## 4、异步刷盘

### 4.1、源码

> 核心类是：FlushRealTimeService

```
// {@link org.apache.rocketmq.store.CommitLog#submitFlushRequest()}
// Asynchronous flush
if (!this.defaultMessageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
    flushCommitLogService.wakeup();
} else  {
    commitLogService.wakeup();
}
return CompletableFuture.completedFuture(PutMessageStatus.PUT_OK);
```

*run*

```
// {@link org.apache.rocketmq.store.CommitLog.FlushRealTimeService#run()}

class FlushRealTimeService extends FlushCommitLogService {
    @Override
    public void run() {
        while (!this.isStopped()) {
            try {
    // 每隔500ms刷一次盘
                if (flushCommitLogTimed) {
                    Thread.sleep(500);
                } else {
                    // 根上面同步刷盘调用的是同一个方法，区别在于这里没有将hasNotified变为true，也就是还是默认的false，那么waitForRunning方法内部的第一个判断就不会走，就不会return掉，就会进行下面的await方法阻塞，默认阻塞时间是500毫秒。也就是默认500ms刷一次盘。
                    this.waitForRunning(500);
                }
                // 调用mappedFileQueue的flush方法
                CommitLog.this.mappedFileQueue.flush(flushPhysicQueueLeastPages);
            } catch (Throwable e) {
            }
        }
    }
}
```

### 4.2、总结

> 核心类#方法：FlushRealTimeService#run()

- 判断`flushCommitLogTimed`是不是true，默认false，是true则直接sleep(500ms)然后进行`mappedFileQueue.flush()`刷盘。
- 若是false，则进入`waitForRunning(500)`，这里是和同步刷盘的区别关键所在，同步刷盘之前将hasNotified变为true了，所以直接一套小连招：`return+doCommit`了 ，异步这里直接调用的`waitForRunning(500)`，在这之前没任何对hasNotified的操作，所以不会return，而是会继续走下面的`waitPoint.await(500, TimeUnit.MILLISECONDS);`进行阻塞500毫秒，500毫秒后自动唤醒然后进行flush刷盘。也就是异步刷盘的话默认500ms刷盘一次。



**END**