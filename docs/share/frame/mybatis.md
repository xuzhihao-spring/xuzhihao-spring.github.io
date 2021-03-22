# Mybatis源码

## 1. Mybatis优缺点

JDBC的弊端:
- 底层没有用到连接池，频繁的创建和关闭，消耗资源
- 非对象的方式传递参，sql需要修改的时候需要重新编译，不利于维护
- 预编译的时候变量位置使用123，不利于维护
- result结果集需要硬编码


## 2. 调用sql的三种方式，优先级是什么

## 3. mybaits的一级缓存、二级缓存

## 4. 插件扩展原理和经典案例

## 5. mybaits和spring整合是如何用到FactoryBean

## 6. #和$的区别和genericTokenParser解析器

## 7. 加载mappers配置的四种方式，优先级

1.使用相对于类路径的资源引用 resource

```xml
<!-- 使用相对于类路径的资源引用 -->
<mappers>
  <mapper resource="org/mybatis/builder/AuthorMapper.xml"/>
  <mapper resource="org/mybatis/builder/BlogMapper.xml"/>
  <mapper resource="org/mybatis/builder/PostMapper.xml"/>
</mappers>
```
2.使用完全限定资源定位符 url
```xml
<!-- 使用完全限定资源定位符（URL） -->
<mappers>
  <mapper url="file:///var/mappers/AuthorMapper.xml"/>
  <mapper url="file:///var/mappers/BlogMapper.xml"/>
  <mapper url="file:///var/mappers/PostMapper.xml"/>
</mappers>
```
3.使用映射器接口实现类的完全限定类名 class
```xml
<!-- 使用映射器接口实现类的完全限定类名 -->
<mappers>
  <mapper class="org.mybatis.builder.AuthorMapper"/>
  <mapper class="org.mybatis.builder.BlogMapper"/>
  <mapper class="org.mybatis.builder.PostMapper"/>
</mappers>
```
4.将包内的映射器接口实现全部注册为映射器 package
```xml
<!-- 将包内的映射器接口实现全部注册为映射器 -->
<mappers>
  <package name="org.mybatis.builder"/>
</mappers>
```


源码验证

XMLConfigBuilder类下的this.mapperElement(root.evalNode("mappers"))

```java
// 加载mapper的形式有4种
private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
            // 4、package 将包内的映射器接口实现全部注册为映射器 <package //
            // name="com.bestcxx.stu.springmvc.mapper"/>
            if ("package".equals(child.getName())) {
                String mapperPackage = child.getStringAttribute("name");
                //注解方式此处放入
                configuration.addMappers(mapperPackage);
            } else {
                String resource = child.getStringAttribute("resource");
                String url = child.getStringAttribute("url");
                String mapperClass = child.getStringAttribute("class");
                // 1、resource 使用相对于类路径的资源引用 <mapper
                // resource="mybatis/mappings/UserModelMapper.xml"/>
                if (resource != null && url == null && mapperClass == null) {
                    ErrorContext.instance().resource(resource);
                    InputStream inputStream = Resources.getResourceAsStream(resource);
                    XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource,
                            configuration.getSqlFragments());
                    mapperParser.parse();
                    // 2、url 使用完全限定资源定位符 <mapper url="file:///var/mappings/UserModelMapper.xml"/>
                } else if (resource == null && url != null && mapperClass == null) {
                    ErrorContext.instance().resource(url);
                    InputStream inputStream = Resources.getUrlAsStream(url);
                    XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url,
                            configuration.getSqlFragments());
                    mapperParser.parse();
                    // 3、class 使用映射器接口实现类的完全限定类名 <mapper
                    // class="com.bestcxx.stu.springmvc.mapper.UserModelMapper"/>
                } else if (resource == null && url == null && mapperClass != null) {
                    Class<?> mapperInterface = Resources.classForName(mapperClass);
                    configuration.addMapper(mapperInterface);
                } else {
                    throw new BuilderException(
                            "A mapper element may only specify a url, resource or class, but not more than one.");
                }
            }
        }
    }
}
```