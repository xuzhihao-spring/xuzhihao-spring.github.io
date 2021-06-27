# 消息队列

## 1. RocketMQ

Zookeeper集群分布式、主从,支持顺序，延时，事务消息，支持查询重试

RocketMQ支持两种消息模式:
- 广播消费: 每个消费者实例都会收到消息,也就是一条消息可以被每个消费者实例处理；
- 集群消费: 一条消息只能被一个消费者实例消费

### 1.1 消息类型

#### 1.1.1 可靠同步发送

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

#### 1.1.2 可靠异步发送

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

#### 1.1.3 单向发送

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

#### 1.1.4 单向顺序发送

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

#### 1.1.5 延迟消息

```java
	// 同步延时
	@RequestMapping("/sendDelay/{pid}")
	public String sendDelay(@PathVariable("pid") Integer pid) {
		SendResult result = rocketMQTemplate.syncSend("delay-topic", MessageBuilder.withPayload(pid).build(), 3000, 4);
		return result.getMsgId();

	}
```

#### 1.1.6 半事务消息

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


#### 1.1.7 客户端

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


### 1.2 MQ中间件消费端可靠性的保障

#### 1.2.1 RabbitMQ的手动消息确认ACK机制

RabbitMQ提供了消息确认机制。消费者在订阅队列时，可以在代码中手动设置autoAck参数为false，这时RabbitMQ会等待消费者显式地回复确认信号（即为显式地调用channel.basicAck(envelope.getDeliveryTag(), false)方法）后才从集群中的内存（或磁盘）节点上移除消息，从而保证了这条消息不会因为消费失败而导致丢失


#### 1.2.2 Kafka消息消费的手动提交

在Kafka中，也可以采用上面那种的消费后的确认机制，通过在Consumer端设置“enable.auto.commit”属性为false后，待业务工程正常处理完消费后，在代码中手动调用KafkaConsumer实例的commitSync()方法提交（ps：这里指的是同步阻塞commit消费的偏移量，等待Broker端的返回响应，需要注意Broker端在对commit请求做出响应之前，消费端会处于阻塞状态，从而限制消息的处理性能和整体吞吐量），以确保消息能够正常被消费。如果在消费过程中，消费端突然Crash，这时候消费偏移量没有commit，等正常恢复后依然还会处理刚刚未commit的消息


#### 1.2.3 RocketMQ消费失败后的消费重试机制

1. 重试队列

如果Consumer端因为各种类型异常导致本次消费失败，为防止该消息丢失而需要将其重新回发给Broker端保存，保存这种因为异常无法正常消费而回发给MQ的消息队列称之为重试队列。RocketMQ会为每个消费组都设置一个Topic名称为“%RETRY%+consumerGroup”的重试队列（这里需要注意的是，这个Topic的重试队列是针对消费组，而不是针对每个Topic设置的），用于暂时保存因为各种异常而导致Consumer端无法消费的消息。考虑到异常恢复起来需要一些时间，会为重试队列设置多个重试级别，每个重试级别都有与之对应的重新投递延时，重试次数越多投递延时就越大。RocketMQ对于重试消息的处理是先保存至Topic名称为“SCHEDULE_TOPIC_XXXX”的延迟队列中，后台定时任务按照对应的时间进行Delay后重新保存至“%RETRY%+consumerGroup”的重试队列中

2. 死信队列

由于有些原因导致Consumer端长时间的无法正常消费从Broker端Pull过来的业务消息，为了确保消息不会被无故的丢弃，那么超过配置的“最大重试消费次数”后就会移入到这个死信队列中。在RocketMQ中，SubscriptionGroupConfig配置常量默认地设置了两个参数，一个是retryQueueNums为1（重试队列数量为1个），另外一个是retryMaxTimes为16（最大重试消费的次数为16次）。Broker端通过校验判断，如果超过了最大重试消费次数则会将消息移至这里所说的死信队列。这里，RocketMQ会为每个消费组都设置一个Topic命名为“%DLQ%+consumerGroup"的死信队列。一般在实际应用中，移入至死信队列的消息，需要人工干预处理



