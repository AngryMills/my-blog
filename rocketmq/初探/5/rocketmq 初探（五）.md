# rocketmq 初探（五）

大家好，我是烤鸭：

&nbsp;&nbsp;&nbsp;&nbsp;上一篇简单介绍部分 NettyRequestProcessor (AdminBrokerProcessor、ClientManageProcessor、ConsumerManageProcessor、EndTransactionProcessor)，这一篇介绍其他的。

PullMessageProcessor、QueryMessageProcessor、ReplyMessageProcessor、SendMessageProcessor

## NettyRequestProcessor
PullMessageProcessor，一个将近300行的代码，差点没给我送走了，把一些无关的判断直接...了，为了省篇幅。

```
private RemotingCommand processRequest(final Channel channel, RemotingCommand request, boolean brokerAllowSuspend)
    throws RemotingCommandException {
    RemotingCommand response = RemotingCommand.createResponseCommand(PullMessageResponseHeader.class);
    final PullMessageResponseHeader responseHeader = (PullMessageResponseHeader) response.readCustomHeader();
    final PullMessageRequestHeader requestHeader =
        (PullMessageRequestHeader) request.decodeCommandCustomHeader(PullMessageRequestHeader.class);
    response.setOpaque(request.getOpaque());

    log.debug("receive PullMessage request command, {}", request);
	// 当前节点是否有读权限
    if (!PermName.isReadable(this.brokerController.getBrokerConfig().getBrokerPermission())) {
        response.setCode(ResponseCode.NO_PERMISSION);
        response.setRemark(String.format("the broker[%s] pulling message is forbidden", this.brokerController.getBrokerConfig().getBrokerIP1()));
        return response;
    }
	// 获取订阅关系进行判断
    SubscriptionGroupConfig subscriptionGroupConfig =        this.brokerController.getSubscriptionGroupManager().findSubscriptionGroupConfig(requestHeader.getConsumerGroup());
    // ... 是否为空,(里边的customer)是否可以消费
    // ...
    // 获取topic 配置进行判断
    TopicConfig topicConfig = this.brokerController.getTopicConfigManager().selectTopicConfig(requestHeader.getTopic());
    // ... 是否为空,是否读权限,请求的queueId<0 或者 <topic的readQueueNums

    SubscriptionData subscriptionData = null;
    ConsumerFilterData consumerFilterData = null;
    // 有订阅关系,解析订阅信息(获取消费标记)
    if (hasSubscriptionFlag) {
        try {
            subscriptionData = FilterAPI.build(
                requestHeader.getTopic(), requestHeader.getSubscription(), requestHeader.getExpressionType()
            );
            // expressionType 不是 tag类型的,根据request创建了一个consumerFilterData(里边有topic、customerGroup)
            if (!ExpressionType.isTagType(subscriptionData.getExpressionType())) {
                consumerFilterData = ConsumerFilterManager.build(
                    requestHeader.getTopic(), requestHeader.getConsumerGroup(), requestHeader.getSubscription(),
                    requestHeader.getExpressionType(), requestHeader.getSubVersion()
                );
                assert consumerFilterData != null;
            }
        } catch (Exception e) {
            // ...
        }
    } else {
    	// 获取 consumer 组信息
        ConsumerGroupInfo consumerGroupInfo =           this.brokerController.getConsumerManager().getConsumerGroupInfo(requestHeader.getConsumerGroup());
        // ... 判断非空,判断订阅组的配置是非广播且消息类型是广播消息,返回错误
		// 根据topic获取订阅信息
        subscriptionData = consumerGroupInfo.findSubscriptionData(requestHeader.getTopic());
        // ... 判断非空,判断时间戳
        // ... expressionType 不是 tag类型的,从 filter获取 consumerFilterData(里边有topic、customerGroup)
		// ... expressionType 不是 tag类型的,broker配置也不支持属性过滤,返回错误
    if (!ExpressionType.isTagType(subscriptionData.getExpressionType())
        && !this.brokerController.getBrokerConfig().isEnablePropertyFilter()) {
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark("The broker does not support consumer to filter message by " + subscriptionData.getExpressionType());
        return response;
    }

    MessageFilter messageFilter;
    // 根据是否支持filter的重试,构建不同的 messageFilter
    if (this.brokerController.getBrokerConfig().isFilterSupportRetry()) {
        messageFilter = new ExpressionForRetryMessageFilter(subscriptionData, consumerFilterData,
            this.brokerController.getConsumerFilterManager());
    } else {
        messageFilter = new ExpressionMessageFilter(subscriptionData, consumerFilterData,
            this.brokerController.getConsumerFilterManager());
    }
	// 从commitlog中获取message,为空的话,response设置error msg
    final GetMessageResult getMessageResult =
        this.brokerController.getMessageStore().getMessage(requestHeader.getConsumerGroup(), requestHeader.getTopic(),
            requestHeader.getQueueId(), requestHeader.getQueueOffset(), requestHeader.getMaxMsgNums(), messageFilter);
    if (getMessageResult != null) {
        response.setRemark(getMessageResult.getStatus().name());
        responseHeader.setNextBeginOffset(getMessageResult.getNextBeginOffset());
        responseHeader.setMinOffset(getMessageResult.getMinOffset());
        responseHeader.setMaxOffset(getMessageResult.getMaxOffset());
		// 消费过慢,可以从 salve节点拉取消息(配置的从节点brokerId)
        if (getMessageResult.isSuggestPullingFromSlave()) {
responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getWhichBrokerWhenConsumeSlowly());
        } else {
            responseHeader.setSuggestWhichBrokerId(MixAll.MASTER_ID);
        }

        switch (this.brokerController.getMessageStoreConfig().getBrokerRole()) {
            case ASYNC_MASTER:
            case SYNC_MASTER:
                break;
            case SLAVE:
            	// 从节点要有读权限
                if (!this.brokerController.getBrokerConfig().isSlaveReadEnable()) {
                    response.setCode(ResponseCode.PULL_RETRY_IMMEDIATELY);
                    responseHeader.setSuggestWhichBrokerId(MixAll.MASTER_ID);
                }
                break;
        }
		
        if (this.brokerController.getBrokerConfig().isSlaveReadEnable()) {
            // consume too slow ,redirect to another machine
            if (getMessageResult.isSuggestPullingFromSlave()) {
          responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getWhichBrokerWhenConsumeSlowly());
            }
            // consume ok
            else {
                responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getBrokerId());
            }
        } else {
            responseHeader.setSuggestWhichBrokerId(MixAll.MASTER_ID);
        }

        switch (getMessageResult.getStatus()) {
            case FOUND:
                response.setCode(ResponseCode.SUCCESS);
                break;
            // ... 
            // 节点的queue中没有message,queue的offset是否为0
            case NO_MESSAGE_IN_QUEUE:
                if (0 != requestHeader.getQueueOffset()) {
                    response.setCode(ResponseCode.PULL_OFFSET_MOVED);

                    // XXX: warn and notify me
                } else {
                    response.setCode(ResponseCode.PULL_NOT_FOUND);
                }
                break;
            default:
                assert false;
                break;
        }
		// 是否有消费消息的钩子
        if (this.hasConsumeMessageHook()) {
            ConsumeMessageContext context = new ConsumeMessageContext();
            context.setConsumerGroup(requestHeader.getConsumerGroup());
            context.setTopic(requestHeader.getTopic());
            context.setQueueId(requestHeader.getQueueId());

            String owner = request.getExtFields().get(BrokerStatsManager.COMMERCIAL_OWNER);

            switch (response.getCode()) {
            	// ... 不同code 构建不同的 context
            }
			// 回调钩子
            this.executeConsumeMessageHookBefore(context);
        }

        switch (response.getCode()) {
            case ResponseCode.SUCCESS:
            // group.getNums + 1(消费数量)
            this.brokerController.getBrokerStatsManager().incGroupGetNums(requestHeader.getConsumerGroup(), requestHeader.getTopic(),getMessageResult.getMessageCount());
            // group.getSize + 1(buffer总和)
this.brokerController.getBrokerStatsManager().incGroupGetSize(requestHeader.getConsumerGroup(), requestHeader.getTopic(),getMessageResult.getBufferTotalSize());
			// broker..getNums + 1(消费数量)
this.brokerController.getBrokerStatsManager().incBrokerGetNums(getMessageResult.getMessageCount());
				// 默认是消息从内存中获取
                if (this.brokerController.getBrokerConfig().isTransferMsgByHeap()) {
                    final long beginTimeMills = this.brokerController.getMessageStore().now();
                    // 从byteBuffer中获取,记录时间
                    final byte[] r = this.readGetMessageResult(getMessageResult, requestHeader.getConsumerGroup(), requestHeader.getTopic(), requestHeader.getQueueId());
                 // getLatency + 1(延迟消息数量) this.brokerController.getBrokerStatsManager().incGroupGetLatency(requestHeader.getConsumerGroup(),
                        requestHeader.getTopic(), requestHeader.getQueueId(),
                        (int) (this.brokerController.getMessageStore().now() - beginTimeMills));
                    response.setBody(r);
                } else {
                	// 不记录刷盘时间,netty response 写回
                    try {
                        FileRegion fileRegion =
                            new ManyMessageTransfer(response.encodeHeader(getMessageResult.getBufferTotalSize()), getMessageResult);
                        channel.writeAndFlush(fileRegion).addListener(new ChannelFutureListener() {
                            @Override
                            public void operationComplete(ChannelFuture future) throws Exception {
                                getMessageResult.release();
                                if (!future.isSuccess()) {
                                    log.error("transfer many message by pagecache failed, {}", channel.remoteAddress(), future.cause());
                                }
                            }
                        });
                    } catch (Throwable e) {
                        log.error("transfer many message by pagecache exception", e);
                        getMessageResult.release();
                    }

                    response = null;
                }
                break;
            case ResponseCode.PULL_NOT_FOUND:
				// 允许延迟,超时延迟拉取
                if (brokerAllowSuspend && hasSuspendFlag) {
                    long pollingTimeMills = suspendTimeoutMillisLong;
                    if (!this.brokerController.getBrokerConfig().isLongPollingEnable()) {
                        pollingTimeMills = this.brokerController.getBrokerConfig().getShortPollingTimeMills();
                    }

                    String topic = requestHeader.getTopic();
                    long offset = requestHeader.getQueueOffset();
                    int queueId = requestHeader.getQueueId();
                    PullRequest pullRequest = new PullRequest(request, channel, pollingTimeMills,
                        this.brokerController.getMessageStore().now(), offset, subscriptionData, messageFilter);
                    this.brokerController.getPullRequestHoldService().suspendPullRequest(topic, queueId, pullRequest);
                    response = null;
                    break;
                }
			// 立即重试
            case ResponseCode.PULL_RETRY_IMMEDIATELY:
                break;
            case ResponseCode.PULL_OFFSET_MOVED:
            	// 主节点或者从节点开启了offset检测
                if (this.brokerController.getMessageStoreConfig().getBrokerRole() != BrokerRole.SLAVE
                    || this.brokerController.getMessageStoreConfig().isOffsetCheckInSlave()) {
                    // 记录 warn 日志
                    MessageQueue mq = new MessageQueue();
                    mq.setTopic(requestHeader.getTopic());
                    mq.setQueueId(requestHeader.getQueueId());
                    mq.setBrokerName(this.brokerController.getBrokerConfig().getBrokerName());

                    OffsetMovedEvent event = new OffsetMovedEvent();
                    event.setConsumerGroup(requestHeader.getConsumerGroup());
                    event.setMessageQueue(mq);
                    event.setOffsetRequest(requestHeader.getQueueOffset());
                    event.setOffsetNew(getMessageResult.getNextBeginOffset());
                    this.generateOffsetMovedEvent(event);
                    log.warn(
                        "PULL_OFFSET_MOVED:correction offset. topic={}, groupId={}, requestOffset={}, newOffset={}, suggestBrokerId={}",
                        requestHeader.getTopic(), requestHeader.getConsumerGroup(), event.getOffsetRequest(), event.getOffsetNew(),
                        responseHeader.getSuggestWhichBrokerId());
                } else {
                    responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getBrokerId());
                    response.setCode(ResponseCode.PULL_RETRY_IMMEDIATELY);
                    log.warn("PULL_OFFSET_MOVED:none correction. topic={}, groupId={}, requestOffset={}, suggestBrokerId={}",
                        requestHeader.getTopic(), requestHeader.getConsumerGroup(), requestHeader.getQueueOffset(),
                        responseHeader.getSuggestWhichBrokerId());
                }

                break;
            default:
                assert false;
        }
    } else {
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark("store getMessage return null");
    }

    boolean storeOffsetEnable = brokerAllowSuspend;
    storeOffsetEnable = storeOffsetEnable && hasCommitOffsetFlag;
    storeOffsetEnable = storeOffsetEnable
        && this.brokerController.getMessageStoreConfig().getBrokerRole() != BrokerRole.SLAVE;
    // 更新offsetTable(Map:key(group@topic),value(queueId,offset))
    if (storeOffsetEnable) {        this.brokerController.getConsumerOffsetManager().commitOffset(RemotingHelper.parseChannelRemoteAddr(channel), requestHeader.getConsumerGroup(), requestHeader.getTopic(), requestHeader.getQueueId(), requestHeader.getCommitOffset());
    }
    return response;
}
```

