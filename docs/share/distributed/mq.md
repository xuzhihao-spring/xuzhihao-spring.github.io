# 消息队列

## RocketMQ

Zookeeper集群分布式、主从,支持顺序，延时，事务消息，支持查询重试

RocketMQ支持两种消息模式:
- 广播消费: 每个消费者实例都会收到消息,也就是一条消息可以被每个消费者实例处理；
- 集群消费: 一条消息只能被一个消费者实例消费

### 1. 消息类型

#### 1.1 可靠同步发送

同步发送是指消息发送方发出数据后，会在收到接收方发回响应之后才发下一个数据包的通讯方式。

此种方式应用场景非常广泛，例如重要通知邮件、报名短信通知、营销短信系统等

```java
@RequestMapping("/orderSyncSend/{pid}")
	public String orderSyncSend(@PathVariable("pid") Integer pid) {

		Order order = new Order();
		order.setUid(1);
		order.setUsername("测试用户1");
		order.setPid(pid);
		order.setPname("潜水望远镜");
		order.setPprice(698.50);
		order.setNumber(1);
		// 向mq中投递一个下单成功的消息
		// 参数一: 指定topic
		// 参数二: 指定消息体
		rocketMQTemplate.convertAndSend("order-topic", order);
		return System.currentTimeMillis() + "->" + pid;
	}
```

#### 1.2 可靠异步发送

异步发送是指发送方发出数据后，不等接收方发回响应，接着发送下个数据包的通讯方式。发送方通过回调接口接收服务器响应，并对响应结果进行处理。

异步发送一般用于链路耗时较长，对RT响应时间较为敏感的业务场景，例如用户视频上传后通知启动转码服务，转码完成后通知推送转码结果等。

```java
// 异步消息
	@RequestMapping("/orderAsyncSend/{pid}")
	public String orderAsyncSend(@PathVariable("pid") Integer pid) {
		Order order = new Order();
		order.setUid(1);
		order.setUsername("测试用户2");
		order.setPid(pid);
		order.setPname("超人衬衫");
		order.setPprice(280.00);
		order.setNumber(1);
		// 参数一: topic, 如果想添加tag 可以使用"topic:tag"的写法
		// 参数二: 消息内容
		// 参数三: 回调函数, 处理返回结果
		rocketMQTemplate.asyncSend("order-topic", order, new SendCallback() {
			@Override
			public void onSuccess(SendResult sendResult) {
				System.out.println(sendResult);
			}

			@Override
			public void onException(Throwable throwable) {
				System.out.println(throwable);
			}
		});
		return System.currentTimeMillis() + "->" + pid;
	}
```

#### 1.3 单向发送

单向发送是指发送方只负责发送消息，不等待服务器回应且没有回调函数触发，即只发送请求不等待应答。

适用于某些耗时非常短，但对可靠性要求并不高的场景，例如日志收集。

```java
// 单向消息
	@RequestMapping("/orderOneWay/{pid}")
	public String orderOneWay(@PathVariable("pid") Integer pid) {

		// 参数一: topic, 如果想添加tag 可以使用"topic:tag"的写法
		// 参数二: 消息内容
		// 参数三: 回调函数, 处理返回结果
		for (int i = 0; i < 10; i++) {
			Order order = new Order();
			order.setUid(i);
			order.setUsername("测试用户3");
			order.setPid(pid);
			order.setPname("肉包子");
			order.setPprice(2.50);
			order.setNumber(1);
			rocketMQTemplate.sendOneWay("order-topic", order);
		}
		return System.currentTimeMillis() + "->" + pid;
	}
```

#### 1.3 单向顺序发送

```java
// 单向顺序消息
	@RequestMapping("/OneWayOrderly/{pid}")
	public void testOneWayOrderly(@PathVariable("pid") Integer pid) {
		for (int i = 0; i < 10; i++) {
			// 第三个参数的作用是用来决定这些消息发送到哪个队列的上的
			Order order = new Order();
			order.setUid(i);
			order.setUsername("测试用户4");
			order.setPid(pid);
			order.setPname("手机充值卡");
			order.setPprice(97.50);
			order.setNumber(1);
			rocketMQTemplate.sendOneWayOrderly("order-topic", order, "xx");
		}
	}
```


#### 1.4 半事务消息

```java
@Service
public class TransactionProducerService {
 
    @Resource
    private RocketMQTemplate rocketMQTemplate;
 
    public void sendInTransaction(){
        for(int i=0;i<100;i++){
            TransactionSendResult result=rocketMQTemplate.sendMessageInTransaction("transaction",
                    "topic-3", MessageBuilder.withPayload("事务消息"+i).build(), i);
            System.out.println(result);
        }
    }
}
 
@Component
@RocketMQTransactionListener(txProducerGroup = "transaction")
class LocalExecutor implements RocketMQLocalTransactionListener {
 
    private ConcurrentHashMap<Integer, RocketMQLocalTransactionState> map=new ConcurrentHashMap<>();
 
    @Override
    public RocketMQLocalTransactionState executeLocalTransaction(Message message, Object o) {
        map.put(message.hashCode(),RocketMQLocalTransactionState.UNKNOWN);
 
        int i=Integer.parseInt(o.toString());
        if(i==2){
            System.out.println("本地事务出错，回滚事务消息");
            map.put(message.hashCode(),RocketMQLocalTransactionState.ROLLBACK);
        }else {
            map.put(message.hashCode(),RocketMQLocalTransactionState.COMMIT);
        }
 
        return map.get(message.hashCode());
    }
 
    @Override
    public RocketMQLocalTransactionState checkLocalTransaction(Message message) {
        return map.get(message.hashCode());
    }
}
```


#### 1.5 客户局

```java
import org.apache.rocketmq.spring.annotation.ConsumeMode;
import org.apache.rocketmq.spring.annotation.MessageModel;
import org.apache.rocketmq.spring.annotation.RocketMQMessageListener;
import org.apache.rocketmq.spring.core.RocketMQListener;
import org.springframework.stereotype.Service;

import cn.com.rocketmq.domain.Order;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Service
@RocketMQMessageListener(consumerGroup = "shop-user", // 消费者组名
		topic = "order-topic", // 消费主题
		consumeMode = ConsumeMode.CONCURRENTLY, // 消费模式,指定是否顺序消费 CONCURRENTLY(同步,默认) ORDERLY(顺序)
		messageModel = MessageModel.CLUSTERING// 消息模式 BROADCASTING(广播) CLUSTERING(集群,默认)
)
public class OrderBonusListener implements RocketMQListener<Order> {

	// 消费逻辑
	@Override
	public void onMessage(Order message) {
		log.info("接收到了一个订单信息{},接下来就可以发送短信通知了", message);
	}
}
```

总结：

| 发送方式 | 发送TPS | 发送结果反馈 | 可靠性
| ----- | ----- | ----- | ----- |
| 同步发送| 快| 有| 不丢失 | 
| 异步发送| 快| 有| 不丢失 | 
| 单向发送| 最快| 无| 可能丢失 | 

## Kafka

Name Server分布式、主从

## RabbitMQ

Erlang集群主从