## 2. Kafka

### 2.1 集群搭建

解压安装

```bash
cd /export/software/
tar -xvzf kafka_2.12-2.4.1.tgz -C ../server/
cd /export/server/kafka_2.12-2.4.1/

# 修改 server.properties
cd /export/server/kafka_2.12-2.4.1/config
vim server.properties
# 指定broker的id
broker.id=0
# 指定Kafka数据的位置
log.dirs=/export/server/kafka_2.12-2.4.1/data
# 配置zk的三个节点
zookeeper.connect=node1:2181,node2:2181,node3:2181

```

复制

```bash
cd /export/server
scp -r kafka_2.12-2.4.1/ node2.itcast.cn:$PWD
scp -r kafka_2.12-2.4.1/ node3.itcast.cn:$PWD

# 修改另外两个节点的broker.id分别为1和2
#---------node2--------------
cd /export/server/kafka_2.12-2.4.1/config
vim erver.properties
broker.id=1

#---------node3--------------
cd /export/server/kafka_2.12-2.4.1/config
vim server.properties
broker.id=2
```

配置KAFKA_HOME环境变量

```bash
vim /etc/profile
export KAFKA_HOME=/export/server/kafka_2.12-2.4.1
export PATH=:$PATH:${KAFKA_HOME}

#分发到各个节点
scp /etc/profile node2:$PWD
scp /etc/profile node3:$PWD
每个节点加载环境变量
source /etc/profile

```


```bash
# 启动ZooKeeper
nohup bin/zookeeper-server-start.sh config/zookeeper.properties &

# 启动Kafka
cd /export/server/kafka_2.12-2.4.1
nohup bin/kafka-server-start.sh config/server.properties &
# 测试Kafka集群是否启动成功
bin/kafka-topics.sh --bootstrap-server node1:9092 --list

```

### 2.2 Kafka中的分区副本机制


### 2.3 Kafka原理



## 3. RabbitMQ

### 3.1 相关定义

- Broker： 简单来说就是消息队列服务器实体
- Exchange： 消息交换机，它指定消息按什么规则，路由到哪个队列
- Queue： 消息队列载体，每个消息都会被投入到一个或多个队列
- Binding： 绑定，它的作用就是把exchange和queue按照路由规则绑定起来
- Routing Key： 路由关键字，exchange根据这个关键字进行消息投递
- VHost： 虚拟主机，一个broker里可以开设多个vhost，用作不同用户的权限分离。
- Producer： 消息生产者，就是投递消息的程序
- Consumer： 消息消费者，就是接受消息的程序
- Channel： 消息通道，在客户端的每个连接里，可建立多个channel，每个channel代表一个会话任务

由Exchange、Queue、RoutingKey三个才能决定一个从Exchange到Queue的唯一的线路。

### 3.2 工作模式

`RabbitMQ不同的发布方式是通过Exchange的设置来完成，这一点区别于RocketMQ是通过消费端绑定方式来区分`

#### 3.2.1 simple：简单模式

消息发布者（Publish）将消息放入队列（默认交换机）

消息的消费者（Consumer）监听（while）消息队列，如果队列中有消息，就消费掉，消息被拿走后，自动从队列中删除（隐患：消息可能没有被消费者正确处理，已经从队列中消失了，造成消息的丢失）。应用场景：聊
天（中间有一个过度的服务器；P端，C端）

#### 3.2.2 work：工作模式(资源的竞争)

一个生产者，多个消费者，消息被多个消费者竞争接收。使用场景多台消费主机分布式部署

#### 3.2.3 publish/subscribe：发布订阅(共享资源)

相关场景:邮件群发,群聊天,广播(广告)

#### 3.2.4 routing：路由模式

一个生产者，多个消费者，多个消费者根据路由类型分摊处理消息，减轻单个客户端的负载压力