QueryMessageProcessor

```
@Override
public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request)
    throws RemotingCommandException {
    // 后台接口
    switch (request.getCode()) {
    		// 条件查询msg(topic、key、maxNum、时间戳)
        case RequestCode.QUERY_MESSAGE:
            return this.queryMessage(ctx, request);
            // 根据offset获取commitlog
        case RequestCode.VIEW_MESSAGE_BY_ID:
            return this.viewMessageById(ctx, request);
        default:
            break;
    }

    return null;
}
```

ReplyMessageProcessor

```
    @Override
    public RemotingCommand processRequest(ChannelHandlerContext ctx,
        RemotingCommand request) throws RemotingCommandException {
        SendMessageContext mqtraceContext = null;
        SendMessageRequestHeader requestHeader = parseRequestHeader(request);
        if (requestHeader == null) {
            return null;
        }
		// 构建 traceContext
        mqtraceContext = buildMsgContext(ctx, requestHeader);
		// 执行hook之前,完善 sendMessageContext 对象
        this.executeSendMessageHookBefore(ctx, request, mqtraceContext);
		// below
        RemotingCommand response = this.processReplyMessageRequest(ctx, request, mqtraceContext, requestHeader);
		// do nothing
        this.executeSendMessageHookAfter(response, mqtraceContext);
        return response;
    }

```

