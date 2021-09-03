# rocketmq 初探（四）

大家好，我是烤鸭：

&nbsp;&nbsp;&nbsp;&nbsp;上一篇简单介绍broker的初始化，这一篇介绍 NettyRequestProcessor 的实现(主要是broker里用到的)。

AdminBrokerProcessor、ClientManageProcessor、ConsumerManageProcessor、EndTransactionProcessor

## NettyRequestProcessor

```
/**
 * Common remoting command processor
 */
public interface NettyRequestProcessor {
    RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request)
        throws Exception;

    boolean rejectRequest();

}
```

先看下哪些是broker里用到的，从包名就能看出来（remoting包下的后边看），接下来一个一个分析。

![1](.\1.png)

AdminBrokerProcessor (后台发起的 CURD)

```
@Override
public RemotingCommand processRequest(ChannelHandlerContext ctx,
    RemotingCommand request) throws RemotingCommandException {
    switch (request.getCode()) {
        case RequestCode.UPDATE_AND_CREATE_TOPIC:
            return this.updateAndCreateTopic(ctx, request);
            //...
        default:
            break;
    }

    return null;
}
```

ClientManageProcessor

```
@Override
public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request)
    throws RemotingCommandException {
    switch (request.getCode()) {
        case RequestCode.HEART_BEAT:
        	// 心跳,针对consumer,如果心跳的信息有更新(group和subscribe),会update
            return this.heartBeat(ctx, request);
        case RequestCode.UNREGISTER_CLIENT:
        	// 下线,针对 producer和consumer, producer和consumer 下线
            return this.unregisterClient(ctx, request);
        case RequestCode.CHECK_CLIENT_CONFIG:
        	// 通过注册的filter校验client的config
            return this.checkClientConfig(ctx, request);
        default:
            break;
    }
    return null;
}
```

ConsumerManageProcessor

```
@Override
public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request)
    throws RemotingCommandException {
    switch (request.getCode()) {
        case RequestCode.GET_CONSUMER_LIST_BY_GROUP:
        	// 根据group获取customer的list
            return this.getConsumerListByGroup(ctx, request);
        case RequestCode.UPDATE_CONSUMER_OFFSET:
        	// 更新topic@group对应的queueId和offset(没有就直接put,已存在的话如果要更新的offset<本地offset,记录日志)
            return this.updateConsumerOffset(ctx, request);
        case RequestCode.QUERY_CONSUMER_OFFSET:
        	// 根据topic@group、queueId获取offset(如果offset<0,并且在磁盘没有刷到内存,返回0,否则返回 QUERY_NOT_FOUND)
            return this.queryConsumerOffset(ctx, request);
        default:
            break;
    }
    return null;
}
```

EndTransactionProcessor