场景：日志

#### 3.2.5 topic：主题模式

一个生产者，多个消费者，多个消费者根据路由类型分摊处理消息，此处的路由的用*号做模糊匹配的。

#### 3.2.6 Header：头交换机模式

一个生产者，多个消费者，在消息头添加条件，匹配才被消费者接收使用（header Exchange类型用的比较少）。


### 3.3 过期时间TTL

消息的有效期，因为网络问题或者逻辑问题，导致消息无法被正确消费，给每一条数据一个过期时间，这样数据超时了就会自动变为一个死信数据

### 3.4 死信队列DLX（dead-letter-exchange）

消息变成死信有以下几种情况

- 消息被拒绝(basic.reject / basic.nack)，并且requeue = false
- 消息TTL过期
- 队列达到最大长度

```java
package cn.com.rabbitmq.config;

import java.util.HashMap;
import java.util.Map;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.DirectExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * 基于TTL和死信队列实现延迟任务
 * @author Administrator
 *
 */
@Configuration
public class RabbitmqDLXConfig {
	/**
	 * 死信交换机
	 *
	 * @return
	 */
	@Bean
	public DirectExchange userOrderDelayExchange() {
		return new DirectExchange("user.order.delay_exchange");
	}

	/**
	 * 死信队列
	 *
	 * @return
	 */
	@Bean
	public Queue userOrderDelayQueue() {
		Map<String, Object> map = new HashMap<>(16);
		map.put("x-dead-letter-exchange", "user.order.receive_exchange");
		map.put("x-dead-letter-routing-key", "user.order.receive_key");
		return new Queue("user.order.delay_queue", true, false, false, map);
	}

	/**
	 * 给死信队列绑定交换机
	 *
	 * @return
	 */
	@Bean
	public Binding userOrderDelayBinding() {
		return BindingBuilder.bind(userOrderDelayQueue()).to(userOrderDelayExchange()).with("user.order.delay_key");
	}

	/**
	 * 死信接收交换机
	 *
	 * @return
	 */
	@Bean
	public DirectExchange userOrderReceiveExchange() {
		return new DirectExchange("user.order.receive_exchange");
	}

	/**
	 * 死信接收队列
	 *
	 * @return
	 */
	@Bean
	public Queue userOrderReceiveQueue() {
		return new Queue("user.order.receive_queue");
	}

	/**
	 * 死信交换机绑定消费队列
	 *
	 * @return
	 */
	@Bean
	public Binding userOrderReceiveBinding() {
		return BindingBuilder.bind(userOrderReceiveQueue()).to(userOrderReceiveExchange())
				.with("user.order.receive_key");
	}

}
```
```java
public void send3(String orderId) {
		// 给延迟TTL队列发送消息 同样步长的场景使用
		OrderItem order = new OrderItem();
		order.setId(Long.valueOf(orderId));
		order.setProductName("钻石头盔" + System.currentTimeMillis());
		rabbitTemplate.convertAndSend("user.order.delay_exchange", "user.order.delay_key", JSONUtil.toJsonStr(order),
				message -> {
					message.getMessageProperties().setExpiration("3000");
					return message;
				}, new CorrelationData(orderId));
	}
```


### 3.5 延迟队列

```bash
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```

```java
package cn.com.rabbitmq.config;

 
import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.CustomExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
 
import java.util.HashMap;
import java.util.Map;
 
/**
 * 延时队列 可实现不同间隔的延迟 彼此不受影响
 * 基于rabbitmq-delayed-message-exchange 插件完成
**/
@Configuration
public class RabbitDelayConfig {
	public static final String QUEUE_NAME = "delay_queue";

	public static final String EXCHANGE_NAME = "delay_exchange";

	/**
	 * 延迟队列
	 * @return
	 */
	@Bean
	Queue queue() {    
	      return new Queue(QUEUE_NAME, true);
	}

	// 定义一个延迟交换机
	@Bean
	CustomExchange delayExchange() {    
	    Map<String, Object> args = new HashMap<String, Object>();    
	    args.put("x-delayed-type", "direct");    
	    return new CustomExchange(EXCHANGE_NAME, "x-delayed-message", true, false, args);
	}

	// 绑定队列到这个延迟交换机上
	@Bean
	Binding binding(Queue queue, CustomExchange delayExchange) {    
	    return BindingBuilder.bind(queue).to(delayExchange).with(QUEUE_NAME).noargs();
	}
}
```