```
private RemotingCommand processReplyMessageRequest(final ChannelHandlerContext ctx,
    final RemotingCommand request,
    final SendMessageContext sendMessageContext,
    final SendMessageRequestHeader requestHeader) {
    final RemotingCommand response = RemotingCommand.createResponseCommand(SendMessageResponseHeader.class);
    final SendMessageResponseHeader responseHeader = (SendMessageResponseHeader) response.readCustomHeader();

    response.setOpaque(request.getOpaque());

    response.addExtField(MessageConst.PROPERTY_MSG_REGION, this.brokerController.getBrokerConfig().getRegionId());
    response.addExtField(MessageConst.PROPERTY_TRACE_SWITCH, String.valueOf(this.brokerController.getBrokerConfig().isTraceOn()));

    log.debug("receive SendReplyMessage request command, {}", request);
    final long startTimstamp = this.brokerController.getBrokerConfig().getStartAcceptSendRequestTimeStamp();
    if (this.brokerController.getMessageStore().now() < startTimstamp) {
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark(String.format("broker unable to service, until %s", UtilAll.timeMillisToHumanString2(startTimstamp)));
        return response;
    }

    response.setCode(-1);
    // 校验topicConfig(如果是重试消息,topicConfig为空则创建)和queueId(大于topicConfig中的读写数量，返回错误)
    super.msgCheck(ctx, requestHeader, response);
    if (response.getCode() != -1) {
        return response;
    }

    final byte[] body = request.getBody();

    int queueIdInt = requestHeader.getQueueId();
    TopicConfig topicConfig = this.brokerController.getTopicConfigManager().selectTopicConfig(requestHeader.getTopic());

    if (queueIdInt < 0) {
        queueIdInt = Math.abs(this.random.nextInt() % 99999999) % topicConfig.getWriteQueueNums();
    }
	// 构建channel msg
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
	// 完善header，同步到broker的client
    PushReplyResult pushReplyResult = this.pushReplyMessage(ctx, requestHeader, msgInner);
    this.handlePushReplyResult(pushReplyResult, response, responseHeader, queueIdInt);
	// 保存返回值
    if (this.brokerController.getBrokerConfig().isStoreReplyMessageEnable()) {
        PutMessageResult putMessageResult = this.brokerController.getMessageStore().putMessage(msgInner);
        this.handlePutMessageResult(putMessageResult, request, msgInner, responseHeader, sendMessageContext, queueIdInt);
    }

    return response;
}
```

