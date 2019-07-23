## 前言

Mybatis参数处理是Mybatis核心内容，围绕着Mybatis的面试题也是层出不穷。接下来跟随源码看下Mybatis是如何处理参数的。

## 代码示例

### Mapper

```java
LoanApplicationEntity getByLoanAppCode(@Param("loanAppCode") String loanAppCode);
```

### XML

```xml
<select id="getByLoanAppCode" resultMap="BaseResultMap">  
    SELECT  <include refid="Base_Column_List"/>
    FROM szb_loan_application
    WHERE entry_code =#{loanAppCode} AND deleted =0
</select>
```

### JunitTest

```java

@ActiveProfiles("dev")
@SpringBootTest
@RunWith(SpringRunner.class)
public class MybatisTest {

    //这里注入的实际上是一个代理类
    @Autowired
    private LoanApplicationMapper loanApplicationMapper;

    @Test
    public void testMybatis(){
        LoanApplicationEntity applicationEntity = loanApplicationMapper.getByLoanAppCode("w1111");
        System.out.println(applicationEntity);
    }
}
```

- 这里注入的实际上是一个代理类，这个代理类是在应用启动的时候spring发现其他bean注入了这个类，就通过BeanFactory.getBean()，再通过FactoryBean（MapperFactoryBean）.getObject()，最后通过动态代理注入得到。Mybatis参数处理

idea debug进入下一步，可以发现进入了MapperProxy的invoke方法。

