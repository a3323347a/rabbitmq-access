接这篇

在上文中，主要实现了可靠模式的consumer。而可靠模式的sender实现的相对简略，主要通过rabbitTemplate来完成。
 本以为这样的实现基本是没有问题的。但是前段时间做了一个性能压力测试，但是发现在使用rabbitTemplate时，会有一定的**丢数据**问题。

当时的场景是用30个线程，无间隔的向rabbitmq发送数据，但是当运行一段时间后发现，会出现一些connection closed错误，rabbitTemplate虽然进行了自动重连，但是在重连的过程中，丢失了一部分数据。当时发送了300万条数据，丢失在2000条左右。
 这种丢失率，对于一些对一致性要求很高的应用(比如扣款，转账)来说，是不可接受的。

在google了很久之后，在stackoverflow上找到rabbitTemplate作者对于这种问题的解决方案，他给的方案很简单，单纯的增加connection数：

```java
connectionFactory.setChannelCacheSize(100);
```

修改之后，确实不再出现connection closed这种错误了，在发送了3000万条数据后，一条都没有丢失。
 似乎问题已经完美的解决了，但是我又想到一个问题：当我们的网络在发生抖动时，这种方式还是不是安全的？
 换句话说，如果我强制切断客户端和rabbitmq服务端的连接，数据还会丢失吗？

为了验证这种场景，我重新发送300万条数据，在发送过程中，在rabbitmq的管理界面上点击强制关闭连接：

<img src="rabbitmq可靠发送的自动重试机制.assets/关闭MQ.png" style="zoom:50%;" />

然后发现，仍然存在**丢失数据**的问题。

看来这个问题，没有想象中的那么简单了。

在阅读了部分rabbitTemplate的代码之后发现：
 1 rabbitTemplate的ack确认机制是异步的
 2 这种确认机制是一种事后发现机制，并不能同步的发现问题
 也就是说，即便打开了

```java
connectionFactory.setPublisherConfirms(true);
rabbitTemplate.setMandatory(true);
```

并且实现了：

```cpp
rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
            if (!ack) {
                log.info("send message failed: " + cause + correlationData.toString());
            } 
        });
```

依旧是**不安全**的。
 rabbitTemplate的发送流程是这样的：
 1 发送数据并返回(不确认rabbitmq服务器已成功接收)
 2 异步的接收从rabbitmq返回的ack确认信息
 3 收到ack后调用confirmCallback函数
 注意：**在confirmCallback中是没有原message的，所以无法在这个函数中调用重发，confirmCallback只有一个通知的作用**

在这种情况下，如果在2，3步中任何时候切断连接，我们都无法确认数据是否真的已经成功发送出去，从而造成数据丢失的问题。

最完美的解决方案只有1种：
 使用rabbitmq的**事务**机制。
 但是在这种情况下，rabbitmq的效率极低，每秒钟处理的message在几百条左右。实在不可取。
 第二种解决方式，使用同步的发送机制，也就是说，客户端发送数据，rabbitmq收到后返回ack，再收到ack后，send函数才返回。代码类似这样：

```bash
创建channel
send message
wait for ack(or 超时)
close channel
返回成功or失败
```

同样的，由于每次发送message都要重新建立连接，效率很低。

基于上面的分析，我们使用一种新的方式来做到数据的不丢失。
 在rabbitTemplate异步确认的基础上
 1 在本地缓存已发送的message
 2 通过confirmCallback或者被确认的ack，将被确认的message从本地删除
 3 定时扫描本地的message，如果大于一定时间未被确认，则重发

当然了，这种解决方式也有一定的**问题**：
 想象这种场景，rabbitmq接收到了消息，在发送ack确认时，网络断了，造成客户端没有收到ack，重发消息。（相比于丢失消息，重发消息要好解决的多，我们可以在consumer端做到幂等）。
 自动重试的代码如下：



```cpp
public class RetryCache {
    private MessageSender sender;
    private boolean stop = false;
    private Map<String, MessageWithTime> map = new ConcurrentHashMap<>();
    private AtomicLong id = new AtomicLong();

    @NoArgsConstructor
    @AllArgsConstructor
    @Data
    private static class MessageWithTime {
        long time;
        Object message;
    }

    public void setSender(MessageSender sender) {
        this.sender = sender;
        startRetry();
    }

    public String generateId() {
        return "" + id.incrementAndGet();
    }

    public void add(String id, Object message) {
        map.put(id, new MessageWithTime(System.currentTimeMillis(), message));
    }

    public void del(String id) {
        map.remove(id);
    }

    private void startRetry() {
        new Thread(() ->{
            while (!stop) {
                try {
                    Thread.sleep(Constants.RETRY_TIME_INTERVAL);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                long now = System.currentTimeMillis();

                for (String key : map.keySet()) {
                    MessageWithTime messageWithTime = map.get(key);

                    if (null != messageWithTime) {
                        if (messageWithTime.getTime() + 3 * Constants.VALID_TIME < now) {
                            log.info("send message failed after 3 min " + messageWithTime);
                            del(key);
                        } else if (messageWithTime.getTime() + Constants.VALID_TIME < now) {
                            DetailRes detailRes = sender.send(messageWithTime.getMessage());

                            if (detailRes.isSuccess()) {
                                del(key);
                            }
                        }
                    }
                }
            }
        }).start();
    }
}
```

在client端发送之前，先在本地缓存message，代码如下：



```tsx
@Override
public DetailRes send(Object message) {
    try {
        String id = retryCache.generateId();
        retryCache.add(id, message);
        rabbitTemplate.correlationConvertAndSend(message, new CorrelationData(id));
    } catch (Exception e) {
        return new DetailRes(false, "");
    }

    return new DetailRes(true, "");
}
```

在收到ack时删除本地缓存，代码如下：



```cpp
rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
    if (!ack) {
        log.info("send message failed: " + cause + correlationData.toString());
    } else {
        retryCache.del(correlationData.getId());
    }
});
```

再次验证刚才的场景，发送300w条数据，在发送的过程中过一段时间close一次connection，发送结束后，实际发送数据301.2w条，有一些重复，但是没有丢失数据。
 同时需要验证本地缓存的**内存泄露**问题，程序连续发送1.5亿条数据，内存占用稳定在900M，并**没有**明显的波动。

最后贴一下rabbitmq的性能测试数据：
 1、300w条1k的数据，单机部署rabbitmq(8核，32G)
 在ack确认模式下平均发送效率为1.1w条/秒
 非ack确认模式下平均发送效率为1.6w条/秒

2、300w条1k的数据，cluster模式部署3台(8核*3， 32G*3）
 在ack确认模式下平均发送效率为1.3w条/秒
 非ack确认模型下平均发送效率为1.7w条/秒

3、300w条1k的数据，单机部署rabbitmq(8核，32G)
 在ack确认模式下平均消费效率为9000条/秒

4、300w条1k的数据，cluster模式部署3台(8核*3， 32G*3）
 在ack确认模式下平均消费效率为1w条/秒