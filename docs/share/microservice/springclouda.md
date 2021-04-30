# Spring Cloud Alibaba

## 1. Sentinel分布式系统流量防卫兵

```bash
java -Dserver.port=8858 -Dcsp.sentinel.dashboard.server=localhost:8858 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard-1.7.0.jar
```

```yml
spring:
   cloud:
      nacos:
         discovery:
            server-addr: http://xuzhihao:8848
         config:
            server-addr: http://xuzhihao:8848
            file-extension: yaml
      sentinel:
         transport:
            dashboard: xuzhihao:8858
```

### 1.1 @SentinelResource注解参数

通过@SentinelResource来指定出现异常时的处理策略。用于定义资源，并提供可选的异常处理和 fallback 配置项

| 文件夹| 说明 | 
| ----- | ----- | 
| value | 资源名称 |
| entryType | entry类型，标记流量的方向，取值IN/OUT，默认是OUT |
| blockHandler | 处理BlockException的函数名称,函数要求：<br>1. 必须是 public<br>2. 返回类型 参数与原方法一致<br>3. 默认需和原方法在同一个类中。若希望使用其他类的函数，可配置blockHandlerClass ，并指定lockHandlerClass里面的方法。 |
| blockHandlerClass | 存放blockHandler的类,对应的处理函数必须static修饰。 | 
| fallback | 用于在抛出异常的时候提供fallback处理逻辑。fallback函数可以针对所有类型的异常（除了 exceptionsToIgnore 里面排除掉的异常类型）进行处理。函数要求：<br>1. 返回类型与原方法一致<br>2. 参数类型需要和原方法相匹配<br>3. 默认需和原方法在同一个类中。若希望使用其他类的函数，可配置fallbackClass ，并指定fallbackClass里面的方法。 | 
| fallbackClass | 存放fallback的类。对应的处理函数必须static修饰。 | 
| defaultFallback | 用于通用的 fallback 逻辑。默认fallback函数可以针对所有类型的异常进行处理。若同时配置了 fallback 和 defaultFallback，以fallback为准。函数要求：<br>1. 返回类型与原方法一致<br>2. 方法参数列表为空，或者有一个 Throwable 类型的参数。<br>3. 默认需要和原方法在同一个类中。若希望使用其他类的函数，可配置fallbackClass ，并指定 fallbackClass 里面的方法。 | 
| exceptionsToIgnore | 指定排除掉哪些异常。排除的异常不会计入异常统计，也不会进入fallback逻辑，而是原样抛出。 | 
| exceptionsToTrace | 需要trace的异常 | 

本地资源加载配置文件见`RuleConstant.java`

### 1.2 

### 1.3 


## 2. Nacos动态服务发现、配置管理

### 2.1 集群配置

nacos/的conf目录下cluster.conf

```xml
192.168.16.101:8847
192.168.16.102:8848
192.168.16.103:8849
```

application.properties配置主备
```properties
spring.datasource.platform=mysql

db.num=2
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.url.1=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user=root
db.password=root
```

两种配置：

1. 将所有节点添加到配置文件中
```yml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: http://nacos1:8848,http://nacos2:8848,http://nacos3:8848
      config:
        server-addr: http://debug-registry:8848
        file-extension: yaml
```

2. 将三个节点映射出一套VIP地址，配置文件中写一个即可


## 3. RocketMQ

[消息队列/RocketMQ](/share/distributed/mq?id=rocketmq)

## 4. Dubbo：Apache Dubbo

[Dubbo源码解析](/share/distributed/dubbo)


## 5. Seata高性能微服务分布式事务解决方案

## 6. Alibaba Cloud ACM

## 7. Alibaba Cloud OSS

## 8. Alibaba Cloud SchedulerX

## 9. Alibaba Cloud SMS

```java
package cn.com.rocketmq.utils;

import com.aliyuncs.DefaultAcsClient;
import com.aliyuncs.IAcsClient;
import com.aliyuncs.dysmsapi.model.v20170525.SendSmsRequest;
import com.aliyuncs.dysmsapi.model.v20170525.SendSmsResponse;
import com.aliyuncs.profile.DefaultProfile;
import com.aliyuncs.profile.IClientProfile;
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class SmsUtil {
	// 替换成自己申请的accessKeyId
	private static String accessKeyId = "xxx";
	// 替换成自己申请的accessKeySecret
	private static String accessKeySecret = "xxx";

	static final String product = "Dysmsapi";
	static final String domain = "dysmsapi.aliyuncs.com";

	/**
	 * 发送短信
	 *
	 * @param phoneNumbers 要发送短信到哪个手机号
	 * @param signName     短信签名[必须使用前面申请的]
	 * @param templateCode 短信短信模板ID[必须使用前面申请的]
	 * @param param        模板中${code}位置传递的内容
	 */
	public static void sendSms(String phoneNumbers, String signName, String templateCode, String param) {
		try {
			System.setProperty("sun.net.client.defaultConnectTimeout", "10000");
			System.setProperty("sun.net.client.defaultReadTimeout", "10000");

			// 初始化acsClient,暂不支持region化
			IClientProfile profile = DefaultProfile.getProfile("cn-hangzhou", accessKeyId, accessKeySecret);
			DefaultProfile.addEndpoint("cn-hangzhou", "cn-hangzhou", product, domain);
			IAcsClient acsClient = new DefaultAcsClient(profile);

			SendSmsRequest request = new SendSmsRequest();
			request.setPhoneNumbers(phoneNumbers);
			request.setSignName(signName);
			request.setTemplateCode(templateCode);
			request.setTemplateParam(param);
			request.setOutId("yourOutId");
			SendSmsResponse sendSmsResponse = acsClient.getAcsResponse(request);
			if (!"OK".equals(sendSmsResponse.getCode())) {
				log.info("发送短信失败,{}", sendSmsResponse);
				throw new RuntimeException(sendSmsResponse.getMessage());
			}
		} catch (Exception e) {
			log.info("发送短信失败,{}", e);
			throw new RuntimeException("发送短信失败");
		}
	}
}
```