![1563851442449](https://raw.githubusercontent.com/wangxiaowu241/blogs/master/Mapper%23invoke.png)

```java
@Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      //这里的method.getDeclaringClass()的值是com.xt.algorithm.mapper.LoanApplicationMapper
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        //isDefaultMethod(method)返回false
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    //将MapperMethod缓存起来 
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    //最后执行mapperMethos.execute()方法
    return mapperMethod.execute(sqlSession, args);
  }

private MapperMethod cachedMapperMethod(Method method) {
    return methodCache.computeIfAbsent(method, k -> new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
  }
```

接下来看下MapperMethod.execute()方法是如何处理参数的。

```java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
      case INSERT: {
    	Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
          if (method.returnsOptional() &&
              (result == null || !method.getReturnType().equals(result.getClass()))) {
            result = Optional.ofNullable(result);
          }
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    //...
    return result;
  }
```

可以看到对于参数处理，都是通过Object param = method.convertArgsToSqlCommandParam(args);去处理的，那么我们看下这个方法到底做了什么操作。

```java
public Object convertArgsToSqlCommandParam(Object[] args) {
      return paramNameResolver.getNamedParams(args);
}

public Object getNamedParams(Object[] args) {
    final int paramCount = names.size();
    if (args == null || paramCount == 0) {
      //如果没有入参，或者方法定义参数个数为0，直接返回null
      return null;
    } else if (!hasParamAnnotation && paramCount == 1) {
      //如果没有使用@Param注解，且参数个数为1个，直接返回入参
      return args[names.firstKey()];
    } else {
      //否则，遍历方法names
      final Map<String, Object> param = new ParamMap<>();
      int i = 0;
      for (Map.Entry<Integer, String> entry : names.entrySet()) {
        //这里将names的键值对放入param中
        param.put(entry.getValue(), args[entry.getKey()]);
        // add generic param names (param1, param2, ...)
        //并添加{"param1":entay.getKey()}形式放入param中
        final String genericParamName = GENERIC_NAME_PREFIX + String.valueOf(i + 1);
        // ensure not to overwrite parameter named with @Param
        if (!names.containsValue(genericParamName)) {
          param.put(genericParamName, args[entry.getKey()]);
        }
        i++;
      }
      return param;
    }
  }
```

![1563853878946](https://raw.githubusercontent.com/wangxiaowu241/blogs/master/ParamNamesResolver%23getNamedParams.png)

可以看到ParamNameResolver.getNamedParams()方法的入参args就是mapper接口上方法值。

![1563853957095](https://raw.githubusercontent.com/wangxiaowu241/blogs/master/names.png)

names是一个SortedMap，内部的键值对，key为参数在接口方法中的索引位置（方法入参中的第几个参数，从0开始），value为@Param的value值（如果没有使用@Param注解，默认为arg0,arg1...）。

这一部分可从ParamNameResolver的构造函数中看出。

```java
public ParamNameResolver(Configuration config, Method method) {
    //获取方法参数类型
    final Class<?>[] paramTypes = method.getParameterTypes();
    //获取方法参数上的注解
    final Annotation[][] paramAnnotations = method.getParameterAnnotations();
    final SortedMap<Integer, String> map = new TreeMap<>();
    int paramCount = paramAnnotations.length;
    // get names from @Param annotations
    for (int paramIndex = 0; paramIndex < paramCount; paramIndex++) {
      //从@Param注解上获取value属性值，并给name字段赋值
      String name = null;
      for (Annotation annotation : paramAnnotations[paramIndex]) {
        if (annotation instanceof Param) {
          hasParamAnnotation = true;
          name = ((Param) annotation).value();
          break;
        }
      }
      if (name == null) {
        // @Param was not specified.
        //如果没参数没使用@Param注解
        if (config.isUseActualParamName()) {
          //从method中取出参数名称，一般为arg0,arg1 ...
          name = getActualParamName(method, paramIndex);
        }
        if (name == null) {
          // use the parameter index as the name ("0", "1", ...)
          // gcode issue #71
          //如果前面几个操作给name赋值都失败了，最后使用下标作为键值对的value  
          name = String.valueOf(map.size());
        }
      }
      //key为参数下标，value为@Param注解value值或者mybatis指定默认值
      map.put(paramIndex, name);
    }
    names = Collections.unmodifiableSortedMap(map);
  }
```

#### names键值对总结

从上述构造方法可以看出，names中的键值对应该是{"0","paramValue"}或者{"1":"arg1"}这样。

getNamedParams方法返回的map中的键值对应该是{"paramValue":"0"}或者{"param1":"1"}这样。

其中：paramValue是指@Param注解的value属性。param1是mybatis通用的参数key。

#### getNamedParams返回的数据类型有以下几种：

- null：mapper方法中没定义参数或者入参为null。
- 除了map、null以外的其他Object类型，包括基本数据类型和java 对象：当入参中仅有一个参数，而且没有使用@Param注解时。
- map：使用了@Param注解或者mapper方法入参不止一个。

#### SELECT方法中的参数继续处理

SELECT类型的方法最后都在SqlSession的slectList方法中进行统一处理。

```java
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      //先根据statement从configuration中获取MappedStatement
      //这里的statement就是mapper接口名.方法名
      //String statementId = mapperInterface.getName() + "." + methodName;
      MappedStatement ms = configuration.getMappedStatement(statement); 
      //这里的wrapCollection对方法又进行了一层包装
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

这里有必要说明一下方法的入参：

- statement：就是statementId：mapper接口名.方法名。详见org.apache.ibatis.binding.MapperMethod.SqlCommand#resolveMappedStatement
- parameter：就是前面getNamedParams方法返回的数据，可能是null，map以及其他object类型的数据。
- rowBounds：分页相关的数据，这里是默认的rowBounds，不分页。

```java
private Object wrapCollection(final Object object) {
    //在对selectList方法入参进行包装前，先判断参数类型
    if (object instanceof Collection) {
      //这里判断了是不是collection类型，如果是则在外面使用map包一层，key为collection，value为入参值。这里仅当getNamedParams返回的是Object类型时才可能进入，就是说mapper方法的入参只有一个，而且没有使用@Param注解
      StrictMap<Object> map = new StrictMap<>();
      map.put("collection", object);
      if (object instanceof List) {
        //这里再次判断是否是List子类型，如果是的话，再添加一个key为"list"的键值对，方便动态SQL中的<foreach>等使用
        map.put("list", object);
      }
      return map;
    } else if (object != null && object.getClass().isArray()) {
      //如果是数组的话，也会用map包装一层，key为"array"  
      StrictMap<Object> map = new StrictMap<>();
      map.put("array", object);
      return map;
    }
    return object;
  }
```

总的来说，wrapCollection方法就是对getNamedParams处理后的参数再次进行处理，如果是数组或者Collection对象，则在外面用map包装一层，方便后续的动态SQL使用参数。

紧接着就到了SimpleExecutor的doQuery方法了

```java
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      //这里的StatementHandler默认是RoutingStatementHandler，被代理类是PreparedStatementHandler
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      //调用内部私有方法
      stmt = prepareStatement(handler, ms.getStatementLog());
      //查询
      return handler.query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }

private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    //底层数据库服务获取数据库连接connection
    Connection connection = getConnection(statementLog);
    //调用底层connection的prepareStatement方法预编译SQL
    stmt = handler.prepare(connection, transaction.getTimeout());
    //handler 参数化
    handler.parameterize(stmt);
    return stmt;
  }
```

DefaultParameterHandler.setParameters()方法

```java
public void setParameters(PreparedStatement ps) {
    ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings != null) {
      //遍历 ParameterMapping，ParameterMapping中包含属性，javaType、jdbcType等 
      for (int i = 0; i < parameterMappings.size(); i++) {
        ParameterMapping parameterMapping = parameterMappings.get(i);
        if (parameterMapping.getMode() != ParameterMode.OUT) {
          Object value;
          //SQL中参数名，#{参数名}
          String propertyName = parameterMapping.getProperty();
          if (boundSql.hasAdditionalParameter(propertyName)) {
              //动态SQL时，解析时会自动假如其他的参数值
              // issue #448 ask first for additional params
            value = boundSql.getAdditionalParameter(propertyName);
          } else if (parameterObject == null) {
            //如果mapper方法的入参parameterObject为空，则直接返回null
            value = null;
          } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
            //如果parameterObject是简单基本类型的话，则value直接等于parameterObject
            value = parameterObject;
          } else {
            //如果parameterObject是map或者java bean等复杂类型的话，构造MetaObject，方便通过属性或者多层嵌套（如user.name）取值
            MetaObject metaObject = configuration.newMetaObject(parameterObject);
            value = metaObject.getValue(propertyName);
          }
          TypeHandler typeHandler = parameterMapping.getTypeHandler();
          JdbcType jdbcType = parameterMapping.getJdbcType();
          if (value == null && jdbcType == null) {
            jdbcType = configuration.getJdbcTypeForNull();
          }
          try {
            //通过typehandler set参数值到SQL中
            typeHandler.setParameter(ps, i + 1, value, jdbcType);
          } catch (TypeException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          } catch (SQLException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          }
        }
      }
    }
  }
```

紧接着就是PreparedStatementHandler的query方法。

```java
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    //这里直接调用execute方法，最后通过数据库底层驱动（如mysql）的PreparedStatement实现类完成execute方法。执行SQL
    ps.execute();
    return resultSetHandler.handleResultSets(ps);
  }
```