```java
public void send2(String orderId) {
		// 给延迟队列发送消息，彼此间隔不同无任何影响
		OrderItem order = new OrderItem();
		order.setId(Long.valueOf(orderId));
		order.setProductName("蓝牙耳机" + System.currentTimeMillis());

		rabbitTemplate.convertAndSend("delay_exchange", "delay_queue", JSONUtil.toJsonStr(order), message -> {
			// 配置消息的过期时间
			message.getMessageProperties().setDelay(3000);
			return message;
		}, new CorrelationData(orderId));
	}
```

### 3.6 消息确认机制

#### 3.6.1 发布确认

```java
@Autowired
	private RabbitMQConfirmAndReturn rabbitMQConfirmAndReturn;

	@PostConstruct
	public void initRabbitTemplate() {
		rabbitTemplate.setMandatory(true);
		rabbitTemplate.setConfirmCallback(rabbitMQConfirmAndReturn);
		rabbitTemplate.setReturnCallback(rabbitMQConfirmAndReturn);
	}
```

```java
package cn.com.rabbitmq.config;

import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.connection.CorrelationData;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.stereotype.Component;

import lombok.extern.slf4j.Slf4j;

@Component
@Slf4j
public class RabbitMQConfirmAndReturn implements RabbitTemplate.ConfirmCallback, RabbitTemplate.ReturnCallback {

    /**
     * confirm机制只保证消息到达exchange，不保证消息可以路由到正确的queue,如果exchange错误，就会触发confirm机制
     *
     * @param correlationData
     * @param ack
     * @param cause
     */
    @Override
    public void confirm(CorrelationData correlationData, boolean ack, String cause) {
        if (ack) {
            log.debug("{}-到达路由",correlationData.getId());
        }else {
        	log.error("{}-未到达路由,cause:{}",correlationData.getId(), cause);
        }
    }


    /**
     * Return 消息机制用于处理一个不可路由的消息。在某些情况下，如果我们在发送消息的时候，当前的 exchange 不存在或者指定路由 key 路由不到，这个时候我们需要监听这种不可达的消息
      * 就需要这种return机制 
     * @param message
     * @param replyCode
     * @param replyText
     * @param exchange
     * @param routingKey
     */
    @Override
    public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
        log.error("消息未达队列,message:{},replyCode:{},replyText:{},exchange:{},routing:{}", message.toString(), replyCode, replyText, exchange, routingKey);
    }
}
```

#### 3.6.2 接收确认

1. basicAck    表示成功确认，使用此回执方法后，消息会被rabbitmq broker 删除。 	
```java
void basicAck(long deliveryTag, boolean multiple)
```
  - deliveryTag：表示消息投递序号，每次消费消息或者消息重新投递后，deliveryTag都会增加。手动消息确认模式下，我们可以对指定deliveryTag的消息进行ack、nack、reject等操作。
  - multiple：是否批量确认，值为 true 则会一次性 ack所有小于当前消息 deliveryTag 的消息。

举个栗子： 假设我先发送三条消息deliveryTag分别是5、6、7，可它们都没有被确认，当我发第四条消息此时deliveryTag为8，multiple设置为 true，会将5、6、7、8的消息全部进行确认

2. basicNack    表示失败确认，一般在消费消息业务异常时用到此方法，可以将消息重新投递入队列
```java
void basicNack(long deliveryTag, boolean multiple, boolean requeue)
```
  - deliveryTag：表示消息投递序号。
  - multiple：是否批量确认。
  - requeue：值为 true 消息将重新入队列