SendMessageProcessor

```
    public CompletableFuture<RemotingCommand> asyncProcessRequest(ChannelHandlerContext ctx,
RemotingCommand request) throws RemotingCommandException {
        final SendMessageContext mqtraceContext;
        switch (request.getCode()) {
        	// 消费者的回复消息
            case RequestCode.CONSUMER_SEND_MSG_BACK:
                return this.asyncConsumerSendMsgBack(ctx, request);
            default:
                SendMessageRequestHeader requestHeader = parseRequestHeader(request);
                if (requestHeader == null) {
                    return CompletableFuture.completedFuture(null);
                }
                mqtraceContext = buildMsgContext(ctx, requestHeader);
                this.executeSendMessageHookBefore(ctx, request, mqtraceContext);
                if (requestHeader.isBatch()) {
                    return this.asyncSendBatchMessage(ctx, request, mqtraceContext, requestHeader);
                } else {
                	// 和batch相比,多个同步和异步刷盘还有事务的判断
                    return this.asyncSendMessage(ctx, request, mqtraceContext, requestHeader);
                }
        }
    }
```

asyncConsumerSendMsgBack

```
private CompletableFuture<RemotingCommand> asyncConsumerSendMsgBack(ChannelHandlerContext ctx,
                                                                    RemotingCommand request) throws RemotingCommandException {
    final RemotingCommand response = RemotingCommand.createResponseCommand(null);
    final ConsumerSendMsgBackRequestHeader requestHeader =
            (ConsumerSendMsgBackRequestHeader)request.decodeCommandCustomHeader(ConsumerSendMsgBackRequestHeader.class);
    String namespace = NamespaceUtil.getNamespaceFromResource(requestHeader.getGroup());
    // 回调钩子之后处理,完善context对象
    if (this.hasConsumeMessageHook() && !UtilAll.isBlank(requestHeader.getOriginMsgId())) {
        ConsumeMessageContext context = buildConsumeMessageContext(namespace, requestHeader, request);
        this.executeConsumeMessageHookAfter(context);
    }
    SubscriptionGroupConfig subscriptionGroupConfig =
    // 订阅组关系和一些校验(非空、写权限) this.brokerController.getSubscriptionGroupManager().findSubscriptionGroupConfig(requestHeader.getGroup());
    if (null == subscriptionGroupConfig) {
        response.setCode(ResponseCode.SUBSCRIPTION_GROUP_NOT_EXIST);
        response.setRemark("subscription group not exist, " + requestHeader.getGroup() + " "
            + FAQUrl.suggestTodo(FAQUrl.SUBSCRIPTION_GROUP_NOT_EXIST));
        return CompletableFuture.completedFuture(response);
    }
    if (!PermName.isWriteable(this.brokerController.getBrokerConfig().getBrokerPermission())) {
        response.setCode(ResponseCode.NO_PERMISSION);
        response.setRemark("the broker[" + this.brokerController.getBrokerConfig().getBrokerIP1() + "] sending message is forbidden");
        return CompletableFuture.completedFuture(response);
    }
	// 无需重试，返回成功
    if (subscriptionGroupConfig.getRetryQueueNums() <= 0) {
        response.setCode(ResponseCode.SUCCESS);
        response.setRemark(null);
        return CompletableFuture.completedFuture(response);
    }
	// 需要重试,构建新topic
    String newTopic = MixAll.getRetryTopic(requestHeader.getGroup());
    int queueIdInt = Math.abs(this.random.nextInt() % 99999999) % subscriptionGroupConfig.getRetryQueueNums();
    int topicSysFlag = 0;
    if (requestHeader.isUnitMode()) {
        topicSysFlag = TopicSysFlag.buildSysFlag(false, true);
    }
	// 创建topicConfig
    TopicConfig topicConfig = this.brokerController.getTopicConfigManager().createTopicInSendMessageBackMethod(
        newTopic,
        subscriptionGroupConfig.getRetryQueueNums(),
        PermName.PERM_WRITE | PermName.PERM_READ, topicSysFlag);
    // 校验(非空、写权限)
    if (null == topicConfig) {
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark("topic[" + newTopic + "] not exist");
        return CompletableFuture.completedFuture(response);
    }

    if (!PermName.isWriteable(topicConfig.getPerm())) {
        response.setCode(ResponseCode.NO_PERMISSION);
        response.setRemark(String.format("the topic[%s] sending message is forbidden", newTopic));
        return CompletableFuture.completedFuture(response);
    }
    // 根据offset获取msg
    MessageExt msgExt = this.brokerController.getMessageStore().lookMessageByOffset(requestHeader.getOffset());
    if (null == msgExt) {
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark("look message by offset failed, " + requestHeader.getOffset());
        return CompletableFuture.completedFuture(response);
    }
	// retryTopic 为空的话,给默认的
    final String retryTopic = msgExt.getProperty(MessageConst.PROPERTY_RETRY_TOPIC);
    if (null == retryTopic) {
        MessageAccessor.putProperty(msgExt, MessageConst.PROPERTY_RETRY_TOPIC, msgExt.getTopic());
    }
    msgExt.setWaitStoreMsgOK(false);
	// 延迟等级
    int delayLevel = requestHeader.getDelayLevel();
	// 客户端版本<3.4.9,最大重试次数取request的
    int maxReconsumeTimes = subscriptionGroupConfig.getRetryMaxTimes();
    if (request.getVersion() >= MQVersion.Version.V3_4_9.ordinal()) {
        Integer times = requestHeader.getMaxReconsumeTimes();
        if (times != null) {
            maxReconsumeTimes = times;
        }
    }
	// 重试次数 > 最大次数 或 非延时
    if (msgExt.getReconsumeTimes() >= maxReconsumeTimes 
        || delayLevel < 0) {
        // 进入前缀为死信的topic
        newTopic = MixAll.getDLQTopic(requestHeader.getGroup());
        queueIdInt = Math.abs(this.random.nextInt() % 99999999) % DLQ_NUMS_PER_GROUP;

        topicConfig = this.brokerController.getTopicConfigManager().createTopicInSendMessageBackMethod(newTopic,
                DLQ_NUMS_PER_GROUP,
                PermName.PERM_WRITE, 0);
        if (null == topicConfig) {
            response.setCode(ResponseCode.SYSTEM_ERROR);
            response.setRemark("topic[" + newTopic + "] not exist");
            return CompletableFuture.completedFuture(response);
        }
    } else {
    	// 延时为0的时候，延迟级别=次数+3(1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h )
        if (0 == delayLevel) {
            delayLevel = 3 + msgExt.getReconsumeTimes();
        }
        msgExt.setDelayTimeLevel(delayLevel);
    }
	// 构建store msg
    MessageExtBrokerInner msgInner = new MessageExtBrokerInner();
    msgInner.setTopic(newTopic);
    msgInner.setBody(msgExt.getBody());
    msgInner.setFlag(msgExt.getFlag());
    MessageAccessor.setProperties(msgInner, msgExt.getProperties());
    msgInner.setPropertiesString(MessageDecoder.messageProperties2String(msgExt.getProperties()));
    msgInner.setTagsCode(MessageExtBrokerInner.tagsString2tagsCode(null, msgExt.getTags()));

    msgInner.setQueueId(queueIdInt);
    msgInner.setSysFlag(msgExt.getSysFlag());
    msgInner.setBornTimestamp(msgExt.getBornTimestamp());
    msgInner.setBornHost(msgExt.getBornHost());
    msgInner.setStoreHost(msgExt.getStoreHost());
    msgInner.setReconsumeTimes(msgExt.getReconsumeTimes() + 1);

    String originMsgId = MessageAccessor.getOriginMessageId(msgExt);
    MessageAccessor.setOriginMessageId(msgInner, UtilAll.isBlank(originMsgId) ? msgExt.getMsgId() : originMsgId);
    msgInner.setPropertiesString(MessageDecoder.messageProperties2String(msgExt.getProperties()));
	// 存入commitlog，并更新最后操作时间
    CompletableFuture<PutMessageResult> putMessageResult = this.brokerController.getMessageStore().asyncPutMessage(msgInner);
    return putMessageResult.thenApply((r) -> {
        if (r != null) {
            switch (r.getPutMessageStatus()) {
                case PUT_OK:
                    String backTopic = msgExt.getTopic();
                    String correctTopic = msgExt.getProperty(MessageConst.PROPERTY_RETRY_TOPIC);
                    if (correctTopic != null) {
                        backTopic = correctTopic;
                    }
                    // 消费次数 + 1
                    this.brokerController.getBrokerStatsManager().incSendBackNums(requestHeader.getGroup(), backTopic);
                    response.setCode(ResponseCode.SUCCESS);
                    response.setRemark(null);
                    return response;
                default:
                    break;
            }
            response.setCode(ResponseCode.SYSTEM_ERROR);
            response.setRemark(r.getPutMessageStatus().name());
            return response;
        }
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark("putMessageResult is null");
        return response;
    });
}
```

