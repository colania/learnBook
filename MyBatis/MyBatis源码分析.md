## 1. 配置解析过程

### 1. 解析mybatis-config.xml配置文件

```java
 				String resource = "mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```

初始化了一个SqlSessionFactory,这里用到了建造者模式。



用于解析 mybatis-config.xml，同时创建了 Configuration 对象.

```java
/**
* 生成一个解析xml文件的XMLConfigBuilder对象，同时创建出存储配置的Configuration。
**/
XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);

private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
    super(new Configuration()); // 完成了Configuration的初始化,为类型注册别名
    ErrorContext.instance().resource("SQL Mapper Configuration");
    this.configuration.setVariables(props); // 设置对应的Properties属性
    this.parsed = false; // 设置 是否解析的标志为 false
    this.environment = environment; // 初始化environment
    this.parser = parser; // 初始化 解析器
  }

```



```java
//开始解析xml
return build(parser.parse());
//parse方法内，该方法内对应xml文件内的标签
parseConfiguration(parser.evalNode("/configuration"));


private void parseConfiguration(XNode root) {
    try {
      //issue #117 read properties first
      // 对于全局配置文件各种标签的解析
      propertiesElement(root.evalNode("properties"));
      // 解析 settings 标签
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      // 读取文件
      loadCustomVfs(settings);
      // 日志设置
      loadCustomLogImpl(settings);
      // 类型别名
      typeAliasesElement(root.evalNode("typeAliases"));
      // 插件
      pluginElement(root.evalNode("plugins"));
      // 用于创建对象
      objectFactoryElement(root.evalNode("objectFactory"));
      // 用于对对象进行加工
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      // 反射工具箱
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      // settings 子标签赋值，默认值就是在这里提供的 >>
      settingsElement(settings);
      // read it after objectFactory and objectWrapperFactory issue #631
      // 创建了数据源 >>
      environmentsElement(root.evalNode("environments"));
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      typeHandlerElement(root.evalNode("typeHandlers"));
      // 解析引用的Mapper映射器
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
```



```java
解析<properties>标签，引入外部配置文件
propertiesElement(root.evalNode("properties"));

// 更新对应的属性信息
parser.setVariables(defaults);
configuration.setVariables(defaults);
解析的最终结果变成configuration的属性。
```

  

```java
解析数据库配置
environmentsElement(root.evalNode("environments"));
每个enviroment对象对应一个数据源，根据配置的<transactionManager>创建一个事务工厂，根据<datasource>创建一个数据源，最后放入configuration的environment属性中。
  
```



```java
处理自定义的typehandler-->数据库，java字段的对应关系
Map<Type, Map<JdbcType, TypeHandler<?>>> typeHandlerMap
typeHandlerElement(root.evalNode("typeHandlers"));
```

### 2. 解析mapper文件



```java
// 解析引用的Mapper映射器，解析mappers标签
mapperElement(root.evalNode("mappers"));
根据配置文件中不同的配置方式，用不同的方式扫描，但最终做的仍然是接口的注册和语句的注册。
 
  public void parse() {
    // 总体上做了两件事情，对于语句的注册和接口的注册
    if (!configuration.isResourceLoaded(resource)) {
      // 1、具体增删改查标签的解析。
      // 一个标签一个MappedStatement。 >>
      configurationElement(parser.evalNode("/mapper"));
      configuration.addLoadedResource(resource);
      // 2、把namespace（接口类型）和工厂类绑定起来，放到一个map。
      // 一个namespace 一个 MapperProxyFactory >>
      bindMapperForNamespace();
    }

    parsePendingResultMaps();
    parsePendingCacheRefs();
    parsePendingStatements();
  }
```

* configurationElement是对mapper.xml文件的解析。包括namespace，parameterMap,resultMap,select|insert|update|delete

* bindMapperForNamespace（）通过反射获取到接口类，调用 configuration.addMapper(boundType);将接口类型注册到**MapperRegistry**：为接口创建一个对应的的MapperProxyFactory.

```java
knownMappers.put(type, new MapperProxyFactory<>(type));
```



* 注册接口后，解析接口类上的注解@Select,创建MappedStatement，添加到MapperRegistry中。(注解和xml中的效果一致)

### 3. 返回DefalultSqlSessionFactory

Mapper.xml解析完成后。调用build(),返回SqlSessionFactory的默认实现类DefaultSqlSessionFactory。

```java
public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }
```



#### 总结

在这里主要完成的是config配置，Mapper.xml，Mapper接口的解析和绑定。

生成一个重要的Configuration对象，里面有全部的配置信息，还有各种各样的容器。返回的DefalultSqlSessionFactory里有Configuration的实例。

<img src="/Users/wwei/Library/Application Support/typora-user-images/image-20210426152043576.png" alt="加载配置文件" style="zoom:50%;" />



## 会话创建

程序每次操作数据库，都创建一个会话，用openSession()创建。

```java
SqlSession sqlSession = sqlSessionFactory.openSession();
```

这个会话里包含一个Executor来执行sql，executor要制定事务类型和执行器的类型。

所以我们从Configuration中拿到Enviroment,有之前创建的事务工厂。