3. basicReject    拒绝消息，与basicNack区别在于不能进行批量操作，其他用法很相似
```java
void basicReject(long deliveryTag, boolean requeue)
```
  - deliveryTag：表示消息投递序号。

  - requeue：值为 true 消息将重新入队列

### 3.7 消息确认机制-事务支持

确认并且保证消息被送达，提供了两种方式：发布确认和事务。(两者不可同时使用)在channel为事务时，不可引入确认模式；同样channel为确认模式下，不可使用事务

### 3.8 消息追踪

```java
rabbitmq-plugins enable rabbitmq_tracing
rabbitmq-plugins disable rabbitmq_tracing
```

### 3.9 集群搭建

#### 3.9.1 HAProxy 实现镜像队列的负载均衡

[HAProxy安装配置](share/database/mycat?id=_624-haproxy安装配置)

#### 3.9.2 KeepAlived 搭建高可用的HAProxy集群

[KeepAlived安装配置](share/database/mycat?id=_625-keepalived安装配置)

### 3.10 集群监控

1. rabbitmq控制台页面监控
2. tracing日志监控
3. 使用api接口自定义实现监控
4. [使用Zabbix监控rabbitmq](linux/deploy/zabbix)

### 3.11 常见问题

消息堆积 
>堆积的原因：异常消费 转移故障点？消费能力不够 动态扩容

消息丢失
>发送丢增加确认机制，但是会影响性能    消费丢增加ack    服务器丢,队列、交换器创建的时候进行持久化


有序消费消息
>真正需要处理的是如何保证多个消息的业务逻辑的有序性

重复消费
>保持幂等性

## 4. EMQX

### 4.1 基础功能

- Dashboard
- 系统认证、
  - UserName认证
  - Client Id认证
  - HTTP认证(基于webhock)
- 客户端DSK
  - Eclipse Paho Java
- 日志追踪


开启auth_username插件（v4.0.5）
```
@hostname = 172.17.17.80
@port=8081
@contentType=application/json
@userName=admin
@password=public

#############查看已有用户认证数据##############
GET http://{{hostname}}:{{port}}/api/v4/auth_username HTTP/1.1
Content-Type: {{contentType}}
Authorization: Basic {{userName}}:{{password}}


########添加用户认证数据##############
POST http://{{hostname}}:{{port}}/api/v4/auth_username HTTP/1.1
Content-Type: {{contentType}}
Authorization: Basic {{userName}}:{{password}}

{
    "username": "user",
    "password": "123456"
}



###########更改指定用户名的密码#############
PUT http://{{hostname}}:{{port}}/api/v4/auth_username/user HTTP/1.1
Content-Type: {{contentType}}
Authorization: Basic {{userName}}:{{password}}

{
    "password": "user"
}


###########查看指定用户名信息#############
GET http://{{hostname}}:{{port}}/api/v4/auth_username/user HTTP/1.1
Content-Type: {{contentType}}
Authorization: Basic {{userName}}:{{password}}


###########删除指定的用户信息#############
DELETE http://{{hostname}}:{{port}}/api/v4/auth_username/user HTTP/1.1
Content-Type: {{contentType}}
Authorization: Basic {{userName}}:{{password}}
```

### 4.2 高级功能

- ACL权限控制
  - 内置ACL
  - HTTP ACL
    - HTTP请求
    - superuser请求
    - 授权查询
- WebHook
- 集群设置
- API监控管理
- 保留消息

### 4.3 高级功能2

- 共享订阅
  - 带群组  $share/<group-name>
  - 不带群组  $queue/
- 延迟发布
- 代理订阅
  - 静态代理
  - 动态代理
- 主题重写
- 黑名单
- 速率限制

### 4.4 高级功能3

- 飞行窗口
- 消息重传
- 规则引擎
  - 故障监听案例
  - 数据筛选限速提醒案例
  - 消息筛选过滤
- 系统调优