asyncSendBatchMessage

```
private CompletableFuture<RemotingCommand> asyncSendBatchMessage(ChannelHandlerContext ctx, RemotingCommand request,
                                                                 SendMessageContext mqtraceContext,
                                                                 SendMessageRequestHeader requestHeader) {
    final RemotingCommand response = preSend(ctx, request, requestHeader);
    final SendMessageResponseHeader responseHeader = (SendMessageResponseHeader)response.readCustomHeader();

    if (response.getCode() != -1) {
        return CompletableFuture.completedFuture(response);
    }

    int queueIdInt = requestHeader.getQueueId();
    // 获取topic配置
    TopicConfig topicConfig = this.brokerController.getTopicConfigManager().selectTopicConfig(requestHeader.getTopic());

    if (queueIdInt < 0) {
        queueIdInt = randomQueueId(topicConfig.getWriteQueueNums());
    }

    if (requestHeader.getTopic().length() > Byte.MAX_VALUE) {
        response.setCode(ResponseCode.MESSAGE_ILLEGAL);
        response.setRemark("message topic length too long " + requestHeader.getTopic().length());
        return CompletableFuture.completedFuture(response);
    }
	// 构建批量message
    MessageExtBatch messageExtBatch = new MessageExtBatch();
    messageExtBatch.setTopic(requestHeader.getTopic());
    messageExtBatch.setQueueId(queueIdInt);

    int sysFlag = requestHeader.getSysFlag();
    if (TopicFilterType.MULTI_TAG == topicConfig.getTopicFilterType()) {
        sysFlag |= MessageSysFlag.MULTI_TAGS_FLAG;
    }
    messageExtBatch.setSysFlag(sysFlag);

    messageExtBatch.setFlag(requestHeader.getFlag());
    MessageAccessor.setProperties(messageExtBatch, MessageDecoder.string2messageProperties(requestHeader.getProperties()));
    messageExtBatch.setBody(request.getBody());
    messageExtBatch.setBornTimestamp(requestHeader.getBornTimestamp());
    messageExtBatch.setBornHost(ctx.channel().remoteAddress());
    messageExtBatch.setStoreHost(this.getStoreHost());
    messageExtBatch.setReconsumeTimes(requestHeader.getReconsumeTimes() == null ? 0 : requestHeader.getReconsumeTimes());
    String clusterName = this.brokerController.getBrokerConfig().getBrokerClusterName();
    MessageAccessor.putProperty(messageExtBatch, MessageConst.PROPERTY_CLUSTER, clusterName);
	// 刷盘
    CompletableFuture<PutMessageResult> putMessageResult = this.brokerController.getMessageStore().asyncPutMessages(messageExtBatch);
    // 处理刷盘结果
    return handlePutMessageResultFuture(putMessageResult, response, request, messageExtBatch, responseHeader, mqtraceContext, ctx, queueIdInt);
}
```