```java
  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      final Environment environment = configuration.getEnvironment();
      // 获取事务工厂
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      // 创建事务
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      // 根据事务工厂和默认的执行器类型，创建执行器 >>
      final Executor executor = configuration.newExecutor(tx, execType);
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

Executor的基本类型有三种：SIMPLE.BATCH.REUSE,默认SIMPLE,他们都集成了抽象类BaseExecutor.

<img src="/Users/wwei/Library/Application Support/typora-user-images/image-20210426160059908.png" alt="image-20210426160059908" style="zoom:50%;" />

返回DefaultSqlSession,包含configuration,executor.



<img src="/Users/wwei/Library/Application Support/typora-user-images/image-20210426160640292.png" alt="image-20210426160640292" style="zoom:50%;" />

## 获取Mapper对象

```java
TestMapper mapper = sqlSession.getMapper(TestMapper.class);
```

有两个问题需要解决：

1. getMapper获取到的是什么对象？为什么可以执行方法？
2. 怎么根据Mapper找到xml中的sql执行？

```java
@Override
  public <T> T getMapper(Class<T> type) {
    return configuration.getMapper(type, this);
  }

public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    // mapperRegistry中注册的有Mapper的相关信息 在解析映射文件时 调用过addMapper方法
    return mapperRegistry.getMapper(type, sqlSession);
  }

	/**
   * 获取Mapper接口对应的代理对象
   */
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    // 获取Mapper接口对应的 MapperProxyFactory 对象
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```

在解析mapper标签和Mapper.xml的时候已经把接口类型和类型对应的MapperProxyFactory放到了MapperRegistry.knownMappers中。获取Mapper代理对象实际上是从knownMappers获取对应的工厂类。

再通过JDK动态代理创建返回代理对象。

```java
return mapperProxyFactory.newInstance(sqlSession);
```

getMapper返回的是JDK动态代理对象。这个代理对象继承proxy类，里面有MapperProxy implements InvocationHandler 的触发管理类。



#### JDK动态代理

有三个核心角色：被代理类（实现类），接口，实现了InvocationHandler的触发管理类。

被代理类必须实现接口，要通过接口获取方法，而且代理类也要实现这个接口。

<img src="/Users/wwei/Library/Application Support/typora-user-images/image-20210426163342308.png" alt="image-20210426163342308" style="zoom:50%;" />

Mybatis的动态代理

<img src="/Users/wwei/Library/Application Support/typora-user-images/image-20210426163453106.png" alt="image-20210426163453106" style="zoom:50%;" />

#### 总结

​	获取Mapper对象的过程，实际上是获取一个JDK动态代理类。



## 执行sql

```java
mapper.selectAll(map)
```

因为Mapper都是有JDK动态代理对象，所有执行的方法都是执行MapperProxy的invoke()方法。

```java
MapperProxy.invoke();
```

```java
return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
```



1. 首先判断是否需要执行sql，还是执行方法。Object的方法不需要执行sql。

2. 获取缓存。提升MapperMethod的获取速度。

3. 普通方法返回DefaultMethodInvoker，Mapper返回PlainMethodInvoker(new MapperMethod());

   ```java
   public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
     	//封装的是statement id 和sql类型
       this.command = new SqlCommand(config, mapperInterface, method);
     	//封装的是返回值类型
       this.method = new MethodSignature(config, mapperInterface, method);
     }
   ```

4. invoke方法调用了mapperMethod.execute();

   在这个方法中，根据不同的sql类型和返回类型：

   * 调用 method.convertArgsToSqlCommandParam(args) 将方法参数转换为sql参数
   * 调用sqlSession的insert(),update()等，最后走到三种Executor(SimpleExecutor,BatchExecutor,ReuseExecutor)创建出PreparedStatement()执行jdbc保重的方法。

<img src="/Users/wwei/Library/Application Support/typora-user-images/image-20210426170518886.png" alt="image-20210426170518886" style="zoom:50%;" />



## 缓存

```java
    CacheKey cacheKey = new CacheKey();
    // -1381545870:4796102018:com.gupaoedu.mapper.BlogMapper.selectBlogById:0:2147483647:select * from blog where bid = ?:1:development
    cacheKey.update(ms.getId()); // com.gupaoedu.mapper.BlogMapper.selectBlogById
    cacheKey.update(rowBounds.getOffset()); // 0
    cacheKey.update(rowBounds.getLimit()); // 2147483647 = 2^31-1
    cacheKey.update(boundSql.getSql());
		cacheKey.update(value); // development
		cacheKey.update(configuration.getEnvironment().getId());
```

这些值相同，才会认为是同一个查询

1. 处理二级缓存

```java
Cache cache = ms.getCache();
    // cache 对象是在哪里创建的？  XMLMapperBuilder类 xmlconfigurationElement()
    // 由 <cache> 标签决定
    if (cache != null) {
      
    }
```

2. 二级缓存要通过TCM管理

```java
private final TransactionalCacheManager tcm = new TransactionalCacheManager();

if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        // 获取二级缓存
        // 缓存通过 TransactionalCacheManager、TransactionalCache 管理
        @SuppressWarnings("unchecked")
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          // 写入二级缓存
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
```

为什么要用tcm管理：

在一个事务中：

* 插入一条数据（没有提交），二级缓存清空。
* 在这个事务中查询数据，写入二级缓存。
* 提交事务，异常回滚。

这时候出现了数据库没有这条数据，但是二级缓存有这个数据。所以二级缓存要和事务关联起来。

一级缓存：一个session就是一个事务，事务结束缓存清空，不会有脏数据的情况。二级缓存是namespace级别的，跨session。



## Mybatis核心对象

![image-20210426202907355](/Users/wwei/Library/Application Support/typora-user-images/image-20210426202907355.png)

![image-20210426202954044](/Users/wwei/Library/Application Support/typora-user-images/image-20210426202954044.png)

