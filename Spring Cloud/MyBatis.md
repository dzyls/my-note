### MyBatis的核心流程

---

MyBatis的核心流程比较简单 :

1. SqlSessionFactoryBuilder读取配置文件，通过配置文件生成SqlSessionFactory

2. 通过SqlSessionFactory获取SqlSession，SqlSession的作用就是获取Mapper（即Mapper接口的代理），或者获取到MappedStatement后就执行语句了。

   - 获取Mapper

     ```java
     @Override
     public <T> T getMapper(Class<T> type) {
       return configuration.getMapper(type, this);
     }
     //实际使用
     public SysUser findDefaultUser() {
         try (SqlSession session = sessionFactory.openSession()) {
             SysUserMapper mapper = session.getMapper(SysUserMapper.class);
             return mapper.findDefaultUser();
         }
     }
     ```

   - 获取MappedStatement

     ```java
     @Override
     public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
       try {
         // 获取MapperStatement
         MappedStatement ms = configuration.getMappedStatement(statement);
         return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
       } catch (Exception e) {
         throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
       } finally {
         ErrorContext.instance().reset();
       }
     }
     ```

3. 执行完语句，根据是否异常，执行回滚或者提交



### 各种接口的作用

---



#### SqlSessionFactoryBuilder

见名知义，就是创建SqlSessionFactory。

通过读取配置文件，来创建SqlSessionFactory。



#### SqlSessionFactory

见名知义，就是SqlSession，最常用的方法是各种的`openSession`。

```java
SqlSession session = sqlSessionFactory.openSession()
```



#### SqlSession

主要作用有两种 ：

- 获取到Mapper接口的代理子类对象（JDK动态代理），然后手动执行，如下 ：

  ```java
  try(SqlSession session = sqlSessionFactory.openSession()){
      UserMapper userMapper = session.getMapper(UserMapper.class);
      return userMaper.selectByUsername(username);
  }
  ```

  `getMapper`底层调用的是`MapperRegistry`中的`getMapper`。

  ```java
  // org.apache.ibatis.session.defaults.DefaultSqlSession#getMapper
  public <T> T getMapper(Class<T> type) {
      return this.configuration.getMapper(type, this);
  }
  ```

  ```java
  // org.apache.ibatis.session.Configuration#getMapper
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }
  ```

- 直接执行

  ```java
  try(SqlSession session = sqlSessionFactory.openSession()){
      // Mapper文件指定的namespace
      return session.selectOne("com.dzyls.UserMapper.selectByUsername",param);
  }
  ```

第二种方式需要直接写Mapper文件的Namespace，比较长，容易出错。



#### MapperRegistry

`MapperRegistry`的作用是存放MapperProxyFactory，通过MapperProxyFactory可以创建代理对象（即Mapper接口的代理子类），进而执行Mapper接口的方法。

```java
public class MapperRegistry {

  private final Configuration config;
  // knownMappers是一个Map，key是class对象，value是MapperFactoryProxy
  private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<>();

  // 省略部分代码
    
  // 获取mapper的代理子类对象
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      // 容器中有的话，就直接使用反射创建Mapper对象
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }


  public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
        knownMappers.put(type, new MapperProxyFactory<>(type));
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }

	// 省略部分代码

}

```

MapperRegistry就是一个存放MapperProxyFactory的容器，key是class对象，value是MapperProxyFactory。



MapperRegistry是SqlSessionFactoryBuilder.build方法时，就扫描加载的。

```java
  public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
      ......
      XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
      return build(parser.parse());
	  ......
  }
```

```java
public Configuration parse() {
  // 解析 mybatis-config.xml
  parseConfiguration(parser.evalNode("/configuration"));
  return configuration;
}
```



#### MappedStatement

如果通过`sqlSession.selectOne()`这种方式来执行SQL语句，那么最终会调用这个方法 ：

```java
@Override
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
  try {
    // MappedStatement
    MappedStatement ms = configuration.getMappedStatement(statement);
    // executor来执行，后面会介绍
    return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```



而configuration类的`getMappedStatement`方法 ：

```java
protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection")
    .conflictMessageProducer((savedValue, targetValue) ->
        ". please check " + savedValue.getResource() + " and " + targetValue.getResource());
  public MappedStatement getMappedStatement(String id, boolean validateIncompleteStatements) {
    if (validateIncompleteStatements) {
      buildAllStatements();
    }
    return mappedStatements.get(id);
  }
```

可以看到Configuration的`mappedStatements`就是一个map，那么是什么时候将`xml`解析加载到map中的呢？

其实和MapperRegistry一样，也是`SqlSessionFactoryBuild`在读取配置文件，并build`SqlSessionFactory`就执行了加载。

```java
  private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          String mapperPackage = child.getStringAttribute("name");
          configuration.addMappers(mapperPackage);
        } else {
          String resource = child.getStringAttribute("resource");
          String url = child.getStringAttribute("url");
          String mapperClass = child.getStringAttribute("class");
          if (resource != null && url == null && mapperClass == null) {
            ErrorContext.instance().resource(resource);
            InputStream inputStream = Resources.getResourceAsStream(resource);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
            // 重点
            mapperParser.parse();
          } else if (resource == null && url != null && mapperClass == null) {
            ErrorContext.instance().resource(url);
            InputStream inputStream = Resources.getUrlAsStream(url);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url == null && mapperClass != null) {
            Class<?> mapperInterface = Resources.classForName(mapperClass);
            configuration.addMapper(mapperInterface);
          } else {
            throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
          }
        }
      }
    }
  }
```

```java
public void parse() {
  if (!configuration.isResourceLoaded(resource)) {
    // 解析配置文件的mapper节点
    configurationElement(parser.evalNode("/mapper"));
    configuration.addLoadedResource(resource);
    bindMapperForNamespace();
  }

  parsePendingResultMaps();
  parsePendingCacheRefs();
  parsePendingStatements();
}
```

`bindMapperForNamespace`是根据namespace来尝试加载mapper

```java
private void bindMapperForNamespace() {
  String namespace = builderAssistant.getCurrentNamespace();
  if (namespace != null) {
    Class<?> boundType = null;
    try {
      // 根据namespace来尝试加载Mapper。如果namespace写的不是类的全限定名，就会找不到类
      boundType = Resources.classForName(namespace);
    } catch (ClassNotFoundException e) {
      // ignore, bound type is not required
    }
    if (boundType != null && !configuration.hasMapper(boundType)) {
      // Spring may not know the real resource name so we set a flag
      // to prevent loading again this resource from the mapper interface
      // look at MapperAnnotationBuilder#loadXmlResource
      configuration.addLoadedResource("namespace:" + namespace);
      configuration.addMapper(boundType);
    }
  }
}
```

而`configurationElement`则是解析xml，放到Confirguration类中的mappedStatements的map容器中。

```java
private void configurationElement(XNode context) {
  try {
    String namespace = context.getStringAttribute("namespace");
    if (namespace == null || namespace.isEmpty()) {
      throw new BuilderException("Mapper's namespace cannot be empty");
    }
    builderAssistant.setCurrentNamespace(namespace);
    cacheRefElement(context.evalNode("cache-ref"));
    cacheElement(context.evalNode("cache"));
    parameterMapElement(context.evalNodes("/mapper/parameterMap"));
    resultMapElements(context.evalNodes("/mapper/resultMap"));
    sqlElement(context.evalNodes("/mapper/sql"));
    buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
  } catch (Exception e) {
    throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
  }
}
```