handlePutMessageResultFuture —> handlePutMessageResult

```
private RemotingCommand handlePutMessageResult(PutMessageResult putMessageResult, RemotingCommand response,
                                               RemotingCommand request, MessageExt msg,
                                               SendMessageResponseHeader responseHeader, SendMessageContext sendMessageContext, ChannelHandlerContext ctx,
                                               int queueIdInt) {
    if (putMessageResult == null) {
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark("store putMessage return null");
        return response;
    }
    boolean sendOK = false;

    switch (putMessageResult.getPutMessageStatus()) {
        // Success
        case PUT_OK:
            sendOK = true;
            response.setCode(ResponseCode.SUCCESS);
            break;
		// ...
        // Failed
        case CREATE_MAPEDFILE_FAILED:
            response.setCode(ResponseCode.SYSTEM_ERROR);
            response.setRemark("create mapped file failed, server is busy or broken.");
            break;
        // ...    
        default:
            response.setCode(ResponseCode.SYSTEM_ERROR);
            response.setRemark("UNKNOWN_ERROR DEFAULT");
            break;
    }

    String owner = request.getExtFields().get(BrokerStatsManager.COMMERCIAL_OWNER);
    // 发送成功
    if (sendOK) {
		// 计数,topic刷盘数量
        this.brokerController.getBrokerStatsManager().incTopicPutNums(msg.getTopic(), putMessageResult.getAppendMessageResult().getMsgNum(), 1);
        // 计数,topic刷盘数据大小
        this.brokerController.getBrokerStatsManager().incTopicPutSize(msg.getTopic(),
            putMessageResult.getAppendMessageResult().getWroteBytes());
        // 计数,broker刷盘数量this.brokerController.getBrokerStatsManager().incBrokerPutNums(putMessageResult.getAppendMessageResult().getMsgNum());

        response.setRemark(null);

        responseHeader.setMsgId(putMessageResult.getAppendMessageResult().getMsgId());
        responseHeader.setQueueId(queueIdInt);
        responseHeader.setQueueOffset(putMessageResult.getAppendMessageResult().getLogicsOffset());
		// response写回
        doResponse(ctx, request, response);
		// 有钩子的话,构建 sendMessageContext 对象
        if (hasSendMessageHook()) {
            sendMessageContext.setMsgId(responseHeader.getMsgId());
            sendMessageContext.setQueueId(responseHeader.getQueueId());
            sendMessageContext.setQueueOffset(responseHeader.getQueueOffset());

            int commercialBaseCount = brokerController.getBrokerConfig().getCommercialBaseCount();
            int wroteSize = putMessageResult.getAppendMessageResult().getWroteBytes();
            int incValue = (int)Math.ceil(wroteSize / BrokerStatsManager.SIZE_PER_COUNT) * commercialBaseCount;

            sendMessageContext.setCommercialSendStats(BrokerStatsManager.StatsType.SEND_SUCCESS);
            sendMessageContext.setCommercialSendTimes(incValue);
            sendMessageContext.setCommercialSendSize(wroteSize);
            sendMessageContext.setCommercialOwner(owner);
        }
        return null;
    } else {
    	// 发送失败,有钩子的话,构建 sendMessageContext 对象,状态是发送失败
        if (hasSendMessageHook()) {
            int wroteSize = request.getBody().length;
            int incValue = (int)Math.ceil(wroteSize / BrokerStatsManager.SIZE_PER_COUNT);

            sendMessageContext.setCommercialSendStats(BrokerStatsManager.StatsType.SEND_FAILURE);
            sendMessageContext.setCommercialSendTimes(incValue);
            sendMessageContext.setCommercialSendSize(wroteSize);
            sendMessageContext.setCommercialOwner(owner);
        }
    }
    return response;
}
```