```
@Override
public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request) throws
    RemotingCommandException {
    final RemotingCommand response = RemotingCommand.createResponseCommand(null);
    final EndTransactionRequestHeader requestHeader =
        (EndTransactionRequestHeader)request.decodeCommandCustomHeader(EndTransactionRequestHeader.class);
    LOGGER.debug("Transaction request:{}", requestHeader);
    // salve 不支持事务
    if (BrokerRole.SLAVE == brokerController.getMessageStoreConfig().getBrokerRole()) {
        response.setCode(ResponseCode.SLAVE_NOT_AVAILABLE);
        LOGGER.warn("Message store is slave mode, so end transaction is forbidden. ");
        return response;
    }
	// 来源于事务检查(ClientRemotingProcessor.processRequest)
    if (requestHeader.getFromTransactionCheck()) {
        switch (requestHeader.getCommitOrRollback()) {
        	// 非事务类型,直接返回
            case MessageSysFlag.TRANSACTION_NOT_TYPE: {
                LOGGER.warn("Check producer[{}] transaction state, but it's pending status."
                        + "RequestHeader: {} Remark: {}",
                    RemotingHelper.parseChannelRemoteAddr(ctx.channel()),
                    requestHeader.toString(),
                    request.getRemark());
                return null;
            }
			// commit, break 执行执行
            case MessageSysFlag.TRANSACTION_COMMIT_TYPE: {
                LOGGER.warn("Check producer[{}] transaction state, the producer commit the message."
                        + "RequestHeader: {} Remark: {}",
                    RemotingHelper.parseChannelRemoteAddr(ctx.channel()),
                    requestHeader.toString(),
                    request.getRemark());

                break;
            }

            case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE: {
                LOGGER.warn("Check producer[{}] transaction state, the producer rollback the message."
                        + "RequestHeader: {} Remark: {}",
                    RemotingHelper.parseChannelRemoteAddr(ctx.channel()),
                    requestHeader.toString(),
                    request.getRemark());
                break;
            }
            default:
                return null;
        }
    } else {
        switch (requestHeader.getCommitOrRollback()) {
            case MessageSysFlag.TRANSACTION_NOT_TYPE: {
                LOGGER.warn("The producer[{}] end transaction in sending message,  and it's pending status."
                        + "RequestHeader: {} Remark: {}",
                    RemotingHelper.parseChannelRemoteAddr(ctx.channel()),
                    requestHeader.toString(),
                    request.getRemark());
                return null;
            }
			
            case MessageSysFlag.TRANSACTION_COMMIT_TYPE: {
                break;
            }

            case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE: {
                LOGGER.warn("The producer[{}] end transaction in sending message, rollback the message."
                        + "RequestHeader: {} Remark: {}",
                    RemotingHelper.parseChannelRemoteAddr(ctx.channel()),
                    requestHeader.toString(),
                    request.getRemark());
                break;
            }
            default:
                return null;
        }
    }
    OperationResult result = new OperationResult();
    // 提交
    if (MessageSysFlag.TRANSACTION_COMMIT_TYPE == requestHeader.getCommitOrRollback()) {
    	// 根据offset从commitlog中获取当前的 halfmessage(半消息)
        result = this.brokerController.getTransactionalMessageService().commitMessage(requestHeader);
        if (result.getResponseCode() == ResponseCode.SUCCESS) {
        	// 校验半消息的合法性,group、事务队列的offset、commitlog的offset是否一致
            RemotingCommand res = checkPrepareMessage(result.getPrepareMessage(), requestHeader);
            if (res.getCode() == ResponseCode.SUCCESS) {
            	// 根据半消息构建一条事务消息
                MessageExtBrokerInner msgInner = endMessageTransaction(result.getPrepareMessage());
                msgInner.setSysFlag(MessageSysFlag.resetTransactionValue(msgInner.getSysFlag(), requestHeader.getCommitOrRollback()));
                msgInner.setQueueOffset(requestHeader.getTranStateTableOffset());
                msgInner.setPreparedTransactionOffset(requestHeader.getCommitLogOffset());
                msgInner.setStoreTimestamp(result.getPrepareMessage().getStoreTimestamp());
                MessageAccessor.clearProperty(msgInner, MessageConst.PROPERTY_TRANSACTION_PREPARED);
                // 发送最终消息,消息刷到commitlog
                RemotingCommand sendResult = sendFinalMessage(msgInner);
                if (sendResult.getCode() == ResponseCode.SUCCESS) {
                   // 最终消息发送成功,在commitlog里原msg打一个'd'的标记this.brokerController.getTransactionalMessageService().deletePrepareMessage(result.getPrepareMessage());
                }
                return sendResult;
            }
            return res;
        }
        // 回滚
    } else if (MessageSysFlag.TRANSACTION_ROLLBACK_TYPE == requestHeader.getCommitOrRollback()) {
    	// 流程同提交差不多，半小时获取成功之后，在commitlog里原msg打一个'd'的标记
        result = this.brokerController.getTransactionalMessageService().rollbackMessage(requestHeader);
        if (result.getResponseCode() == ResponseCode.SUCCESS) {
            RemotingCommand res = checkPrepareMessage(result.getPrepareMessage(), requestHeader);
            if (res.getCode() == ResponseCode.SUCCESS) {
                this.brokerController.getTransactionalMessageService().deletePrepareMessage(result.getPrepareMessage());
            }
            return res;
        }
    }
    response.setCode(result.getResponseCode());
    response.setRemark(result.getResponseRemark());
    return response;
}
```

ForwardRequestProcessor

```
@Override
public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request) {
    return null;
}
```



## 小结 

介绍了4个 processor：

AdminBrokerProcessor：给console台提供的 curd接口

ClientManageProcessor：针对client端的请求，包含 心跳(上线)、下线、配置检查

ConsumerManageProcessor：针对consumer的，包含 获取consumer列表、更新queueId和offset、获取offset

EndTransactionProcessor：提交模式，从commitlog根据offset获取半消息，写入commitlog，原半消息打一个'd'的标记。

回滚模式，从commitlog根据offset获取半消息，原半消息打一个'd'的标记。

