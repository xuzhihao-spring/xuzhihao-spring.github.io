# Spring的底层原理与源码实现

## 1. 常用注解

### 1.1 @Import

实现ImportSelector接口是springboot自动装配的核心原来

导入ImportBeanDefinitionRegistrar的实现类方式是`Feign`整合的关键

导入bean的三种方式

1. 导入@Configuration注解的配置类或者是直接将bean导入，导入的bean以全限定类名注册到容器中
   
```java
@Data
public class User {
	private int id;
	private String name;
}

@ComponentScan(value = "com.xuzhihao")
@Import({ User.class })
public class SpringConfiguration {

}
//输出结果
AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(SpringConfiguration.class);
String[] names = ac.getBeanDefinitionNames();
for (String name : names) {
    System.out.println(name);
}

```

2. 导入实现ImportSelector接口或子接口DeferredImportSelector的类

重写ImportSelector的selectImports方法 两种使用方式 1. 返回值注册 2. 通过beanFactory注册

```java
@Import({ MyImportSelector.class })
public class SpringConfiguration {

}

//单独的class
@Data
public class Order {
	private int id;
	private String name;
}
//单独的class
@Data
public class Stock {
	private int id;
	private String name;
}



public class MyImportSelector implements ImportSelector, BeanFactoryAware {

	private BeanFactory beanFactory;

	@Override
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
		GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
		beanDefinition.setBeanClass(Stock.class);
		beanDefinition.getPropertyValues().addPropertyValue("id", 123456);
		((DefaultListableBeanFactory) beanFactory).registerBeanDefinition("Stock", beanDefinition);//beanName注册
		return new String[] { "com.xuzhihao.domain.Order" };//全限定类型注册
	}

	@Override
	public void setBeanFactory(BeanFactory beanFactory) {
		this.beanFactory = beanFactory;
	}
}

//输出结果
AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(SpringConfiguration.class);
String[] names = ac.getBeanDefinitionNames();
for (String name : names) {
    System.out.println(name);
}
System.out.println(ac.getBean("Stock"));
```


3. 导入ImportBeanDefinitionRegistrar的实现类

```java
@ComponentScan(value = "com.xuzhihao")
@Import({ MyImportBeanDefinitionRegistrar.class })
public class SpringConfiguration {

}

public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {

	@Override
	public void registerBeanDefinitions(AnnotationMetadata annotationMetadata, BeanDefinitionRegistry registry) {
		RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(Stock.class);
		registry.registerBeanDefinition("Stock2", rootBeanDefinition);
	}

}

//输出结果
AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(SpringConfiguration.class);
String[] names = ac.getBeanDefinitionNames();
for (String name : names) {
    System.out.println(name);
}
System.out.println(ac.getBean("Stock2"));

```



## 1. Spring核心知识点

![](../../images/share/frame/spring/spring.png)

## 2. Bean的生命周期原理详解与源码分析

![](../../images/share/frame/spring/bean.png)

## 3. 依赖注入的工作流程

![](../../images/share/frame/spring/ioc.png)

## 4. BeanFactory架构

![](../../images/share/frame/spring/beanfactory.png)

## 4. 事务隔离级别

![](../../images/share/frame/spring/transaction.png)