## 小结 

**PullMessageProcessor**：

先看下方法的流转，broker接收到消息会触发监听，同时通知 PullMessageProcessor.processRequest() 对请求进行处理

DefaultMessageStore#ReputMessageService.doReput() —> NotifyMessageArrivingListener.arriving() —>

pullRequestHoldService.notifyMessageArriving() —> PullMessageProcessor.executeRequestWhenWakeup()

获取订阅关系和topic配置，必要的判断。

获取消费标记，产生 subscriptionData 和 consumerFilterData。做一些版本和tag类型的判断。

根据messageFilter的配置，初始化消息过滤。

消费过慢,可以从 salve节点拉取消息(配置的从节点brokerId)。从节点要有权限。

是否有消费消息的钩子，回调钩子。

拉取成功：更新组的消息数量和buffer的大小。默认从内存中获取消息,记录时间并netty返回。

拉取失败：允许延迟的话，调用延迟拉取。

offset移动：记录warn日志。

更新offsetTable(Map:key(group@topic),value(queueId,offset))。

**QueryMessageProcessor**：

后台接口，条件查询msg(topic、key、maxNum、时间戳)。

后台接口，根据offset获取commitlog。

**ReplyMessageProcessor**：

DefaultMQProducerImpl.sendMessage(SEND_REPLY_MESSAGE/SEND_REPLY_MESSAGE_V2) —>

ReplyMessageProcessor.processRequest()

构建 traceContext

执行hook之前,完善 sendMessageContext 对象

校验topicConfig(如果是重试消息,topicConfig为空则创建)和queueId(大于topicConfig中的读写数量，返回错误)

建channel msg

完善header，同步到broker的client

保存返回值

**SendMessageProcessor**：

DefaultMQProducerImpl.sendMessage(SEND_MESSAGE/SEND_MESSAGE_V2/SEND_BATCH_MESSAGE) / DefaultMQPullConsumerImpl/DefaultMQPushConsumerImpl.consumerSendMessageBack  ——>

SendMessageProcessor.processRequest ()

消费的回复消息：

做校验和判断，定义重试topic

重试次数 > 最大次数 或 非延时，创建死信的topic

延时为0的时候，延迟级别=次数+3

构建store msg 存入 commitlog,更新消费成功数量 +1

正常异步发送消息单条/批量:

单条：同步/异步刷盘、事务的话构建半消息

批量：刷盘，计数，topic刷盘数量/数据大小，broker刷盘数量。构建回调钩子的sendMessageContext，并标记状态。

