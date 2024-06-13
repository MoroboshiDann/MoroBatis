# MoroBatis

手动实现Mybatis框架，摸清“为什么在使用Mybatis的时候，只需定义一个Dao接口，不用写实现类就能使用XML中或者注解上配置好的SQL语句，就能完成对数据库CRUD的操作呢？”这样的问题。

Mybatis的作用：

不使用Mybatis之前，需要通过JDBC来手动连接数据库，Java代码和SQL代码耦合在一起，不利于后期修改维护。同时Mybatis免除了JDBC参数设置和获取结果集的工作。

> 代码执行流程：
> 启动阶段：
> - XMLConfigBuilder读取`datasource.xml`，解析数据源信息。根据`<mappers>`标签给出的路径为每一个`Mapper.xml`调用XMLMapperBuilder
>   - 在该类型的构造器中，会实例化一个`Configuration`对象，用来存储数据源和SQL相关的所有信息。一个程序只会有一个Configuration对象。
>   - Configuration在构造器中就会实例化一个`LanguageDriver`，用于后续处理SQL语句内容。
> - XMLMapperBuilder读取`namespace`属性，并注册Mapper。为每一个`<select>`等SQL语句标签调用XMLStatementBuilder
>   - 注册Mapper，通过MapperRegistry提供的API，根据Dao接口的`Class`对象，注册代理工厂，生成Mapper代理对象。
> - XMLStatementBuilder读取SQL语句的ID、入参类型、结果类型并记录。调用语言驱动器LanguageDriver来解析SQL语句具体内容，生成SqlSource对象。
>   - LanguageDriver在Configuration对象被实例化时一起被实例化，并存储在Configuration对象中。
>   - LanguageDriver在处理SQL语句时需要入参类型`Class<?> parameterType`，于是根据SQL标签中的`parameterType`属性，来将入参对应的Java类型Class对象加载出来。
> - LanguageDriver为当前SQL语句实例化一个`XMLScriptBuilder`对象，调用其`parseScriptNode()`方法来处理SQL语句内容。
> - `XMLScriptBuilder#parseScriptNode`会将SQL每一个标签(if、where等)实例化为一个`StaticTextSqlNode`对象，然后将所有静态文本节点对象存储在一个List集合中，封装到`MixedSqlNode`对象中。然后再使用混合节点对象来实例化`RawSqlSource`对象。
> - RawSqlSource在构造器方法中，会调用`getSql()`方法。实例化一个`DynamicContext`对象，并将MixedSqlNode中所有的静态SQL作为字符串拼接起来，存储在内部。
>   - 通过DynamicContext，RawSqlSource就获得了一个以字符串形式表示的SQL语句，其中的参数都以`#{id}, ${user.id}`的形式表示着。
> - 接着RawSqlSource实例化一个`SqlSourceBuilder`对象，让其对SQL字符串进行处理，主要是处理参数，并返回一个`StaticSqlSource`对象。
> - SqlSourceBuilder会实例化一个`GenericTokenParser`对象，调用其`parse()`方法来对SQL字符串的参数进行处理。
> - 在`GenericTokenParser#parse`方法中，对于每一个以`#{}`表示的参数，会调用`ParameterMappingTokenHandler#handleToken`方法来处理。
> - `ParameterMappingTokenHandler#handleToken`中，将当前的参数用`?`来占位，并调用`buildParameterMapping()`方法来构建参数映射。
> - `builderParameterMapping()`方法中，将当前参数根据参数名和从一开始就传入的JavaType，转换为一个`ParameterMapping`对象。
>   - 由于ParameterMappingTokenHandler是SqlSourceBuilder的内部类，而SqlSourceBuilder一个实例只处理一个SQL语句，所以在该类中通过List集合记录下每次调用`handleToken()`方法产生的ParameterMapping对象，就可以按照顺序存储一条SQL语句每个`?`处应该填入的变量名、JdbcType和JavaType。
> - SqlSourceBuilder处理完一条SQL语句后，会将处理完生成的SQL语句(变量用`?`占位的语句)，以及参数映射表都封装进一个StaticSqlSource对象返回。
> - 最终，一路返回，直到XMLStatementBuilder拿到SqlSource实例(拿到的是RawSqlSource，而其内部封装了一个StaticSqlSource对象，所以最终每一条SQL都会封装为一个StaticSqlSource对象)。
>   - StaticSqlSource提供了`getBoundSql()`方法，会实例化一个BoundSql对象返回。相当于在运行SQL语句时，会生成一个BoundSql对象，而不是直接拿着SqlSource对象使用。
> - XMLStatementBuilder再将SqlSource(SQL语句的本体)和解析出来的ID、语句类型、结果类型等信息一起封装到一个`MappedStatement`对象中。
> - 至此，`datasoure.xml`和所有的`Mapper.xml`文件的扫描解析就完成了。所有的结果都存储在一个Configuration对象中。


> 执行SQL：
> - 程序调用`Dao#selectOne`并传入了一个Long类型参数。
> - Mapper代理对象通过反射，拦截到了方法的执行，转而执行`MapperProxy#invoke`方法。
> - `invoke()`方法中，先检查该方法是已经在Dao接口的某个实现类中有了实现，有了就直接调用该方法，如果没有就执行自己的逻辑。
> - `invoke()`方法，检查当前执行方法是否已经解析生成了`MapperMethod`实例，如果没有就解析生成一个并记录。
> - `invoke()`方法中，拿到方法对应的MapperMethod实例，调用`MapperMethod#execute`方法。
> - `MapperMethod#execute`中，先将传入参数进行转化处理，将数组类型转化为Object(可能对应一个对象，也可能对应一个Map集合)，然后查看方法类型(select，update，delete等)，调用对应的`SqlSession#selectOne`方法。
>   - 由于`invoke()`方法的入参为`Object[] args`数组类型，所以解析当前方法签名，将方法传入参数，记录在一个Map集合中。key为参数序号`0, 1, 2`，value为对应传入的对象。
> - `SqlSession#selectOne`入参为SQL语句的编号和SQL语句对应参数的集合(此时参数还是Java的对象)。根据SQL编号从Configuration中拿到对应的MappedStatement对象，然后根据MS对象中封装的SqlSource生成一个BoundSql对象。最后，将MS对象、入参集合、结果处理器和BoundSql对象一起，调用`Executor#query`方法。
> - Executor是一个接口，所以实际调用的是其实现类方法`doQuery()`。在其中，执行查询数据库的逻辑。
> - 首先，根据方法的入参，实例化一个`StatementHandler`对象，用于处理SQL语句(此时SQL语句中的参数还是使用`?`占位符，入参也还是Java类型的)。
>   - StatementHandler内部会实例化一个`ParameterHanlder`对象，用处处理SQL语句中的参数传入问题。
> - 然后，通过事务打开连接Connection，并通过StatementHandler来初始化连接信息，获得一个`java.sql.Statement`对象。(JDBC基本流程)。
> - 接着，通过`StatementHandler#parameterize`方法调用`ParameterHandler#setParameters`方法，来将`?`处的参数传入。
>   - 在上述方法中，根据启动阶段解析出来的ParameterMapping集合，为对应位置的`?`，将Java类型的参数转换为Jdbc类型的。然后调用`TypeHandler#setParameter`方法，执行替换。
>   - 由于在该方法中是直接遍历处理ParameterMappings集合，所以每个参数都会执行一次`setParameter()`方法。
>   - 由于参数类型很多，所以才抽取出来一个TypeHandler接口约束功能，每个类型对应一个实现类。同时在方法调用时也能通过多态的方法节约代码量，方便后期维护。
> - 接着回到`doQuery()`中，SQL语句中的参数已经传入，可以直接执行数据库操作。然后，将返回的结果通过ResultHandler处理，返回给调用处。
> - 最终一路返回，`Dao#selectOne`方法拿到了从数据库查询出来的值。


## 第一章 创建简单的映射器代理工厂

首先，为什么Mybatis框架中，能够只定义一个接口，其内部只进行方法声明而不实现方法，就可以通过XML文件来执行SQL语句，与数据库进行操作呢。

Mybatis就是通过反射和代理的方式，为我们定的接口实现了一个代理类Mapper，代理类来实现该接口。从而，可以通过Mapper对象来调用接口中定义的方法，去实现数据库的CRUD操作。

JDK代理原理：

JDK提供的`java.lang.reflect.Proxy`类中，有`newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)`可以为传入的接口创建一个代理类实例。创建时，需要传入一个`InvocationHandler`接口的实现类对象，该实现类对象需要重写`invoke()`方法。

`Proxy#newInstance`方法返回的代理类对象实现了被代理的接口，因此，调用接口的方法，会调用代理对象中的方法。而代理对象则会调用`InvocationHandler#invoke`方法，将被调用方法和传入参数传给`invoke()`。相当于在`invoke()`方法中执行具体被调用方的逻辑。

### MapperProxy

于是，我们先创建一个代理类`MapperProxy`实现InvocationHandler接口，重写`invoke()`方法，在其中实现Dao接口中方法的具体执行逻辑。

```java
public class MapperProxy<T> implements InvocationHandler, Serializable {
    private static final long serialVersionUID = -6424540398559729838L;
    private Map<String, String> sqlSession;
    private final Class<T> mapperInterface;

    public MapperProxy(Map<String, String> sqlSession, Class<T> mapperInterface) {
        this.sqlSession = sqlSession;
        this.mapperInterface = mapperInterface;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, args); // 接口中方法的实现
        } else {
            return "你的被代理了！" + sqlSession.get(mapperInterface.getName() + "." + method.getName());
        }
    }
}
```

### MapperProxyFactory

然后，再使用工厂模式，创建一个`MapperProxyFactory`代理工厂，通过动态代理(`java.lang.reflect.Proxy`)的方式，为Dao接口生成代理对象。该代理对象实现了Dao接口，于是可以直接通过代理对象来调用Dao接口中的方法。当调用Dao接口方法时，实际上执行的是上文提到的`MapperProxy#invoke`方法。

由于，一个Dao接口只会生成一个代理对象，且代理对象实现了Dao接口，通过接口就可以获取到代理对象，就像映射一般。所以，我们称代理对象为Mapper对象。

```java
public class MapperProxyFactory<T> {
    private final Class<T> mapperInterface;

    public MapperProxyFactory(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }

    public T newInstance(Map<String, String> sqlSession) { // Java提供的API
        final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface);
        return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[]{mapperInterface}, mapperProxy);
    }
}
```

### 映射关系

一个Dao接口 -> 一个Mapper对象 -> MapperProxy工厂 -> MapperProxy对象


## 第二章 实现映射器的注册和使用

第一章的讲解中，可以看出Dao接口、Mapper对象、MapperProxy对象、MapperProxy工厂都是一一对应的，创建MapperProxyFactory工厂对象时就需要以参数的形式告知要生成哪个Dao接口的Mapper对象。

那么，如何能够将这种一一对应的关系绑定起来呢。本章就实现一个映射器注册机来将所有的Dao接口何其对应的MapperProxyFactory绑定起来。只要Dao接口和工厂对象绑定起来，其余的关系也就绑定起来了。

在绑定Dao接口和工厂对象时，不能采用硬编码的方式。参考SpringMVC的做法，在项目启动时，用户只需要指定路径即可完成对需要代理的Dao的扫描和注册。于是，我们需要对映射器的注册提供注册器来处理，同时还需要对SqlSession进行规范化处理，让它可以将我们的映射器代理和方法调用进行包装，建立一个生命周期模型，方便后期扩展。

经上述分析，我们需要一个注册器类，它可以扫描包路径完成Dao接口与代理类的映射。以及一个SqlSession接口的定义。

### MapperRegistry映射器注册器类

注册器类，其功能顾名思义就是将Dao接口与代理工厂一一对应起来。则其应该实现如下方法：
- `getMapper`：根据Dao接口，返回对应的代理类实例。
- `addMapper`：存储Dao接口与其代理工厂的映射关系，完成注册。
- `hasMapper`：根据Dao接口，查询是否已经注册了与代理工厂的映射关系。

绑定Dao接口和工厂对象，就是通过Map集合来记录二者的映射关系即可。

```java
public class MapperRegistry {
    /**
     * 将已添加的映射器代理加入到 HashMap
     */
    private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<>();

    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
        if (mapperProxyFactory == null) {
            throw new RuntimeException("Type " + type + " is not known to the MapperRegistry.");
        }
        try {
            return mapperProxyFactory.newInstance(sqlSession);
        } catch (Exception e) {
            throw new RuntimeException("Error getting mapper instance. Cause: " + e, e);
        }
    }

    public <T> void addMapper(Class<T> type) {
        /* Mapper 必须是接口才会注册 */
        if (type.isInterface()) {
            if (hasMapper(type)) {
                // 如果重复添加了，报错
                throw new RuntimeException("Type " + type + " is already known to the MapperRegistry.");
            }
            // 注册映射器代理工厂
            knownMappers.put(type, new MapperProxyFactory<>(type));
        }
    }

    public <T> boolean hasMapper(Class<T> type) {
        return knownMappers.containsKey(type);
    }

    public void addMappers(String packageName) {
        Set<Class<?>> mapperSet = ClassScanner.scanPackage(packageName);
        for (Class<?> mapperClass : mapperSet) {
            addMapper(mapperClass);
        }
    }
}
```

### SqlSession规范

SqlSession即SQL会话，代表Java程序和数据库的一次会话。在Java程序中，一个Dao接口往往代表的是对数据库中一个表的操作，而一个会话中可能会操作任意一张表，也可能会操作多张表。因此，SqlSession中需要一个MapperRegistry实例，来通过Dao接口获取对应的Mapper对象，以在会话中操作对应的数据库表。

目前，先拿最简单的`select`语句作为基础功能。SqlSession应该提供如下功能：
- `getMapper`：根据传入的Dao接口类型，返回对应的Mapper实例。
- `selectOne`：传入参数为`String statement`，SQL语句的ID。方法应该根据该ID，执行对应的SQL，返回数据库中的一条记录。

```java
public interface SqlSession {

    /**
     * Retrieve a single row mapped from the statement key
     * 根据指定的SqlID获取一条记录的封装对象
     *
     * @param <T>       the returned object type 封装之后的对象类型
     * @param statement sqlID
     * @return Mapped object 封装之后的对象
     */
    <T> T selectOne(String statement);

    /**
     * Retrieve a single row mapped from the statement key and parameter.
     * 根据指定的SqlID获取一条记录的封装对象，只不过这个方法容许我们可以给sql传递一些参数
     * 一般在实际使用中，这个参数传递的是pojo，或者Map或者ImmutableMap
     *
     * @param <T>       the returned object type
     * @param statement Unique identifier matching the statement to use.
     * @param parameter A parameter object to pass to the statement.
     * @return Mapped object
     */
    <T> T selectOne(String statement, Object parameter);

    /**
     * Retrieves a mapper.
     * 得到映射器，这个巧妙的使用了泛型，使得类型安全
     *
     * @param <T>  the mapper type
     * @param type Mapper interface class
     * @return a mapper bound to this SqlSession
     */
    <T> T getMapper(Class<T> type);
}
```

对于SqlSession，我们要用工厂模式将其封装起来，于是定义`SqlSessionFactory`。工厂模式的主要功能就是返回SqlSession对象实例。

```java
public interface SqlSessionFactory {
    /**
     * 打开一个 session
     * @return SqlSession
     */
   SqlSession openSession();
}
```

### SqlSession和SqlSessionFactory的简单实现

在这里，先提供一种默认实现类。

在Java程序中，一个Dao接口往往代表的是对数据库中一个表的操作，而一个会话中可能会操作任意一张表，也可能会操作多张表。因此，SqlSession中需要一个MapperRegistry实例，来通过Dao接口获取对应的Mapper对象，以在会话中操作对应的数据库表。

```java
public class DefaultSqlSession implements SqlSession {
    // 注册映射机
    private MapperRegistry mapperRegistry;

    public DefaultSqlSession(MapperRegistry mapperRegistry) {
        this.mapperRegistry = mapperRegistry;
    }

    @Override
    public <T> T selectOne(String statement) {
        return (T) ("你被代理了！" + statement);
    }

    @Override
    public <T> T selectOne(String statement, Object parameter) {
        return (T) ("你被代理了！" + "方法：" + statement + " 入参：" + parameter);
    }

    @Override
    public <T> T getMapper(Class<T> type) {
        return mapperRegistry.getMapper(type, this);
    }
}
```

而SqlSessionFactory的实现，就比较简单，无需讲解。

```java
public class DefaultSqlSessionFactory implements SqlSessionFactory {
    private final MapperRegistry mapperRegistry;

    public DefaultSqlSessionFactory(MapperRegistry mapperRegistry) {
        this.mapperRegistry = mapperRegistry;
    }

    @Override
    public SqlSession openSession() {
        return new DefaultSqlSession(mapperRegistry);
    }
}
```

同时，第一章提到的代理类和代理工厂，也要做出修改，不多赘述。


## 第三章 Mapper XML的解析和注册使用

Mybatis中，Dao接口中的每一个方法，都会对应着一条SQL语句。并且，将一个Dao接口中所有对应的SQL都存放在一个`xxxMapper.xml`文件中。所有的Mapper XML文件的路径都存放在配置文件的`<mappers>`标签中。

在第二章，我们已经通过MapperRegistry类来扫描包路径对Dao接口进行注册。本章节，我们就解析所有的`xxxMapper.xml`文件来获取将命名空间、SQL描述、映射信息。

所以，本章的重点就是如何解析XML文件。在本章节只对`<mappers>`标签中指定的Mapper XML文件路径进行解析，获取其中的SQL信息。

### SqlSessionFactoryBuilder

首先，需要定义一个`SqlSessionFactoryBuilder`工厂建造者模式类，作为整个MoroBatis的入口。通过入口I/O的方式，对配置XML文件进行解析。在目前阶段，主要解析SQL语句，并注册Mapper，串联出整个核心流程的脉络。

```java
public class SqlSessionFactoryBuilder {
    public SqlSessionFactory build(Reader reader) {
        XMLConfigBuilder xmlConfigBuilder = new XMLConfigBuilder(reader);
        return build(xmlConfigBuilder.parse());
    }

    public SqlSessionFactory build(Configuration config) {
        return new DefaultSqlSessionFactory(config);
    }
}
```

在代码中，使用I/O创建了一个`XMLConfigBuilder`对象，其内部会对配置XML文件进行解析，将命名空间、SQL描述、资源等等信息解析出来，放置在`Configuration`对象中。然后调用`XMLConfigBuilder.parse()`方法返回这个Configuration对象。

SqlSessionFactoryBuilder再利用获取的Configuration对象来创建SqlSessionFactory工厂对象并返回。

整个程序只会有一个SqlSessionFactoryBuilder对象，因此配置XML文件只会被解析一次，同时也只会有一个SqlSessionFactory对象。由于只解析一次XML，因此整个程序也只会有一个Configuration对象。

### XMLConfigBuilder

上一章节中，我们在MapperRegistry中实现了通过指定包路径来扫描注册Dao接口的方法，由于XML文件中的`namespace`属性就已经指明了每一个XML文件对应的Dao接口位置，所以，我们可以直接在解析XML文件的时候实现对应Dao接口的注册。

```java
public class XMLConfigBuilder extends BaseBuilder {
    private Element root;

    public XMLConfigBuilder(Reader reader) {
        // 1. 调用父类初始化Configuration
        super(new Configuration());
        // 2. dom4j 处理 xml
        SAXReader saxReader = new SAXReader();
        try {
            Document document = saxReader.read(new InputSource(reader));
            root = document.getRootElement();
        } catch (DocumentException e) {
            e.printStackTrace();
        }
    }

    public Configuration parse() {
        try {
            // 解析映射器
            mapperElement(root.element("mappers"));
        } catch (Exception e) {
            throw new RuntimeException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
        }
        return configuration;
    }

    private void mapperElement(Element mappers) throws Exception {
        List<Element> mapperList = mappers.elements("mapper");
        for (Element e : mapperList) {
            String resource = e.attributeValue("resource");
            Reader reader = Resources.getResourceAsReader(resource);
            SAXReader saxReader = new SAXReader();
            Document document = saxReader.read(new InputSource(reader));
            Element root = document.getRootElement();
            //命名空间
            String namespace = root.attributeValue("namespace");

            // SELECT
            List<Element> selectNodes = root.elements("select");
            for (Element node : selectNodes) {
                String id = node.attributeValue("id");
                String parameterType = node.attributeValue("parameterType");
                String resultType = node.attributeValue("resultType");
                String sql = node.getText();

                // ? 匹配
                Map<Integer, String> parameter = new HashMap<>();
                Pattern pattern = Pattern.compile("(#\\{(.*?)})");
                Matcher matcher = pattern.matcher(sql);
                for (int i = 1; matcher.find(); i++) {
                    String g1 = matcher.group(1);
                    String g2 = matcher.group(2);
                    parameter.put(i, g2);
                    sql = sql.replace(g1, "?");
                }

                String msId = namespace + "." + id;
                String nodeName = node.getName();
                SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
                MappedStatement mappedStatement = new MappedStatement.Builder(configuration, msId, sqlCommandType, parameterType, resultType, sql, parameter).build();
                // 添加解析 SQL
                configuration.addMappedStatement(mappedStatement);
            }

            // 注册Mapper映射器
            configuration.addMapper(Resources.classForName(namespace));
        }
    }
}
```

读取XML文件的标签，然后解析信息，存放在Configuration对象中。最后，根据`namespace`注册Mapper，生成代理对象。

### Configuration

Configuration类是配置类，整个程序只会有一个实例。它将记录整个程序中关于数据库操作的所有信息，如：SQL语句、Mapper对象、数据库信息、数据源等。

其对MapperRegistry的功能再做了一次封装。配置类将XMLConfigBuilder解析出来的XML文件的信息存储在内部。(其实，Configuration将其存储的各个不部分的信息都做了封装，直接操作Configuration对象就能获取到各种信息)。

具体在使用MoroBatis时，通过`Reader`I/O对象，指定XML文件，然后通过SqlSessionFactoryBuilder根据Reader对象，解析XML文件，返回SqlSessionFactory工厂对象。然后，再根据工厂对象获取SqlSession对象。有了SqlSession对象，就可以根据Dao接口来获取Mapper代理对象，去使用Mapper中的方法与数据库进行交互。

```java
public class Configuration {
    /**
     * 映射注册机
     */
    protected MapperRegistry mapperRegistry = new MapperRegistry(this);
    /**
     * 映射的语句，存在Map里
     */
    protected final Map<String, MappedStatement> mappedStatements = new HashMap<>();

    public void addMappers(String packageName) {
        mapperRegistry.addMappers(packageName);
    }

    public <T> void addMapper(Class<T> type) {
        mapperRegistry.addMapper(type);
    }

    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        return mapperRegistry.getMapper(type, sqlSession);
    }

    public boolean hasMapper(Class<?> type) {
        return mapperRegistry.hasMapper(type);
    }

    public void addMappedStatement(MappedStatement ms) {
        mappedStatements.put(ms.getId(), ms);
    }

    public MappedStatement getMappedStatement(String id) {
        return mappedStatements.get(id);
    }
}
```
### MappedStatement

另外，本章新增一个`MappedStatement`对象，用于记录SQL信息，包括：SQL类型，SQL语句，入参类型，出参类型等。

在解析XML文件时，会为每一个SQL语句创建一个该对象，并将该对象存储在Configuration类对象中。

```java
public class MappedStatement {
    private Configuration configuration;
    private String id;
    private SqlCommandType sqlCommandType;

    private String parameterType;
    private String resultType;
    private String sql;
    private Map<Integer, String> parameter;

    MappedStatement() {
        // constructor disabled
    }
    /**
     * 建造者
     */
    public static class Builder {

        private MappedStatement mappedStatement = new MappedStatement();

        public Builder(Configuration configuration, String id, SqlCommandType sqlCommandType, String parameterType, String resultType, String sql, Map<Integer, String> parameter) {
            mappedStatement.configuration = configuration;
            mappedStatement.id = id;
            mappedStatement.sqlCommandType = sqlCommandType;
            mappedStatement.parameterType = parameterType;
            mappedStatement.resultType = resultType;
            mappedStatement.sql = sql;
            mappedStatement.parameter = parameter;
        }

        public MappedStatement build() {
            assert mappedStatement.configuration != null;
            assert mappedStatement.id != null;
            return mappedStatement;
        }

    }
}
```

### MappedMethod

对于Dao接口中的每一个方法，其对应着一条SQL语句，为了执行方便，为每个方法都生成一个`MappedMethod`实例，用来管理SQL执行相关的操作。

在`MappedMethod#execute`方法中，会根据SQL的类型来执行相应的逻辑。

```java
public class MapperMethod {
    private final SqlCommand command;

    public MapperMethod(Class<?> mapperInterface, Method method, Configuration configuration) {
        this.command = new SqlCommand(configuration, mapperInterface, method);
    }

    public Object execute(SqlSession sqlSession, Object[] args) {
        Object result = null;
        switch (command.getType()) {
            case INSERT:
                break;
            case DELETE:
                break;
            case UPDATE:
                break;
            case SELECT:
                result = sqlSession.selectOne(command.getName(), args);
                break;
            default:
                throw new RuntimeException("Unknown execution method for: " + command.getName());
        }
        return result;
    }
    /**
     * SQL 指令
     */
    public static class SqlCommand {

        private final String name;
        private final SqlCommandType type;

        public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
            String statementName = mapperInterface.getName() + "." + method.getName();
            MappedStatement ms = configuration.getMappedStatement(statementName);
            name = ms.getId();
            type = ms.getSqlCommandType();
        }

        public String getName() {
            return name;
        }

        public SqlCommandType getType() {
            return type;
        }
    }
}
```

### DefaultSqlSession 结合配置类获取信息

由于Configuration已经对MapperRegistry做了二次封装，所以将DefaultSqlSession中的MapperRegistry替换为Configuration，这样可以传递更加丰富的消息。`DefaultSqlSession.getMapper()`方法中，就是通过Configuration获取的信息。

```java
public class DefaultSqlSession implements SqlSession {
    private Configuration configuration;

    public DefaultSqlSession(Configuration configuration) {
        this.configuration = configuration;
    }

    @Override
    public <T> T selectOne(String statement) {
        return (T) ("你被代理了！" + statement);
    }

    @Override
    public <T> T selectOne(String statement, Object parameter) {
        MappedStatement mappedStatement = configuration.getMappedStatement(statement);
        return (T) ("你被代理了！" + "\n方法：" + statement + "\n入参：" + parameter + "\n待执行SQL：" + mappedStatement.getSql());
    }

    @Override
    public <T> T getMapper(Class<T> type) {
        return configuration.getMapper(type, this);
    }

    @Override
    public Configuration getConfiguration() {
        return configuration;
    }
}
```

## 第四章 数据源的解析创建和使用

通过JDBC来连接数据库时，需要先注册JDBC驱动。然后，每一次交互，都需要打开一个连接Connection。通过该连接来操作数据库。操作结束后还需要关闭连接。

数据源`dataSource`，就是要先维护一个数据库连接池Connection Pool，里面有事先创建好的一写Connection。程序只需要操作dataSource对象就可以获取其中的Connection(如果连接池里面没有可用Connection，会创建新的连接)。操作完成后执行dataSource的`close()`方法，此时不会关闭连接，只是会释放连接回连接池。

本章节，需要通过解析XML文件来获取定义在其中的数据源信息，包括驱动名称、数据库地址、用户名、密码。以及数据库事务管理方式。

### 解析数据源配置

#### XMLConfigBuilder

XMLConfigBuilder就是用来解析XML文件的，上一章节已经解析了`<mappers>`标签中的内容，本章节只需要继续添加解析`<environments>`标签中的内容即可。解析完的结果仍然存储在Configuration中。

```java
private void environmentsElement(Element context) throws Exception {
    String environment = context.attributeValue("default");

    List<Element> environmentList = context.elements("environment");
    for (Element e : environmentList) {
        String id = e.attributeValue("id");
        if (environment.equals(id)) {
            // 事务管理器
            TransactionFactory txFactory = (TransactionFactory) typeAliasRegistry.resolveAlias(e.element("transactionManager").attributeValue("type")).newInstance();
            // 数据源
            Element dataSourceElement = e.element("dataSource");
            DataSourceFactory dataSourceFactory = (DataSourceFactory) typeAliasRegistry.resolveAlias(dataSourceElement.attributeValue("type")).newInstance();
            List<Element> propertyList = dataSourceElement.elements("property");
            Properties props = new Properties();
            for (Element property : propertyList) {
                props.setProperty(property.attributeValue("name"), property.attributeValue("value"));
            }
            dataSourceFactory.setProperties(props);
            DataSource dataSource = dataSourceFactory.getDataSource();
            // 构建环境
            Environment.Builder environmentBuilder = new Environment.Builder(id)
                    .transactionFactory(txFactory)
                    .dataSource(dataSource);
            configuration.setEnvironment(environmentBuilder.build());
        }
    }
}
```

#### Environment

Java万事万物皆对象，将环境相关的内容存放在Environment对象中，方便处理。同时，为其设计了建造者模式，方便创建。

```java
public final class Environment {
    // 环境id
    private final String id;
    // 事务工厂
    private final TransactionFactory transactionFactory;
    // 数据源
    private final DataSource dataSource;

    public Environment(String id, TransactionFactory transactionFactory, DataSource dataSource) {
        this.id = id;
        this.transactionFactory = transactionFactory;
        this.dataSource = dataSource;
    }

    public static class Builder {

        private String id;
        private TransactionFactory transactionFactory;
        private DataSource dataSource;

        public Builder(String id) {
            this.id = id;
        }

        public Builder transactionFactory(TransactionFactory transactionFactory) {
            this.transactionFactory = transactionFactory;
            return this;
        }

        public Builder dataSource(DataSource dataSource) {
            this.dataSource = dataSource;
            return this;
        }

        public String id() {
            return this.id;
        }

        public Environment build() {
            return new Environment(this.id, this.transactionFactory, this.dataSource);
        }

    }
}
```

#### BoundSql

在上一章节的内容中，我们直接将SQL语句作为字符串存储在MappedStatement对象中。本章节，将SQL语句自身的信息抽取出来，存放在BoundSql类中，包括SQL语句、参数类型、结果类型等。将剩余的如SQL语句的类型(select、update、delete、insert)、SQL的ID等存放在MappedStatement中。

并用一个枚举类SqlCommandType来给定SQL语句的类型

```java
public class BoundSql {
    private String sql;
    private Map<Integer, String> parameterMappings;
    private String parameterType;
    private String resultType;

    public BoundSql(String sql, Map<Integer, String> parameterMappings, String parameterType, String resultType) {
        this.sql = sql;
        this.parameterMappings = parameterMappings;
        this.parameterType = parameterType;
        this.resultType = resultType;
    }
}
```

```java
public enum SqlCommandType {
    UNKNOWN,
    INSERT,
    UPDATE,
    DELETE,
    SELECT;
}
```



### 事务管理

一次数据库的操作，应该具备事务管理能力，而不是简单地通过数据库连接直接操作。事务管理能力应当包括把控连接、提交、回滚、关闭等操作。

#### Transaction

首先，我们对事务的基本功能做出约束。

```java
public interface Transaction {
    Connection getConnection() throws SQLException;

    void commit() throws SQLException;

    void rollback() throws SQLException;

    void close() throws SQLException;
}
```

#### JdbcTransaction

有了事务的定义，我们给出一个基于JDBC事务的实现。

我们创建事务时，可以通过数据库连接Connection直接创建，也可以通过数据源DataSource来创建。

由于是基于JDBC事务来实现的，所以事务的提交、回滚和关闭都是直接通过数据库连接Connection提供的接口来实现的。

```java
public class JdbcTransaction implements Transaction {
    protected Connection connection;
    protected DataSource dataSource;
    protected TransactionIsolationLevel level = TransactionIsolationLevel.NONE;
    protected boolean autoCommit;

    public JdbcTransaction(DataSource dataSource, TransactionIsolationLevel level, boolean autoCommit) {
        this.dataSource = dataSource;
        this.level = level;
        this.autoCommit = autoCommit;
    }

    public JdbcTransaction(Connection connection) {
        this.connection = connection;
    }

    @Override
    public Connection getConnection() throws SQLException {
        connection = dataSource.getConnection();
        connection.setTransactionIsolation(level.getLevel());
        connection.setAutoCommit(autoCommit);
        return connection;
    }

    @Override
    public void commit() throws SQLException {
        if (connection != null && !connection.getAutoCommit()) {
            connection.commit();
        }
    }

    @Override
    public void rollback() throws SQLException {
        if (connection != null && !connection.getAutoCommit()) {
            connection.rollback();
        }
    }

    @Override
    public void close() throws SQLException {
        if (connection != null && !connection.getAutoCommit()) {
            connection.close();
        }
    }
}
```

#### TransactionFactory

有了事务的定义，现在通过工厂模式将事务封装起来，方便获取实例对象。于是，给出工厂接口的定义。

```java
public interface TransactionFactory {
    /**
     * 根据 Connection 创建 Transaction
     * @param conn Existing database connection
     * @return Transaction
     */
    Transaction newTransaction(Connection conn);

    /**
     * 根据数据源和事务隔离级别创建 Transaction
     * @param dataSource DataSource to take the connection from
     * @param level Desired isolation level
     * @param autoCommit Desired autocommit
     * @return Transaction
     */
    Transaction newTransaction(DataSource dataSource, TransactionIsolationLevel level, boolean autoCommit);
}
```

#### JdbcTransactionFactory

给出事务工厂的实现。支持通过连接创建事务和通过数据源与事务隔离等级创建事务。

```java
public class JdbcTransactionFactory implements TransactionFactory {
    @Override
    public Transaction newTransaction(Connection conn) {
        return new JdbcTransaction(conn);
    }

    @Override
    public Transaction newTransaction(DataSource dataSource, TransactionIsolationLevel level, boolean autoCommit) {
        return new JdbcTransaction(dataSource, level, autoCommit);
    }
}
```

### 类型映射

数据库中的数据类型和Java中的数据类型之间需要转换才能正常使用。类型注册器就是记录数据类型之间的对应关系。Mybatis在设置预处理语句PreparedStatement中的参数或从结果集中取出一个值时，都会用类型处理器将获取到的值以合适的方式转换成Java类型。


#### TypeAliasRegistry类型注册器

在解析XML文件时(?存疑，还有别的地方也会用到)，通常会使用字符串来表示数据类型，于是我们需要预先记录下字符串与对应类型的对应关系。

我们可以使用一个Map集合来记录类型的对应关系。同时也需要记录数据库事务管理和数据源的字符串与类型对应关系。

```java
public class TypeAliasRegistry {
    private final Map<String, Class<?>> TYPE_ALIASES = new HashMap<>();

    public TypeAliasRegistry() {
        // 构造函数里注册系统内置的类型别名
        registerAlias("string", String.class);
        // 基本包装类型
        registerAlias("byte", Byte.class);
        registerAlias("long", Long.class);
        registerAlias("short", Short.class);
        registerAlias("int", Integer.class);
        registerAlias("integer", Integer.class);
        registerAlias("double", Double.class);
        registerAlias("float", Float.class);
        registerAlias("boolean", Boolean.class);
    }

    public void registerAlias(String alias, Class<?> value) {
        String key = alias.toLowerCase(Locale.ENGLISH);
        TYPE_ALIASES.put(key, value);
    }

    public <T> Class<T> resolveAlias(String string) {
        String key = string.toLowerCase(Locale.ENGLISH);
        return (Class<T>) TYPE_ALIASES.get(key);
    }
}
```

#### JdbcType

数据库中的数据类型在`java.sql.Types`包中有相关记录。为了使用方便，我们直接创建一个枚举类，来表示数据库中的各个类型。

```java
public enum JdbcType {
    INTEGER(Types.INTEGER),
    FLOAT(Types.FLOAT),
    DOUBLE(Types.DOUBLE),
    DECIMAL(Types.DECIMAL),
    VARCHAR(Types.VARCHAR),
    TIMESTAMP(Types.TIMESTAMP);

    public final int TYPE_CODE;
    private static Map<Integer,JdbcType> codeLookup = new HashMap<>();

    // 就将数字对应的枚举型放入 HashMap
    static {
        for (JdbcType type : JdbcType.values()) {
            codeLookup.put(type.TYPE_CODE, type);
        }
    }

    JdbcType(int code) {
        this.TYPE_CODE = code;
    }

    public static JdbcType forCode(int code)  {
        return codeLookup.get(code);
    }
}
```

#### ParameterMapping参数映射

在Mapper XML文件的SQL语句标签中，通常会以`#{property,javaType=int,jdbcType=NUMERIC}`的形式，给出当前参数名、参数的Java类型和数据库类型。

我们用一个ParameterMapping对来记录下一对映射关系。

```java
public class ParameterMapping {
    private Configuration configuration;

    // property
    private String property;
    // javaType = int
    private Class<?> javaType = Object.class;
    // jdbcType=NUMERIC
    private JdbcType jdbcType;

    private ParameterMapping() {
    }

    public static class Builder {

        private ParameterMapping parameterMapping = new ParameterMapping();

        public Builder(Configuration configuration, String property) {
            parameterMapping.configuration = configuration;
            parameterMapping.property = property;
        }

        public Builder javaType(Class<?> javaType) {
            parameterMapping.javaType = javaType;
            return this;
        }

        public Builder jdbcType(JdbcType jdbcType) {
            parameterMapping.jdbcType = jdbcType;
            return this;
        }

    }
}
```

#### 注册事务

在Configuration中添加类型别名注册机，通过构造函数添加JDBC，Druid注册操作。

```java
// 类型别名注册机
public class Configuration {
    protected final TypeAliasRegistry typeAliasRegistry = new TypeAliasRegistry();

    public Configuration() {
        typeAliasRegistry.registerAlias("JDBC", JdbcTransactionFactory.class);
        typeAliasRegistry.registerAlias("DRUID", DruidDataSourceFactory.class);
    }
}
```

### SQL执行和结果封装

通过`java.sql`提供的PreparedStatement来执行我们解析好的BoundSql语句。

SQL语句的执行结果是一个`java.sql.ResultSet`类型的集合，每个元素都是一条记录。每个记录里又包含了多个对象，每个对象代表数据库表的一个属性。

我们通过反射的方式，先为每条记录申请一个对应类型的对象。然后通过获取属性名的方式获取对象的成员变量名。然后构造`set()`方法，为对象的该成员变量赋值。最终获得一个数据库表记录对应的对象。

```java
@Override
public <T> T selectOne(String statement, Object parameter) {
    try {
        MappedStatement mappedStatement = configuration.getMappedStatement(statement);
        Environment environment = configuration.getEnvironment();

        Connection connection = environment.getDataSource().getConnection();

        BoundSql boundSql = mappedStatement.getBoundSql();
        PreparedStatement preparedStatement = connection.prepareStatement(boundSql.getSql());
        preparedStatement.setLong(1, Long.parseLong(((Object[]) parameter)[0].toString()));
        ResultSet resultSet = preparedStatement.executeQuery();

        List<T> objList = resultSet2Obj(resultSet, Class.forName(boundSql.getResultType()));
        return objList.get(0);
    } catch (Exception e) {
        e.printStackTrace();
        return null;
    }
}

private <T> List<T> resultSet2Obj(ResultSet resultSet, Class<?> clazz) {
    List<T> list = new ArrayList<>();
    try {
        ResultSetMetaData metaData = resultSet.getMetaData();
        int columnCount = metaData.getColumnCount();
        // 每次遍历行值
        while (resultSet.next()) {
            T obj = (T) clazz.newInstance();
            for (int i = 1; i <= columnCount; i++) {
                Object value = resultSet.getObject(i);
                String columnName = metaData.getColumnName(i);
                String setMethod = "set" + columnName.substring(0, 1).toUpperCase() + columnName.substring(1);
                Method method;
                if (value instanceof Timestamp) {
                    method = clazz.getMethod(setMethod, Date.class);
                } else {
                    method = clazz.getMethod(setMethod, value.getClass());
                }
                method.invoke(obj, value);
            }
            list.add(obj);
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
    return list;
}
```

### 数据源的使用

在使用Mybatis时，通常使用的是阿里的Druid数据源。本章节暂时先使用Druid作为数据源直接使用。

既然已经有数据源的实现，我们只需要创建数据源工厂即可。首先，给出DataSourceFactory接口的定义。其核心功能就是返回一个数据源。

```java
public interface DataSourceFactory {
    void setProperties(Properties props);
    DataSource getDataSource();
}
```

然后，再基于Druid给出一个简单的数据源工厂的实现。生产的数据源就是Druid数据源。

```java
public class DruidDataSourceFactory implements DataSourceFactory {
    private Properties props;

    @Override
    public void setProperties(Properties props) {
        this.props = props;
    }

    @Override
    public DataSource getDataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setDriverClassName(props.getProperty("driver"));
        dataSource.setUrl(props.getProperty("url"));
        dataSource.setUsername(props.getProperty("username"));
        dataSource.setPassword(props.getProperty("password"));
        return dataSource;
    }
}
```


## 第五章 数据源池化技术

### Driver, Connection, DataSource三者联系

`Driver`是JDBC中的一个接口，数据库厂商需要提供该接口的实现类，从而对Java提供操作数据库的API。而Java程序中不需要关心Driver接口的实现类，只需要通过`DriverManager`类来操作数据库。

`DriverManager`用来管理Driver的实现类，即是管理驱动的管理类。`DriverManager#getConnection`方法，可以根据给出的数据库信息`Properties`来建立一个Connection。一个Driver可以开启多个Connection。

`Connection`是连接数据库的物理链路。一个Connection对应了和数据库的一个连接。一个DriverManager可以根据提供的数据库信息Properties开启多个Connection。

`DataSource`数据源，数据的来源，可以不是数据库。在连接数据库时，DataSource用来记录数据库的相关信息，如地址、账户名、密码等。DataSource相当于将DriverManager又做了一次封装。用来管理连接等。

### 概述

上一章节中，我们使用Druid数据源完成数据库操作，但是Mybatis中有自己的数据源实现，本章节我们实现一个池化数据源。帮助理解最大活跃连接数、空闲连接数、检测时长等参数在连接池中的作用。

池化技术是享元模式的一种实现。通常我们对一些需要较高创建成本且高频使用的资源，需要进行缓存或者也称预热处理。并把这些资源存放到一个预热池子中，需要用的时候从池子中获取，使用完毕再进行使用。通过池化可以非常有效的控制资源的使用成本，包括；资源数量、空闲时长、获取方式等进行统一控制和管理。

> 享元模式(Flyweight Pattern)：运用共享技术有效地支持大量细粒度对象的复用。系统只使用少量的对象，而这些对象都很相似，状态变化很小，可以实现对象的多次复用。由于享元模式要求能够共享的对象必须是细粒度对象，因此它又称为轻量级模式，它是一种对象结构型模式。

通过提供统一的连接池中心，存放数据源链接，并根据配置按照请求获取链接的操作，创建连接池的数据源链接数量。这里就包括了最大空闲链接和最大活跃链接，都随着创建过程被控制。

此外由于控制了连接池中连接的数量，所以当外部从连接池获取链接时，如果链接已满则会进行循环等待。这也是大家日常使用DB连接池，如果一个SQL操作引起了慢查询，则会导致整个服务进入瘫痪的阶段，各个和数据库相关的接口调用，都不能获得到链接，接口查询TP99陡然增高，系统开始大量报警。

> 那连接池可以配置的很大吗，也不可以，因为连接池要和数据库所分配的连接池对应上，避免应用配置连接池超过数据库所提供的连接池数量，否则会出现夯住不能分配链接的问题，导致数据库拖垮从而引起连锁反应。


### 非池化型数据源实现

对于数据库连接池的实现，不一定非得提供池化技术，对于某些场景可以只使用非池化型的数据源。那么在实现的过程中，可以把非池化型的实现和池化实现拆分解耦，在需要的时候只需要配置对应的数据源即可。

非池化型的数据源，也可以一次建立多个Connection，但是每次操作完毕，都会将Connection关闭。下次再次建立连接时会重新建立。

#### UnPooledDataSource

类中使用了一个静态代码块，在类文件被加载进内存时便会执行。通过`java.sql.DriverManager`包给出的`getDrivers()`方法，将当前阶段被加载进内存的所有JDBC驱动读入类中的Map集合中保存起来。

`initializeDriver()`方法会检查当前非池化型数据源实例使用的驱动是否已经被加载进内存，如果没有就会将其加载，并实例化一个驱动对象。同时记录进Map集合。

通过代码可以看出，非池化型的数据源只会保持一个Connection实例，在初始化驱动方法中加载驱动创建数据库连接，通过`DriverManager#getConnection`来获取连接的，如果没有可用连接就会抛出异常。

```java
public class UnpooledDataSource implements DataSource {
    private ClassLoader driverClassLoader;
    // 驱动配置，也可以扩展属性信息 driver.encoding=UTF8
    private Properties driverProperties;
    // 驱动注册器
    private static Map<String, Driver> registeredDrivers = new ConcurrentHashMap<>();
    // 驱动
    private String driver;
    // DB 链接地址
    private String url;
    // 账号
    private String username;
    // 密码
    private String password;
    // 是否自动提交
    private Boolean autoCommit;
    // 事务级别
    private Integer defaultTransactionIsolationLevel;

    static { // 静态代码块，在类被加载时便执行。将当前已经加载的所有JDBC驱动都添加到Map集合中。
        Enumeration<Driver> drivers = DriverManager.getDrivers();
        while (drivers.hasMoreElements()) {
            Driver driver = drivers.nextElement();
            registeredDrivers.put(driver.getClass().getName(), driver);
        }
    }

    /**
     * 驱动代理
     */
    private static class DriverProxy implements Driver {

        private Driver driver;

        DriverProxy(Driver driver) {
            this.driver = driver;
        }

        @Override
        public Connection connect(String u, Properties p) throws SQLException {
            return this.driver.connect(u, p);
        }

        @Override
        public boolean acceptsURL(String u) throws SQLException {
            return this.driver.acceptsURL(u);
        }

        @Override
        public DriverPropertyInfo[] getPropertyInfo(String u, Properties p) throws SQLException {
            return this.driver.getPropertyInfo(u, p);
        }

        @Override
        public int getMajorVersion() {
            return this.driver.getMajorVersion();
        }

        @Override
        public int getMinorVersion() {
            return this.driver.getMinorVersion();
        }

        @Override
        public boolean jdbcCompliant() {
            return this.driver.jdbcCompliant();
        }

        @Override
        public Logger getParentLogger() throws SQLFeatureNotSupportedException {
            return Logger.getLogger(Logger.GLOBAL_LOGGER_NAME);
        }
    }

    /**
     * 初始化驱动
     */
    private synchronized void initializerDriver() throws SQLException {
        if (!registeredDrivers.containsKey(driver)) {
            try {
                Class<?> driverType = Class.forName(driver, true, driverClassLoader);
                // https://www.kfu.com/~nsayer/Java/dyn-jdbc.html
                Driver driverInstance = (Driver) driverType.newInstance();
                DriverManager.registerDriver(new DriverProxy(driverInstance));
                registeredDrivers.put(driver, driverInstance);
            } catch (Exception e) {
                throw new SQLException("Error setting driver on UnpooledDataSource. Cause: " + e);
            }
        }
    }

    private Connection doGetConnection(String username, String password) throws SQLException {
        Properties props = new Properties();
        if (driverProperties != null) {
            props.putAll(driverProperties);
        }
        if (username != null) {
            props.setProperty("user", username);
        }
        if (password != null) {
            props.setProperty("password", password);
        }
        return doGetConnection(props);
    }

    private Connection doGetConnection(Properties properties) throws SQLException {
        initializerDriver();
        Connection connection = DriverManager.getConnection(url, properties);
        if (autoCommit != null && autoCommit != connection.getAutoCommit()) {
            connection.setAutoCommit(autoCommit);
        }
        if (defaultTransactionIsolationLevel != null) {
            connection.setTransactionIsolation(defaultTransactionIsolationLevel);
        }
        return connection;
    }

    @Override
    public Connection getConnection() throws SQLException {
        return doGetConnection(username, password);
    }

    @Override
    public Connection getConnection(String username, String password) throws SQLException {
        return doGetConnection(username, password);
    }

    @Override
    public <T> T unwrap(Class<T> iface) throws SQLException {
        throw new SQLException(getClass().getName() + " is not a wrapper.");
    }

    @Override
    public boolean isWrapperFor(Class<?> iface) throws SQLException {
        return false;
    }

    @Override
    public PrintWriter getLogWriter() throws SQLException {
        return DriverManager.getLogWriter();
    }

    @Override
    public void setLogWriter(PrintWriter logWriter) throws SQLException {
        DriverManager.setLogWriter(logWriter);
    }

    @Override
    public void setLoginTimeout(int loginTimeout) throws SQLException {
        DriverManager.setLoginTimeout(loginTimeout);
    }

    @Override
    public int getLoginTimeout() throws SQLException {
        return DriverManager.getLoginTimeout();
    }

    @Override
    public Logger getParentLogger() throws SQLFeatureNotSupportedException {
        return Logger.getLogger(Logger.GLOBAL_LOGGER_NAME);
    }
}
```

#### UnpooledDataSourceFactory

核心功能为返回一个非池化型的数据源实例。

```java
public class UnpooledDataSourceFactory implements DataSourceFactory {
    protected Properties props;

    @Override
    public void setProperties(Properties props) {
        this.props = props;
    }

    @Override
    public DataSource getDataSource() {
        UnpooledDataSource unpooledDataSource = new UnpooledDataSource();
        unpooledDataSource.setDriver(props.getProperty("driver"));
        unpooledDataSource.setUrl(props.getProperty("url"));
        unpooledDataSource.setUsername(props.getProperty("username"));
        unpooledDataSource.setPassword(props.getProperty("password"));
        return unpooledDataSource;
    }
}
```

### 池化数据源-连接池

池化数据源核心在于对非池化型数据源的包装，同时提供了相应的池化技术实现，包括：pushConnection、popConnection、forceCloseAll、pingConnection的操作处理。

当用户想要获取链接的时候，则会从连接池中进行获取，同时判断是否有空闲链接、最大活跃链接多少，以及是否需要等待处理或是最终抛出异常。

> 池化的数据源，其内部包含一个非池化的数据源，保持一个Connection实例。PooledConnection

#### PooledConnection池化连接处理

池化数据源，用户执行了`close()`方法后，不能真的关闭Connection，而是将其释放回连接池，允许其他用户获取。因此，就需要对Connection类进行代理包装，处理`close()`方法。

PooledConnection实现InvocationHandler接口，重写`invoke()`方法，在其中执行Connection中方法的具体逻辑。

PooledConnection需要记录被代理的真实连接Connection对象，并通过代理的方式生成一个代理的Connection对象。
- 真实连接的Connection实例，可能不是一一对应的，一个Connection可能对应多个PooledConnection。// TODO ? 真的吗

`invoke()`方法中，需要判断当前执行的是什么方法。如果是`close()`方法，则执行数据源中的`pushConnection()`方法，将连接置为可用。

> 代理连接PooledConnection的作用就是，给每一个数据库连接Connection生成一个代理对象，每当要执行`Connection#close`时，就执行代理连接的逻辑，即`PooledDataSource#pushConnection`。

```java
public class PooledConnection implements InvocationHandler {
    private static final String CLOSE = "close";
    private static final Class<?>[] IFACES = new Class<?>[]{Connection.class};

    private int hashCode = 0;
    private PooledDataSource dataSource;

    // 真实的连接
    private Connection realConnection;
    // 代理的连接
    private Connection proxyConnection;

    private long checkoutTimestamp;
    private long createdTimestamp;
    private long lastUsedTimestamp;
    private int connectionTypeCode;
    private boolean valid;

    public PooledConnection(Connection connection, PooledDataSource dataSource) {
        this.hashCode = connection.hashCode();
        this.realConnection = connection;
        this.dataSource = dataSource;
        this.createdTimestamp = System.currentTimeMillis();
        this.lastUsedTimestamp = System.currentTimeMillis();
        this.valid = true;
        this.proxyConnection = (Connection) Proxy.newProxyInstance(Connection.class.getClassLoader(), IFACES, this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String methodName = method.getName();
        // 如果是调用 CLOSE 关闭链接方法，则将链接加入连接池中，并返回null
        if (CLOSE.hashCode() == methodName.hashCode() && CLOSE.equals(methodName)) {
            dataSource.pushConnection(this);
            return null;
        } else {
            if (!Object.class.equals(method.getDeclaringClass())) {
                // 除了toString()方法，其他方法调用之前要检查connection是否还是合法的,不合法要抛出SQLException
                checkConnection();
            }
            // 其他方法交给connection去调用
            return method.invoke(realConnection, args);
        }
    }

    private void checkConnection() throws SQLException {
        if (!valid) {
            throw new SQLException("Error accessing PooledConnection. Connection is invalid.");
        }
    }

    public void invalidate() {
        valid = false;
    }

    public boolean isValid() {
        return valid && realConnection != null && dataSource.pingConnection(this);
    }
}
```

#### PooledDataSource

一个池化数据源，其实就是将非池化型的数据源UnpooledDataSource做了一次封装。UnpooledDataSource负责申请Connection。而PooledDataSource返回给外界的连接其实是代理对象PooledConnection。

其在内部维护了如最大空闲连接数、最大活跃连接数等参数，以及一个连接池`PoolState`。会同时保持着多个Connection以及其对应的代理PooledConnection，避免反复建立连接释放连接。

##### pushConnection

`PooledDataSource#pushConnection`用于回收连接。核心逻辑是判断需要回收的连接是否有效、空闲连接的数量以及保证数据库一致性。

先检查连接是否无效，如果无效就打印日志，无需处理逻辑。

如果有效，就需要判断空闲连接数是否小于设定值，如果大于设定值，证明空闲连接比较充足，回收连接后直接PooledConnection对应的Connection关闭即可。

如果小于设定值，就实例化一个新的PooledConnection，加入到连接池中。


##### popConnection

`PooledDataSource#popConnection`用于获取链接(向外界返回连接)。其执行逻辑是一个无限循环，只有抛出异常才会停止。外界申请获得连接时，会先判断有无空闲连接，入股有，就直接返回一个空闲的PooledConnection；如果没有执行以下逻辑：

如果活跃连接数没有达到上限，就创建一个新的PooledConnection，并返回。如果已经到达上限，无法创建新的连接，只能淘汰已有的连接。

先找到存活时间最长的连接，检查是否已经过期，如果过期就删除(需要注意数据库一致性)，并实例化一个新的连接。否则，不能删除，只能让当前的申请阻塞等待。

在返回新连接之前，需要作出处理。


```java
PooledDataSource implements DataSource {
    private org.slf4j.Logger logger = LoggerFactory.getLogger(PooledDataSource.class);
    // 池状态
    private final PoolState state = new PoolState(this);

    private final UnpooledDataSource dataSource;

    // 活跃连接数
    protected int poolMaximumActiveConnections = 10;
    // 空闲连接数
    protected int poolMaximumIdleConnections = 5;
    // 在被强制返回之前,池中连接被检查的时间
    protected int poolMaximumCheckoutTime = 20000;
    // 这是给连接池一个打印日志状态机会的低层次设置,还有重新尝试获得连接, 这些情况下往往需要很长时间 为了避免连接池没有配置时静默失败)。
    protected int poolTimeToWait = 20000;
    // 发送到数据的侦测查询,用来验证连接是否正常工作,并且准备 接受请求。默认是“NO PING QUERY SET” ,这会引起许多数据库驱动连接由一 个错误信息而导致失败
    protected String poolPingQuery = "NO PING QUERY SET";
    // 开启或禁用侦测查询
    protected boolean poolPingEnabled = false;
    // 用来配置 poolPingQuery 多次时间被用一次
    protected int poolPingConnectionsNotUsedFor = 0;

    private int expectedConnectionTypeCode;

    public PooledDataSource() {
        this.dataSource = new UnpooledDataSource();
    }

    protected void pushConnection(PooledConnection connection) throws SQLException {
        synchronized (state) {
            state.activeConnections.remove(connection);
            // 判断链接是否有效
            if (connection.isValid()) {
                // 如果空闲链接小于设定数量，也就是太少时
                if (state.idleConnections.size() < poolMaximumIdleConnections && connection.getConnectionTypeCode() == expectedConnectionTypeCode) {
                    state.accumulatedCheckoutTime += connection.getCheckoutTime();
                    // 它首先检查数据库连接是否处于自动提交模式，如果不是，则调用rollback()方法执行回滚操作。
                    // 在MyBatis中，如果没有开启自动提交模式，则需要手动提交或回滚事务。因此，这段代码可能是在确保操作完成后，如果没有开启自动提交模式，则执行回滚操作。
                    // 总的来说，这段代码用于保证数据库的一致性，确保操作完成后，如果未开启自动提交模式，则执行回滚操作。
                    if (!connection.getRealConnection().getAutoCommit()) {
                        connection.getRealConnection().rollback();
                    }
                    // 实例化一个新的DB连接，加入到idle列表
                    PooledConnection newConnection = new PooledConnection(connection.getRealConnection(), this);
                    state.idleConnections.add(newConnection);
                    newConnection.setCreatedTimestamp(connection.getCreatedTimestamp());
                    newConnection.setLastUsedTimestamp(connection.getLastUsedTimestamp());
                    connection.invalidate();
                    logger.info("Returned connection " + newConnection.getRealHashCode() + " to pool.");

                    // 通知其他线程可以来抢DB连接了
                    state.notifyAll();
                }
                // 否则，空闲链接还比较充足
                else {
                    state.accumulatedCheckoutTime += connection.getCheckoutTime();
                    if (!connection.getRealConnection().getAutoCommit()) {
                        connection.getRealConnection().rollback();
                    }
                    // 将connection关闭
                    connection.getRealConnection().close();
                    logger.info("Closed connection " + connection.getRealHashCode() + ".");
                    connection.invalidate();
                }
            } else {
                logger.info("A bad connection (" + connection.getRealHashCode() + ") attempted to return to the pool, discarding connection.");
                state.badConnectionCount++;
            }
        }
    }

    private PooledConnection popConnection(String username, String password) throws SQLException {
        boolean countedWait = false;
        PooledConnection conn = null;
        long t = System.currentTimeMillis();
        int localBadConnectionCount = 0;

        while (conn == null) {
            synchronized (state) {
                // 如果有空闲链接：返回第一个
                if (!state.idleConnections.isEmpty()) {
                    conn = state.idleConnections.remove(0);
                    logger.info("Checked out connection " + conn.getRealHashCode() + " from pool.");
                }
                // 如果无空闲链接：创建新的链接
                else {
                    // 活跃连接数不足
                    if (state.activeConnections.size() < poolMaximumActiveConnections) {
                        conn = new PooledConnection(dataSource.getConnection(), this);
                        logger.info("Created connection " + conn.getRealHashCode() + ".");
                    }
                    // 活跃连接数已满
                    else {
                        // 取得活跃链接列表的第一个，也就是最老的一个连接
                        PooledConnection oldestActiveConnection = state.activeConnections.get(0);
                        long longestCheckoutTime = oldestActiveConnection.getCheckoutTime();
                        // 如果checkout时间过长，则这个链接标记为过期
                        if (longestCheckoutTime > poolMaximumCheckoutTime) {
                            state.claimedOverdueConnectionCount++;
                            state.accumulatedCheckoutTimeOfOverdueConnections += longestCheckoutTime;
                            state.accumulatedCheckoutTime += longestCheckoutTime;
                            state.activeConnections.remove(oldestActiveConnection);
                            if (!oldestActiveConnection.getRealConnection().getAutoCommit()) {
                                oldestActiveConnection.getRealConnection().rollback();
                            }
                            // 删掉最老的链接，然后重新实例化一个新的链接
                            conn = new PooledConnection(oldestActiveConnection.getRealConnection(), this);
                            oldestActiveConnection.invalidate();
                            logger.info("Claimed overdue connection " + conn.getRealHashCode() + ".");
                        }
                        // 如果checkout超时时间不够长，则等待
                        else {
                            try {
                                if (!countedWait) {
                                    state.hadToWaitCount++;
                                    countedWait = true;
                                }
                                logger.info("Waiting as long as " + poolTimeToWait + " milliseconds for connection.");
                                long wt = System.currentTimeMillis();
                                state.wait(poolTimeToWait);
                                state.accumulatedWaitTime += System.currentTimeMillis() - wt;
                            } catch (InterruptedException e) {
                                break;
                            }
                        }

                    }
                }
                // 获得到链接
                if (conn != null) {
                    if (conn.isValid()) {
                        if (!conn.getRealConnection().getAutoCommit()) {
                            conn.getRealConnection().rollback();
                        }
                        conn.setConnectionTypeCode(assembleConnectionTypeCode(dataSource.getUrl(), username, password));
                        // 记录checkout时间
                        conn.setCheckoutTimestamp(System.currentTimeMillis());
                        conn.setLastUsedTimestamp(System.currentTimeMillis());
                        state.activeConnections.add(conn);
                        state.requestCount++;
                        state.accumulatedRequestTime += System.currentTimeMillis() - t;
                    } else {
                        logger.info("A bad connection (" + conn.getRealHashCode() + ") was returned from the pool, getting another connection.");
                        // 如果没拿到，统计信息：失败链接 +1
                        state.badConnectionCount++;
                        localBadConnectionCount++;
                        conn = null;
                        // 失败次数较多，抛异常
                        if (localBadConnectionCount > (poolMaximumIdleConnections + 3)) {
                            logger.debug("PooledDataSource: Could not get a good connection to the database.");
                            throw new SQLException("PooledDataSource: Could not get a good connection to the database.");
                        }
                    }
                }
            }
        }

        if (conn == null) {
            logger.debug("PooledDataSource: Unknown severe error condition.  The connection pool returned a null connection.");
            throw new SQLException("PooledDataSource: Unknown severe error condition.  The connection pool returned a null connection.");
        }

        return conn;
    }

    public void forceCloseAll() {
        synchronized (state) {
            expectedConnectionTypeCode = assembleConnectionTypeCode(dataSource.getUrl(), dataSource.getUsername(), dataSource.getPassword());
            // 关闭活跃链接
            for (int i = state.activeConnections.size(); i > 0; i--) {
                try {
                    PooledConnection conn = state.activeConnections.remove(i - 1);
                    conn.invalidate();

                    Connection realConn = conn.getRealConnection();
                    if (!realConn.getAutoCommit()) {
                        realConn.rollback();
                    }
                    realConn.close();
                } catch (Exception ignore) {

                }
            }
            // 关闭空闲链接
            for (int i = state.idleConnections.size(); i > 0; i--) {
                try {
                    PooledConnection conn = state.idleConnections.remove(i - 1);
                    conn.invalidate();

                    Connection realConn = conn.getRealConnection();
                    if (!realConn.getAutoCommit()) {
                        realConn.rollback();
                    }
                } catch (Exception ignore) {

                }
            }
            logger.info("PooledDataSource forcefully closed/removed all connections.");
        }
    }

    protected boolean pingConnection(PooledConnection conn) {
        boolean result = true;

        try {
            result = !conn.getRealConnection().isClosed();
        } catch (SQLException e) {
            logger.info("Connection " + conn.getRealHashCode() + " is BAD: " + e.getMessage());
            result = false;
        }

        if (result) {
            if (poolPingEnabled) {
                if (poolPingConnectionsNotUsedFor >= 0 && conn.getTimeElapsedSinceLastUse() > poolPingConnectionsNotUsedFor) {
                    try {
                        logger.info("Testing connection " + conn.getRealHashCode() + " ...");
                        Connection realConn = conn.getRealConnection();
                        Statement statement = realConn.createStatement();
                        ResultSet resultSet = statement.executeQuery(poolPingQuery);
                        resultSet.close();
                        if (!realConn.getAutoCommit()) {
                            realConn.rollback();
                        }
                        result = true;
                        logger.info("Connection " + conn.getRealHashCode() + " is GOOD!");
                    } catch (Exception e) {
                        logger.info("Execution of ping query '" + poolPingQuery + "' failed: " + e.getMessage());
                        try {
                            conn.getRealConnection().close();
                        } catch (SQLException ignore) {
                        }
                        result = false;
                        logger.info("Connection " + conn.getRealHashCode() + " is BAD: " + e.getMessage());
                    }
                }
            }
        }

        return result;
    }

    public static Connection unwrapConnection(Connection conn) {
        if (Proxy.isProxyClass(conn.getClass())) {
            InvocationHandler handler = Proxy.getInvocationHandler(conn);
            if (handler instanceof PooledConnection) {
                return ((PooledConnection) handler).getRealConnection();
            }
        }
        return conn;
    }

    private int assembleConnectionTypeCode(String url, String username, String password) {
        return ("" + url + username + password).hashCode();
    }

    @Override
    public Connection getConnection() throws SQLException {
        return popConnection(dataSource.getUsername(), dataSource.getPassword()).getProxyConnection();
    }

    @Override
    public Connection getConnection(String username, String password) throws SQLException {
        return popConnection(username, password).getProxyConnection();
    }

    protected void finalize() throws Throwable {
        forceCloseAll();
        super.finalize();
    }

    @Override
    public <T> T unwrap(Class<T> iface) throws SQLException {
        throw new SQLException(getClass().getName() + " is not a wrapper.");
    }

    @Override
    public boolean isWrapperFor(Class<?> iface) throws SQLException {
        return false;
    }

    @Override
    public PrintWriter getLogWriter() throws SQLException {
        return DriverManager.getLogWriter();
    }

    @Override
    public void setLogWriter(PrintWriter logWriter) throws SQLException {
        DriverManager.setLogWriter(logWriter);
    }

    @Override
    public void setLoginTimeout(int loginTimeout) throws SQLException {
        DriverManager.setLoginTimeout(loginTimeout);
    }

    @Override
    public int getLoginTimeout() throws SQLException {
        return DriverManager.getLoginTimeout();
    }

    @Override
    public Logger getParentLogger() throws SQLFeatureNotSupportedException {
        return Logger.getLogger(Logger.GLOBAL_LOGGER_NAME);
    }
}
```

#### 池化数据源工厂

逻辑较简单，就是实例化一个池化数据源，设置相关属性并返回。

```java
public class PooledDataSourceFactory extends UnpooledDataSourceFactory {
    @Override
    public DataSource getDataSource() {
        PooledDataSource pooledDataSource = new PooledDataSource();
        pooledDataSource.setDriver(props.getProperty("driver"));
        pooledDataSource.setUrl(props.getProperty("url"));
        pooledDataSource.setUsername(props.getProperty("username"));
        pooledDataSource.setPassword(props.getProperty("password"));
        return pooledDataSource;
    }
}
```

#### PoolState 连接池状态

用于维护当前连接池中PooledConnection的状态，分别用两个List集合保存空闲和活跃连接。

```java
public class PoolState {
    protected PooledDataSource dataSource;

    // 空闲链接
    protected final List<PooledConnection> idleConnections = new ArrayList<>();
    // 活跃链接
    protected final List<PooledConnection> activeConnections = new ArrayList<>();

    // 请求次数
    protected long requestCount = 0;
    // 总请求时间
    protected long accumulatedRequestTime = 0;
    protected long accumulatedCheckoutTime = 0;
    protected long claimedOverdueConnectionCount = 0;
    protected long accumulatedCheckoutTimeOfOverdueConnections = 0;

    // 总等待时间
    protected long accumulatedWaitTime = 0;
    // 要等待的次数
    protected long hadToWaitCount = 0;
    // 失败连接次数
    protected long badConnectionCount = 0;

    public PoolState(PooledDataSource dataSource) {
        this.dataSource = dataSource;
    }
}
```


## 第六章 SQL执行器的定义

在上一章节中，我们实现了池化数据源和非池化数据源，但是，与数据源以及与数据库的操作逻辑是直接写在SqlSession中的。这不利于后期扩展其他数据源，也不利于SqlSession中新增其他的CRUD方法(每一个方法都需要去直接与数据源交互)。

所以，我们应该将对数据源的调用、SQL的执行和结果的处理都抽取出来，只提供一个方法入口，这样方便后期扩展和维护。

主要分为两部分：执行器和语句处理器
- 执行器负责具体的执行逻辑。
- 语句处理器负责将SQL语句的处理，如传参操作等。

### 执行器的定义和实现

首先，执行器分为三部分：接口、抽象类和简单实现类。接口中，将事务操作和SQL执行的统一标准做出约束。然后，由抽象类来定义统一共用的事务操作和SQL执行流程。

在具体的实现类中，完成SQL执行的具体逻辑。

#### Executor

```java
public interface Executor {
    ResultHandler NO_RESULT_HANDLER = null;

    <E> List<E> query(MappedStatement ms, Object parameter, ResultHandler resultHandler, BoundSql boundSql);

    Transaction getTransaction();

    void commit(boolean required) throws SQLException;

    void rollback(boolean required) throws SQLException;

    void close(boolean forceRollback);
}
```

#### BaseExecutor

事务的操作逻辑都是一致的，所以直接在抽象类中定义。具体的SQL语句的执行逻辑是因SQL语句不同而变化的，所以放在实现在类中实现。

```java
public abstract class BaseExecutor implements Executor {
    private org.slf4j.Logger logger = LoggerFactory.getLogger(BaseExecutor.class);

    protected Configuration configuration;
    protected Transaction transaction;
    protected Executor wrapper;

    private boolean closed;

    protected BaseExecutor(Configuration configuration, Transaction transaction) {
        this.configuration = configuration;
        this.transaction = transaction;
        this.wrapper = this;
    }

    @Override
    public <E> List<E> query(MappedStatement ms, Object parameter, ResultHandler resultHandler, BoundSql boundSql) {
        if (closed) {
            throw new RuntimeException("Executor was closed.");
        }
        return doQuery(ms, parameter, resultHandler, boundSql);
    }

    protected abstract <E> List<E> doQuery(MappedStatement ms, Object parameter, ResultHandler resultHandler, BoundSql boundSql);

    @Override
    public Transaction getTransaction() {
        if (closed) {
            throw new RuntimeException("Executor was closed.");
        }
        return transaction;
    }

    @Override
    public void commit(boolean required) throws SQLException {
        if (closed) {
            throw new RuntimeException("Cannot commit, transaction is already closed");
        }
        if (required) {
            transaction.commit();
        }
    }

    @Override
    public void rollback(boolean required) throws SQLException {
        if (!closed) {
            if (required) {
                transaction.rollback();
            }
        }
    }

    @Override
    public void close(boolean forceRollback) {
        try {
            try {
                rollback(forceRollback);
            } finally {
                transaction.close();
            }
        } catch (SQLException e) {
            logger.warn("Unexpected exception on closing transaction.  Cause: " + e);
        } finally {
            transaction = null;
            closed = true;
        }
    }
}
```

#### SimpleExecutor

具体的SQL执行逻辑交给执行器的实现类完成。

```java
public class SimpleExecutor extends BaseExecutor {
    public SimpleExecutor(Configuration configuration, Transaction transaction) {
        super(configuration, transaction);
    }

    @Override
    protected <E> List<E> doQuery(MappedStatement ms, Object parameter, ResultHandler resultHandler, BoundSql boundSql) {
        try {
            Configuration configuration = ms.getConfiguration();
            StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, resultHandler, boundSql);
            Connection connection = transaction.getConnection();
            Statement stmt = handler.prepare(connection);
            handler.parameterize(stmt);
            return handler.query(stmt, resultHandler);
        } catch (SQLException e) {
            e.printStackTrace();
            return null;
        }
    }
}
```

### 语句处理器

SQL中，是由`java.sql.Statement`实例来通过`execute(String sql)`来执行SQL语句的。因此，将实例化Statement对象、参数处理和具体执行都做出规范化约束，提取到`StatementHandler`中。然后，将通用的逻辑交给`BaseStatementHandler`来实现。其余的具体逻辑放在实现类实现。

#### StatementHandler

对准备Statement(实例化一个对象)、参数化和执行逻辑做出约束。

```java
public interface StatementHandler {
    /** 准备语句 */
    Statement prepare(Connection connection) throws SQLException;

    /** 参数化 */
    void parameterize(Statement statement) throws SQLException;

    /** 执行查询 */
    <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException;
}
```

#### BaseStatementHandler

这是一个抽象类，对准备Statement需要用到的引用和共有逻辑做出实现。

```java
public abstract class BaseStatementHandler implements StatementHandler {
    protected final Configuration configuration;
    protected final Executor executor;
    protected final MappedStatement mappedStatement;

    protected final Object parameterObject;
    protected final ResultSetHandler resultSetHandler;

    protected BoundSql boundSql;

    public BaseStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, ResultHandler resultHandler, BoundSql boundSql) {
        this.configuration = mappedStatement.getConfiguration();
        this.executor = executor;
        this.mappedStatement = mappedStatement;
        this.boundSql = boundSql;

        this.parameterObject = parameterObject;
        this.resultSetHandler = configuration.newResultSetHandler(executor, mappedStatement, boundSql);
    }

    @Override
    public Statement prepare(Connection connection) throws SQLException {
        Statement statement = null;
        try {
            // 实例化 Statement
            statement = instantiateStatement(connection);
            // 参数设置，可以被抽取，提供配置
            statement.setQueryTimeout(350);
            statement.setFetchSize(10000);
            return statement;
        } catch (Exception e) {
            throw new RuntimeException("Error preparing statement.  Cause: " + e, e);
        }
    }

    protected abstract Statement instantiateStatement(Connection connection) throws SQLException;
}
```

#### PreparedStatementHandler

```java
public class PreparedStatementHandler extends BaseStatementHandler{
    public PreparedStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, ResultHandler resultHandler, BoundSql boundSql) {
        super(executor, mappedStatement, parameterObject, resultHandler, boundSql);
    }

    @Override
    protected Statement instantiateStatement(Connection connection) throws SQLException {
        String sql = boundSql.getSql();
        return connection.prepareStatement(sql);
    }

    @Override
    public void parameterize(Statement statement) throws SQLException {
        PreparedStatement ps = (PreparedStatement) statement;
        ps.setLong(1, Long.parseLong(((Object[]) parameterObject)[0].toString()));
    }

    @Override
    public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
        PreparedStatement ps = (PreparedStatement) statement;
        ps.execute();
        return resultSetHandler.<E> handleResultSets(ps);
    }
}
```

### 执行器的创建与使用

#### 执行器的创建

我们一开始就将程序与数据库的一次交互封装为一个SqlSession。于是，在实例化SqlSession对象时就要为其实例化一个执行器。于是将这部分逻辑放在工厂中。

```java
public class DefaultSqlSessionFactory implements SqlSessionFactory {
    private final Configuration configuration;

    public DefaultSqlSessionFactory(Configuration configuration) {
        this.configuration = configuration;
    }

    @Override
    public SqlSession openSession() {
        Transaction tx = null;
        try {
            final Environment environment = configuration.getEnvironment();
            TransactionFactory transactionFactory = environment.getTransactionFactory();
            tx = transactionFactory.newTransaction(configuration.getEnvironment().getDataSource(), TransactionIsolationLevel.READ_COMMITTED, false);
            // 创建执行器
            final Executor executor = configuration.newExecutor(tx);
            // 创建DefaultSqlSession
            return new DefaultSqlSession(configuration, executor);
        } catch (Exception e) {
            try {
                assert tx != null;
                tx.close();
            } catch (SQLException ignore) {
            }
            throw new RuntimeException("Error opening session.  Cause: " + e);
        }
    }
}
```

#### 执行器使用

有了执行器，SqlSession中的CRUD就可以直接调用Executor中的方法，从而实现解耦。

```java
public class DefaultSqlSession implements SqlSession {
    private Configuration configuration;
    private Executor executor;

    public DefaultSqlSession(Configuration configuration, Executor executor) {
        this.configuration = configuration;
        this.executor = executor;
    }

    @Override
    public <T> T selectOne(String statement) {
        return this.selectOne(statement, null);
    }

    @Override
    public <T> T selectOne(String statement, Object parameter) {
        MappedStatement ms = configuration.getMappedStatement(statement);
        List<T> list = executor.query(ms, parameter, Executor.NO_RESULT_HANDLER, ms.getBoundSql());
        return list.get(0);
    }

    @Override
    public <T> T getMapper(Class<T> type) {
        return configuration.getMapper(type, this);
    }

    @Override
    public Configuration getConfiguration() {
        return configuration;
    }
}
```

## 第七章 将反射运用到出神入化

观察实例化数据源时，对于相关属性采用的硬编码的方式读取，如果后期需要拓展字段，添加新的属性，这样做就很不利。

本章节的做法是，根据需要反射填充的对象`originalObject`生成一个元对象`MetaObject`。在元对象中进行拆解解析，将对象所属的类，对象的属性字段，对象的方法都拆解出来。

然后，在XMLConfigBuilder解析`<environment>`标签时，根据读取出来的属性字段，利用元对象，将这些字段填充进dataSource对象中。这样就不需要在DataSource类中以硬编码的方式定义这些属性值。后期即便有新的字段出现，只需要在XML文件中给出，就能够自动填充进对象中。


## 第八章 细化XML语句构造器，完善静态SQL解析

本章节，将从解析XML文件到解析SQL语句的功能分为三部分：解析XML文件加载数据源信息、解析XML文件生成代理Mapper对象、解析SQL语句。

其中，解析SQL语句又可以进行细分：解析SQL基本信息(参数类型、返回值类型、命令类型)、解析SQL语句的内容(静态文本、动态标签)。解析基本信息的功能放在`XMLStatementBuilder`来实现。解析SQL语句内容的功能交给`LanguageDriver`语言驱动来实现。

LanguageDriver的主要功能就是解析SQL语句内容，返回一个SqlSource对象。其依赖于`XMLScriptBuilder`脚本构建器，

> SqlSource对应的是一整个SQL语句，SqlNode对应的是一个动态标签。
> SqlSource的实现类包括：StaticSqlSource静态SQL语句、RawSqlSource动态SQL语句。
> SqlNode的实现类包括：StaticTextSqlNode静态SQL节点、MixedSqlNode动态SQL节点。后者是将所有的SqlNode放在一个List集合中。
> SqlNode需要实现一个`apply()`方法，而MixedSqlNode直接将内部的所有Node都调用自己的`apply()`。通过阅读源码发现，每个Node都是StaticTextSqlNode，其`apply()`方法就是调用`DynamicContext#appednSql`来将字符串拼接起来。



在之前的做法中，我们在XMLConfigBuilder中先执行`XMLConfigBuilder#environmentsElement`方法，解析环境信息。然后，根据`<mappers>`标签中指定的`Mapper.XML`文件的包路径，一一解析分析其中的SQL语句。这样将所有的处理逻辑都放在同一个循环中进行，十分耦合，且随着内容的增多会越来越臃肿。

我们现在应该将这些功能分开。首先，一个`datasource.xml`文件中主要存放的是数据源的基本信息，以及指定多个`Mapper.xml`文件的包路径。一个`Mapper.xml`文件中，有一个`<mapper>`标签作为根元素，标签内有一个属性`namespace`是唯一的，标签内包含多个SQL语句。

于是，我们按照功能来讲执行逻辑划分为三部分：
- `XMLConfigBuilder`负责解析数据源环境，并调用`XMLMapperBuilder`。
- `XMLMapperBuilder`负责解析一个`Mapper.xml`文件，记录命名空间，并调用`XMLStatementBuilder`。
- `XMLStatementBuilder`负责解析一条SQL语句，包括ID、参数类型、返回值类型等。


### 解耦映射解析

#### XMLConfigBuilder

负责解析`datasource.xml`文件。根元素为`<configuration>`标签。通过入口I/O的方式给定XML文件的路径。

核心逻辑为读取`<environments>`标签，解析数据源的配置。并根据`<mappers>`标签中指定的所有的`Mapper.xml`文件的包路径，一一创建`XMLMapperBuilder`对象，对其处理。

```java
public class XMLConfigBuilder extends BaseBuilder {
    private Element root;
    public XMLConfigBuilder(Reader reader) {
        // 1. 调用父类初始化Configuration
        super(new Configuration());
        // 2. dom4j 处理 xml
        SAXReader saxReader = new SAXReader();
        try {
            Document document = saxReader.read(new InputSource(reader));
            root = document.getRootElement();
        } catch (DocumentException e) {
            e.printStackTrace();
        }
    }

    /**
     * 解析配置；类型别名、插件、对象工厂、对象包装工厂、设置、环境、类型转换、映射器
     *
     * @return Configuration
     */
    public Configuration parse() {
        try {
            // 环境
            environmentsElement(root.element("environments"));
            // 解析映射器
            mapperElement(root.element("mappers"));
        } catch (Exception e) {
            throw new RuntimeException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
        }
        return configuration;
    }

    /**
     * <environments default="development">
     * <environment id="development">
     * <transactionManager type="JDBC">
     * <property name="..." value="..."/>
     * </transactionManager>
     * <dataSource type="POOLED">
     * <property name="driver" value="${driver}"/>
     * <property name="url" value="${url}"/>
     * <property name="username" value="${username}"/>
     * <property name="password" value="${password}"/>
     * </dataSource>
     * </environment>
     * </environments>
     */
    private void environmentsElement(Element context) throws Exception {
        String environment = context.attributeValue("default");
        List<Element> environmentList = context.elements("environment");
        for (Element e : environmentList) {
            String id = e.attributeValue("id");
            if (environment.equals(id)) {
                // 事务管理器
                TransactionFactory txFactory = (TransactionFactory) typeAliasRegistry.resolveAlias(e.element("transactionManager").attributeValue("type")).newInstance();
                // 数据源
                Element dataSourceElement = e.element("dataSource");
                DataSourceFactory dataSourceFactory = (DataSourceFactory) typeAliasRegistry.resolveAlias(dataSourceElement.attributeValue("type")).newInstance();
                List<Element> propertyList = dataSourceElement.elements("property");
                Properties props = new Properties();
                for (Element property : propertyList) {
                    props.setProperty(property.attributeValue("name"), property.attributeValue("value"));
                }
                dataSourceFactory.setProperties(props);
                DataSource dataSource = dataSourceFactory.getDataSource();
                // 构建环境
                Environment.Builder environmentBuilder = new Environment.Builder(id)
                        .transactionFactory(txFactory)
                        .dataSource(dataSource);
                configuration.setEnvironment(environmentBuilder.build());
            }
        }
    }
    /*
     * <mappers>
     *	 <mapper resource="org/mybatis/builder/AuthorMapper.xml"/>
     *	 <mapper resource="org/mybatis/builder/BlogMapper.xml"/>
     *	 <mapper resource="org/mybatis/builder/PostMapper.xml"/>
     * </mappers>
     */
    private void mapperElement(Element mappers) throws Exception {
        List<Element> mapperList = mappers.elements("mapper");
        for (Element e : mapperList) {
            String resource = e.attributeValue("resource"); 
            InputStream inputStream = Resources.getResourceAsStream(resource);
            // 在for循环里每个mapper都重新new一个XMLMapperBuilder，来解析
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource);
            mapperParser.parse();
        }
    }
}
```

#### XMLMapperBuilder

负责解析`Mapper.xml`文件。根元素为`<mapper>`标签。通过入口I/O的方式指定需要解析的文件。

核心逻辑为：首先，根据当前文件的包路径`resource`检查当前文件是否已经解析过(存储在Configuration中)。然后，记录namespace。最后，为每一个SQL语句实例化一个`XMLStatementBuilder`进行解析。

```java
public class XMLMapperBuilder extends BaseBuilder {
    private Element element;
    private String resource;
    private String currentNamespace;

    public XMLMapperBuilder(InputStream inputStream, Configuration configuration, String resource) throws DocumentException {
        this(new SAXReader().read(inputStream), configuration, resource);
    }

    private XMLMapperBuilder(Document document, Configuration configuration, String resource) {
        super(configuration);
        this.element = document.getRootElement();
        this.resource = resource;
    }
    /**
     * 解析
     */
    public void parse() throws Exception {
        // 如果当前资源没有加载过再加载，防止重复加载
        if (!configuration.isResourceLoaded(resource)) {
            configurationElement(element);
            // 标记一下，已经加载过了
            configuration.addLoadedResource(resource);
            // 绑定映射器到namespace
            configuration.addMapper(Resources.classForName(currentNamespace));
        }
    }
    // 配置mapper元素
    // <mapper namespace="org.mybatis.example.BlogMapper">
    //   <select id="selectBlog" parameterType="int" resultType="Blog">
    //    select * from Blog where id = #{id}
    //   </select>
    // </mapper>
    private void configurationElement(Element element) {
        // 1.配置namespace
        currentNamespace = element.attributeValue("namespace");
        if (currentNamespace.equals("")) {
            throw new RuntimeException("Mapper's namespace cannot be empty");
        }
        // 2.配置select|insert|update|delete
        buildStatementFromContext(element.elements("select"));
    }
    // 配置select|insert|update|delete
    private void buildStatementFromContext(List<Element> list) {
        for (Element element : list) {
            final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, element, currentNamespace);
            statementParser.parseStatementNode();
        }
    }
}
```

#### XMLStatementBuilder

负责解析一条SQL语句。根元素为`<select>`等标签。

核心逻辑为：解析SQL语句的ID、参数类型、返回值类型，并将后两者的类型映射为具体的Java类型。然后，获取一个语言驱动器实例`LanguageDriver`，在其内部解析SQL语句的内容，获得一个`SqlSource`实例。将SqlSource同ID、参数类型、返回值类型等信息封装到MappedStatement实例中，记录在Configuration里。

```java
public class XMLStatementBuilder extends BaseBuilder {
    private String currentNamespace;
    private Element element;

    public XMLStatementBuilder(Configuration configuration, Element element, String currentNamespace) {
        super(configuration);
        this.element = element;
        this.currentNamespace = currentNamespace;
    }
    // 解析语句(select|insert|update|delete)
    // <select id="selectPerson" parameterType="int" parameterMap="deprecated" resultType="hashmap" resultMap="personResultMap" flushCache="false" useCache="true" timeout="10000" fetchSize="256" statementType="PREPARED" resultSetType="FORWARD_ONLY"> 
    // SELECT * FROM PERSON WHERE ID = #{id} 
    // </select>
    public void parseStatementNode() {
        String id = element.attributeValue("id");
        // 参数类型
        String parameterType = element.attributeValue("parameterType");
        Class<?> parameterTypeClass = resolveAlias(parameterType);
        // 结果类型
        String resultType = element.attributeValue("resultType");
        Class<?> resultTypeClass = resolveAlias(resultType);
        // 获取命令类型(select|insert|update|delete)
        String nodeName = element.getName();
        SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
        // 获取默认语言驱动器
        Class<?> langClass = configuration.getLanguageRegistry().getDefaultDriverClass();
        LanguageDriver langDriver = configuration.getLanguageRegistry().getDriver(langClass);

        SqlSource sqlSource = langDriver.createSqlSource(configuration, element, parameterTypeClass);

        MappedStatement mappedStatement = new MappedStatement.Builder(configuration, currentNamespace + "." + id, sqlCommandType, sqlSource, resultTypeClass).build();
        // 添加解析 SQL
        configuration.addMappedStatement(mappedStatement);
    }
}
```

### 脚本语言驱动

上文中的XMLStatementBuilder只负责记录SQL语句的基本信息，具体的解析过程放在了语言驱动中实现。

XMLStatementBuilder中，对于当前的SQL语句，调用了`LanguageDriver#createSqlSource`方法，获取了一个SqlSource对象。

而我们实现的语言驱动器为XMLLanguageDriver，它直接封装了XMLScriptBuilder来完成该功能。

XMLScriptBuilder先将SQL语句中的所有标签都各自封装进一个StaticTextSqlNode对象，然后用List集合封装进MixedSqlNode对象。

再用MixedSqlNode生一个RawSqlSource对象。在RawSqlSource构造器中，就会实例化一个DynamicContext对象，将MixedSqlNode中的标签拼接起来转换为SQL字符串。

然后，实例化一个SqlSourceBuilder对象，负责将SQL字符串中`#{}`等占位符表示的参数的符号引用替换为真实引用，并将结果封装进SqlSource返回。

至此，一条SQL语句从XML中的定义解析为一个SqlSource对象的过程就完成了。

#### LanguageDriver

首先，对语言驱动接口给出定义。语言驱动的功能就是根据给定的参数和XML标签，生成SQL语句源代码对象。

```java
public interface LanguageDriver {
    SqlSource createSqlSource(Configuration configuration, Element script, Class<?> parameterType);
}
```

#### XMLLanguageDriver

语言驱动器的实现类，封装XMLScriptBuilder对语句的处理。

```java
public class XMLLanguageDriver implements LanguageDriver {
    @Override
    public SqlSource createSqlSource(Configuration configuration, Element script, Class<?> parameterType) {
        XMLScriptBuilder builder = new XMLScriptBuilder(configuration, script, parameterType);
        return builder.parseScriptNode();
    }
}
```

#### XMLScriptBuilder脚本构建器

核心功能为为一条SQL语句生成一个SqlSource实例对象。生成过程需要用到SQL源码构建器`SqlSourceBuilder`。

具体逻辑为：
- 先将所有的SQL标签(if、where等)，都各自封装进一个StaticTextSqlNode对象中，并用List集合存储。
- 然后，将List集合封装进一个MixedSqlNode对象。
- 再利用，MixedSqlNode去构建一个RawSqlSource对象，并返回。
- RawSqlSource的构建过程放在其章节描述。

```java
public class XMLScriptBuilder extends BaseBuilder {
    private Element element;
    private boolean isDynamic;
    private Class<?> parameterType;

    public XMLScriptBuilder(Configuration configuration, Element element, Class<?> parameterType) {
        super(configuration);
        this.element = element;
        this.parameterType = parameterType;
    }

    public SqlSource parseScriptNode() {
        List<SqlNode> contents = parseDynamicTags(element);
        MixedSqlNode rootSqlNode = new MixedSqlNode(contents);
        return new RawSqlSource(configuration, rootSqlNode, parameterType);
    }

    List<SqlNode> parseDynamicTags(Element element) {
        List<SqlNode> contents = new ArrayList<>();
        // element.getText 拿到 SQL
        String data = element.getText();
        contents.add(new StaticTextSqlNode(data));
        return contents;
    }
}
```

#### 语言驱动器注册

负责维护一个驱动器容器其，内部保持系统中语言驱动器的实例对象。需要用到时直接调用，不需要反复创建。

```java
public class LanguageDriverRegistry {

    // map
    private final Map<Class<?>, LanguageDriver> LANGUAGE_DRIVER_MAP = new HashMap<Class<?>, LanguageDriver>();

    private Class<?> defaultDriverClass = null;

    public void register(Class<?> cls) {
        if (cls == null) {
            throw new IllegalArgumentException("null is not a valid Language Driver");
        }
        if (!LanguageDriver.class.isAssignableFrom(cls)) {
            throw new RuntimeException(cls.getName() + " does not implements " + LanguageDriver.class.getName());
        }
        // 如果没注册过，再去注册
        LanguageDriver driver = LANGUAGE_DRIVER_MAP.get(cls);
        if (driver == null) {
            try {
                //单例模式，即一个Class只有一个对应的LanguageDriver
                driver = (LanguageDriver) cls.newInstance();
                LANGUAGE_DRIVER_MAP.put(cls, driver);
            } catch (Exception ex) {
                throw new RuntimeException("Failed to load language driver for " + cls.getName(), ex);
            }
        }
    }

    public LanguageDriver getDriver(Class<?> cls) {
        return LANGUAGE_DRIVER_MAP.get(cls);
    }

    public LanguageDriver getDefaultDriver() {
        return getDriver(getDefaultDriverClass());
    }

    public Class<?> getDefaultDriverClass() {
        return defaultDriverClass;
    }

    //Configuration()有调用，默认的为XMLLanguageDriver
    public void setDefaultDriverClass(Class<?> defaultDriverClass) {
        register(defaultDriverClass);
        this.defaultDriverClass = defaultDriverClass;
    }

}
```

### SQL源码构建器

#### RawSqlSource

SqlSource的实现类(SqlSource共有两种实现类StaticSqlSource对应没有参数的静态SQL，以及RawSqlSource对应有传入参数的动态SQL)。最终所有的SQL语句都会转换为一个StaticSqlSource对象。

构建RawSqlSource需要一个SqlNode实例对象，即将XML文件中的SQL语句内部标签(if、where)等都封装为对象。其构造逻辑如下：
- 在构造方法中，会实例化一个DynamicContext对象，对所有的SqlNode调用其`SqlNode#apply`方法，将它们都转换为字符串拼接起来，得到一个SQL字符串。
- 然后，实例化一个SqlSourceBuilder对象，调用其`parse()`方法，返回一个SqlSource对象，记录在当前RawSqlSource内部。

```java
public class RawSqlSource implements SqlSource {
    private final SqlSource sqlSource;

    public RawSqlSource(Configuration configuration, SqlNode rootSqlNode, Class<?> parameterType) {
        this(configuration, getSql(configuration, rootSqlNode), parameterType);
    }

    public RawSqlSource(Configuration configuration, String sql, Class<?> parameterType) {
        SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
        Class<?> clazz = parameterType == null ? Object.class : parameterType;
        sqlSource = sqlSourceParser.parse(sql, clazz, new HashMap<>());
    }

    @Override
    public BoundSql getBoundSql(Object parameterObject) {
        return sqlSource.getBoundSql(parameterObject);
    }

    private static String getSql(Configuration configuration, SqlNode rootSqlNode) {
        DynamicContext context = new DynamicContext(configuration, null);
        rootSqlNode.apply(context);
        return context.getSql();
    }
}
```

#### StaticSqlSource

负责记录解析好的SQL语句，以及其参数。

在StaticSqlSource中以List集合的方式，按顺序记录下了传入参数的映射关系(JdbcType和JavaType)，以便于执行SQL时按照映射关系传入参数。

```java
public class StaticSqlSource implements SqlSource {
    private String sql;
    private List<ParameterMapping> parameterMappings;
    private Configuration configuration;

    public StaticSqlSource(Configuration configuration, String sql) {
        this(configuration, sql, null);
    }

    public StaticSqlSource(Configuration configuration, String sql, List<ParameterMapping> parameterMappings) {
        this.sql = sql;
        this.parameterMappings = parameterMappings;
        this.configuration = configuration;
    }

    @Override
    public BoundSql getBoundSql(Object parameterObject) {
        return new BoundSql(configuration, sql, parameterMappings, parameterObject);
    }
}
```

#### SqlSourceBuilder

核心功能为将SQL字符串中的`#{}, ${}`符号引用，转换为`?`站位，并记录对应位置应该填入的参数类型。将结果封装进一个StaticSqlSource对象保存。

核心逻辑为：
- 实例化一个`GenericTokenParer`对象，让其负责处理`#{}, ${}`符号引用
- 遇到符号引用时，就会调用内部类`ParameterMappingTokenHandler#handleToken`方法，将其替换为`?`并记录参数类型，按顺序记录在List集合中。

```java
public class SqlSourceBuilder extends BaseBuilder {
    private static final String parameterProperties = "javaType,jdbcType,mode,numericScale,resultMap,typeHandler,jdbcTypeName";

    public SqlSourceBuilder(Configuration configuration) {
        super(configuration);
    }

    public SqlSource parse(String originalSql, Class<?> parameterType, Map<String, Object> additionalParameters) {
        ParameterMappingTokenHandler handler = new ParameterMappingTokenHandler(configuration, parameterType, additionalParameters);
        GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
        String sql = parser.parse(originalSql);
        // 返回静态 SQL
        return new StaticSqlSource(configuration, sql, handler.getParameterMappings());
    }

    private static class ParameterMappingTokenHandler extends BaseBuilder implements TokenHandler {

        private List<ParameterMapping> parameterMappings = new ArrayList<>();
        private Class<?> parameterType;
        private MetaObject metaParameters;

        public ParameterMappingTokenHandler(Configuration configuration, Class<?> parameterType, Map<String, Object> additionalParameters) {
            super(configuration);
            this.parameterType = parameterType;
            this.metaParameters = configuration.newMetaObject(additionalParameters);
        }

        public List<ParameterMapping> getParameterMappings() {
            return parameterMappings;
        }

        @Override
        public String handleToken(String content) {
            parameterMappings.add(buildParameterMapping(content));
            return "?";
        }
        // 构建参数映射
        private ParameterMapping buildParameterMapping(String content) {
            // 先解析参数映射,就是转化成一个 HashMap | #{favouriteSection,jdbcType=VARCHAR}
            Map<String, String> propertiesMap = new ParameterExpression(content);
            String property = propertiesMap.get("property");
            Class<?> propertyType = parameterType;
            ParameterMapping.Builder builder = new ParameterMapping.Builder(configuration, property, propertyType);
            return builder.build();
        }

    }
}
```

#### GenericTokenParser

负责处理`#{}, #{}`。

```java
public class GenericTokenParser {
    // 有一个开始和结束记号
    private final String openToken;
    private final String closeToken;
    // 记号处理器
    private final TokenHandler handler;

    public GenericTokenParser(String openToken, String closeToken, TokenHandler handler) {
        this.openToken = openToken;
        this.closeToken = closeToken;
        this.handler = handler;
    }

    public String parse(String text) {
        StringBuilder builder = new StringBuilder();
        if (text != null && text.length() > 0) {
            char[] src = text.toCharArray();
            int offset = 0;
            int start = text.indexOf(openToken, offset);
            // #{favouriteSection,jdbcType=VARCHAR}
            // 这里是循环解析参数，参考GenericTokenParserTest,比如可以解析${first_name} ${initial} ${last_name} reporting.这样的字符串,里面有3个${}
            while (start > -1) {
                //判断一下 ${ 前面是否是反斜杠，这个逻辑在老版的mybatis中（如3.1.0）是没有的
                if (start > 0 && src[start - 1] == '\\') {
                    // the variable is escaped. remove the backslash.
                    // 新版已经没有调用substring了，改为调用如下的offset方式，提高了效率
                    builder.append(src, offset, start - offset - 1).append(openToken);
                    offset = start + openToken.length();
                } else {
                    int end = text.indexOf(closeToken, start);
                    if (end == -1) {
                        builder.append(src, offset, src.length - offset);
                        offset = src.length;
                    } else {
                        builder.append(src, offset, start - offset);
                        offset = start + openToken.length();
                        String content = new String(src, offset, end - offset);
                        // 得到一对大括号里的字符串后，调用handler.handleToken,比如替换变量这种功能
                        builder.append(handler.handleToken(content));
                        offset = end + closeToken.length();
                    }
                }
                start = text.indexOf(openToken, offset);
            }
            if (offset < src.length) {
                builder.append(src, offset, src.length - offset);
            }
        }
        return builder.toString();
    }
}
```

## 第九章 使用策略模式，调用参数处理器

在之前的做法中，我们已经将SQL语句中动态传参的部分替换为了`?`，并在`ParameterMappings`记录了参数映射类型。在执行SQL时，会通过`StatementHandler#parameterize`方法来将占位符替换为参数。

但是，这么做是通过硬编码的方式来实现的，比如在`PreparedStatementHandler#parameterize`中，我们直接使用`setLong()`方法来将Long类型参数传入。我们需要对这一部分进行处理，将硬编码转换为自动类型处理。因为真实情况下，参数类型有多种，不可能为每一个类型的参数重载一个方法，这样也不方便后期扩展。


### 入参校准

之前的做法中，在`MappedMethod#execute`方法中直接将代理对象`invoke()`方法的入参`args[]`传入了SqlSession的对应方法中。直接使用数组类型来进行参数映射不方便，因为我们不知道会传入几个参数。

因此，我们在代理对象执行`invoke()`时，就将参数`args[]`进行处理。

#### MapperMethod

该类对应的是Dao接口中的一个方法，存储了方法的方法签名和对应的SQL指令的ID和类型。

当程序调用`Dao#select`方法时，会通过Mapper代理对象执行`Mapper#invoke`方法。在`invoke()`内部调用该方法对应的`MapperMethod#excute`方法。在`excute()`方法内部，根据方法对应的SQL类型来执行SqlSession中的对应方法。

先前的做法是直接将`invoke()`的入参`args[]`传给`SqlSession#select`。而现在，我们要先对入参进行处理，将数组类型转换为对象。

首先，定义一个内部类`MethodSignature`用来解析方法签名。直接根据用户调用的Dao接口的方法签名，来解析`args[]`数组中到底应该是什么对象。

使用一个``Map<String, Object>`集合来按照顺序记录数组中的对象。即集合的key为字符串`0, 1, 2, ... `，value为对应位置的参数对象。

于是，在调用SqlSession具体方法时，如果入参是单个对象，直接传入该对象；如果入参是多个对象，我们传入的是一个Map集合。
- 那么，在最终由`StatementHandler#parameterize`给SQL传入参数时，怎么知道到底接收到的参数是一个，还是多个呢。这个就需要用到ParameterHandler，TypeHandler，以及其注册机TypeHandlerRegistry。

```java
public class MapperMethod {
    private final SqlCommand command;
    private final MethodSignature method;

    public MapperMethod(Class<?> mapperInterface, Method method, Configuration configuration) {
        this.command = new SqlCommand(configuration, mapperInterface, method);
        this.method = new MethodSignature(configuration, method);
    }

    public Object execute(SqlSession sqlSession, Object[] args) {
        Object result = null;
        switch (command.getType()) {
            case INSERT:
                break;
            case DELETE:
                break;
            case UPDATE:
                break;
            case SELECT:
                Object param = method.convertArgsToSqlCommandParam(args);
                result = sqlSession.selectOne(command.getName(), param);
                break;
            default:
                throw new RuntimeException("Unknown execution method for: " + command.getName());
        }
        return result;
    }
    /**
     * SQL 指令
     */
    public static class SqlCommand {
        private final String name;
        private final SqlCommandType type;

        public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
            String statementName = mapperInterface.getName() + "." + method.getName();
            MappedStatement ms = configuration.getMappedStatement(statementName);
            name = ms.getId();
            type = ms.getSqlCommandType();
        }

        public String getName() {
            return name;
        }

        public SqlCommandType getType() {
            return type;
        }
    }
    /**
     * 方法签名
     */
    public static class MethodSignature {
        private final SortedMap<Integer, String> params;

        public MethodSignature(Configuration configuration, Method method) {
            this.params = Collections.unmodifiableSortedMap(getParams(method));
        }

        public Object convertArgsToSqlCommandParam(Object[] args) {
            final int paramCount = params.size();
            if (args == null || paramCount == 0) {
                //如果没参数
                return null;
            } else if (paramCount == 1) {
                return args[params.keySet().iterator().next().intValue()];
            } else {
                // 否则，返回一个ParamMap，修改参数名，参数名就是其位置
                final Map<String, Object> param = new ParamMap<Object>();
                int i = 0;
                for (Map.Entry<Integer, String> entry : params.entrySet()) {
                    // 1.先加一个#{0},#{1},#{2}...参数
                    param.put(entry.getValue(), args[entry.getKey().intValue()]);
                    // issue #71, add param names as param1, param2...but ensure backward compatibility
                    final String genericParamName = "param" + (i + 1);
                    if (!param.containsKey(genericParamName)) {
                        /*
                         * 2.再加一个#{param1},#{param2}...参数
                         * 你可以传递多个参数给一个映射器方法。如果你这样做了,
                         * 默认情况下它们将会以它们在参数列表中的位置来命名,比如:#{param1},#{param2}等。
                         * 如果你想改变参数的名称(只在多参数情况下) ,那么你可以在参数上使用@Param(“paramName”)注解。
                         */
                        param.put(genericParamName, args[entry.getKey()]);
                    }
                    i++;
                }
                return param;
            }
        }

        private SortedMap<Integer, String> getParams(Method method) {
            // 用一个TreeMap，这样就保证还是按参数的先后顺序
            final SortedMap<Integer, String> params = new TreeMap<Integer, String>();
            final Class<?>[] argTypes = method.getParameterTypes();
            for (int i = 0; i < argTypes.length; i++) {
                String paramName = String.valueOf(params.size());
                // 不做 Param 的实现，这部分不处理。如果扩展学习，需要添加 Param 注解并做扩展实现。
                params.put(i, paramName);
            }
            return params;
        }
    }
    /**
     * 参数map，静态内部类,更严格的get方法，如果没有相应的key，报错
     */
    public static class ParamMap<V> extends HashMap<String, V> {
        private static final long serialVersionUID = -2212268410512043556L;

        @Override
        public V get(Object key) {
            if (!super.containsKey(key)) {
                throw new RuntimeException("Parameter '" + key + "' not found. Available parameters are " + keySet());
            }
            return super.get(key);
        }
    }
}
```

### 参数策略处理器

负责给SQL语句传入参数的具体逻辑。

由于参数种类不同，所以我们先定义一个接口约束功能，然后再根据具体的类型给出实现类即可。

#### TypeHandler

通过方法签名可以看出，TypeHandler的核心功能就是：给SQL语句中第i个位置的参数，传入具体的数值。

但TypHandler是一个接口，且参数类型是一个泛型，那么传入参数时具体调用哪个实现类的方法？这就需要用到一个注册机，根据JdbcType和JavaType来确定用哪个类型处理器。

```java
public interface TypeHandler<T> {
    /**
     * 设置参数
     */
    void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException;
}
```

#### BaseTypeHandler

抽象类，负责抽取传参的公共逻辑。

```java
public abstract class BaseTypeHandler<T> implements TypeHandler<T> {
    protected Configuration configuration;

    public void setConfiguration(Configuration configuration) {
        this.configuration = configuration;
    }

    @Override
    public void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException {
        // 定义抽象方法，由子类实现不同类型的属性设置
        setNonNullParameter(ps, i, parameter, jdbcType);
    }

    protected abstract void setNonNullParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException;
}
```

#### LongTypeHandler

具体的类型处理器，负责给SQL传入Long类型的数据。

给SQL语句第i个参数，传入Long类型数据。

```java
public class LongTypeHandler extends BaseTypeHandler<Long> {
    @Override
    protected void setNonNullParameter(PreparedStatement ps, int i, Long parameter, JdbcType jdbcType) throws SQLException {
        ps.setLong(i, parameter);
    }
}
```

#### 类型处理器注册机TypeHanlderRegistry

负责注册TypeHandler。

TypeHandler的`setParameter()`负责给SQL语句的第i个参数赋值。但是，我们根据参数类型有着不同的TypeHanlder实现类，那么具体使用时应该调用哪个实现类来提供服务，就需要通道注册机。

```java
public final class TypeHandlerRegistry {
    private final Map<JdbcType, TypeHandler<?>> JDBC_TYPE_HANDLER_MAP = new EnumMap<>(JdbcType.class);
    private final Map<Type, Map<JdbcType, TypeHandler<?>>> TYPE_HANDLER_MAP = new HashMap<>();
    private final Map<Class<?>, TypeHandler<?>> ALL_TYPE_HANDLERS_MAP = new HashMap<>();

    public TypeHandlerRegistry() {
        register(Long.class, new LongTypeHandler());
        register(long.class, new LongTypeHandler());

        register(String.class, new StringTypeHandler());
        register(String.class, JdbcType.CHAR, new StringTypeHandler());
        register(String.class, JdbcType.VARCHAR, new StringTypeHandler());
    }

    private <T> void register(Type javaType, TypeHandler<? extends T> typeHandler) {
        register(javaType, null, typeHandler);
    }

    private void register(Type javaType, JdbcType jdbcType, TypeHandler<?> handler) {
        if (null != javaType) {
            Map<JdbcType, TypeHandler<?>> map = TYPE_HANDLER_MAP.computeIfAbsent(javaType, k -> new HashMap<>());
            map.put(jdbcType, handler);
        }
        ALL_TYPE_HANDLERS_MAP.put(handler.getClass(), handler);
    }

    @SuppressWarnings("unchecked")
    public <T> TypeHandler<T> getTypeHandler(Class<T> type, JdbcType jdbcType) {
        return getTypeHandler((Type) type, jdbcType);
    }

    public boolean hasTypeHandler(Class<?> javaType) {
        return hasTypeHandler(javaType, null);
    }

    public boolean hasTypeHandler(Class<?> javaType, JdbcType jdbcType) {
        return javaType != null && getTypeHandler((Type) javaType, jdbcType) != null;
    }

    private <T> TypeHandler<T> getTypeHandler(Type type, JdbcType jdbcType) {
        Map<JdbcType, TypeHandler<?>> jdbcHandlerMap = TYPE_HANDLER_MAP.get(type);
        TypeHandler<?> handler = null;
        if (jdbcHandlerMap != null) {
            handler = jdbcHandlerMap.get(jdbcType);
            if (handler == null) {
                handler = jdbcHandlerMap.get(null);
            }
        }
        // type drives generics here
        return (TypeHandler<T>) handler;
    }
}
```

### 参数构造

我们是通过`ParamenterMapping`类来记录JdbcType和JavaType的映射关系的，那么在进行映射时就应该根据参数类型来对TypeHandler进行注册。

```java
public ParameterMapping build() {
    if (parameterMapping.typeHandler == null && parameterMapping.javaType != null) {
        Configuration configuration = parameterMapping.configuration;
        TypeHandlerRegistry typeHandlerRegistry = configuration.getTypeHandlerRegistry();
        parameterMapping.typeHandler = typeHandlerRegistry.getTypeHandler(parameterMapping.javaType, parameterMapping.jdbcType);
    }
    return parameterMapping;
}
```

另外，ParameterMapping的构造是在`SqlSourceBuilder`调用`GenericTokenParser#parse`时，执行其内部类`ParameterMappingTokenHandler#handleToken`来实现的。

```java
 @Override
public String handleToken(String content) {
    parameterMappings.add(buildParameterMapping(content));
    return "?";
}
```

## 第十章 流程解耦，封装结果处理器

在前面的章节中，虽然解析XML文件后得到了`resultType`属性，但是在处理返回结果时没有用到。虽然定义了ResultHandler接口，且在实例化执行器Excutor时也作为参数传入，但是一直没有实现对结果的处理逻辑。

JDBC的查询结果ResultSet是可以根据不同的属性类型返回结果的，不一定必须返回Object类型对象。

本章节，我们将通过XML文件解析返回结果的类型，并封装到MappedStatement类中。这样在获取到结果后，就可以按照参数类型创建对象，并使用MetaObject反射填充属性。

### 出参参数处理

#### ResultMap结果映射封装类

在Mybatis中，一条SQL语句会有一个返回类型配置，可以是`resultType`属性，也可以是`resultMap`属性来指定。但无论使用哪种，我们都同意封装到ResultMap结果映射类中。

```java
public class ResultMap {
    private String id;
    private Class<?> type;
    private List<ResultMapping> resultMappings;
    private Set<String> mappedColumns;

    private ResultMap() {
    }

    public static class Builder {
        private ResultMap resultMap = new ResultMap();

        public Builder(Configuration configuration, String id, Class<?> type, List<ResultMapping> resultMappings) {
            resultMap.id = id;
            resultMap.type = type;
            resultMap.resultMappings = resultMappings;
        }

        public ResultMap build() {
            resultMap.mappedColumns = new HashSet<>();
            return resultMap;
        }
    }
}
```

#### MapperBuilderAssistant构建器助手

我们在XMLStatementBuilder类中对SQL语句的信息进行解析，并调用语言驱动来解析具体的SQL内容。最终得到的结果需要一并封装到MappedStatement对象中。由于解析过程已经在该类中实现，如果将封装过程也一并在此实现，就会显得比较臃肿，且耦合度太高。

于是，我们将SQL语句解析结果的封装映射过程提取到MapperBuilderAssistant类中实现。

```java
public class MapperBuilderAssistant extends BaseBuilder {
    private String currentNamespace;
    private String resource;

    public MapperBuilderAssistant(Configuration configuration, String resource) {
        super(configuration);
        this.resource = resource;
    }

    public String getCurrentNamespace() {
        return currentNamespace;
    }

    public void setCurrentNamespace(String currentNamespace) {
        this.currentNamespace = currentNamespace;
    }

    public String applyCurrentNamespace(String base, boolean isReference) {
        if (base == null) {
            return null;
        }
        if (isReference) {
            if (base.contains(".")) return base;
        }
        return currentNamespace + "." + base;
    }
    /**
     * 添加映射器语句
     */
    public MappedStatement addMappedStatement(
            String id,
            SqlSource sqlSource,
            SqlCommandType sqlCommandType,
            Class<?> parameterType,
            String resultMap,
            Class<?> resultType,
            LanguageDriver lang
    ) {
        // 给id加上namespace前缀：cn.bugstack.mybatis.test.dao.IUserDao.queryUserInfoById
        id = applyCurrentNamespace(id, false);
        MappedStatement.Builder statementBuilder = new MappedStatement.Builder(configuration, id, sqlCommandType, sqlSource, resultType);

        // 结果映射，给 MappedStatement#resultMaps
        setStatementResultMap(resultMap, resultType, statementBuilder);

        MappedStatement statement = statementBuilder.build();
        // 映射语句信息，建造完存放到配置项中
        configuration.addMappedStatement(statement);

        return statement;
    }

    private void setStatementResultMap(
            String resultMap,
            Class<?> resultType,
            MappedStatement.Builder statementBuilder) {
        // 因为暂时还没有在 Mapper XML 中配置 Map 返回结果，所以这里返回的是 null
        resultMap = applyCurrentNamespace(resultMap, true);

        List<ResultMap> resultMaps = new ArrayList<>();

        if (resultMap != null) {
            // TODO：暂无Map结果映射配置，本章节不添加此逻辑
        }
        /*
         * 通常使用 resultType 即可满足大部分场景
         * <select id="queryUserInfoById" resultType="cn.bugstack.mybatis.test.po.User">
         * 使用 resultType 的情况下，Mybatis 会自动创建一个 ResultMap，基于属性名称映射列到 JavaBean 的属性上。
         */
        else if (resultType != null) {
            ResultMap.Builder inlineResultMapBuilder = new ResultMap.Builder(
                    configuration,
                    statementBuilder.id() + "-Inline",
                    resultType,
                    new ArrayList<>());
            resultMaps.add(inlineResultMapBuilder.build());
        }
        statementBuilder.resultMaps(resultMaps);
    }
}
```

#### 构建器助手的使用

在XMLStatementBuilder中对SQL解析完成后，将解析出来的信息传入构建器助手来封装。

```java
// 调用助手类【本节新添加，便于统一处理参数的包装】
builderAssistant.addMappedStatement(id,
    sqlSource,
    sqlCommandType,
    parameterTypeClass,
    resultMap,
    resultTypeClass,
    langDriver);
```

### 查询结果封装

JDBC插叙数据库的结果是一个List集合，于是，我们将对结果集合的处理交给`ResultSetHandler`来处理。然后对于集合中每一条记录，分别调用`ResultHandler`来处理。

处理逻辑包括生成对象，属性填充和加入集合。

#### ResultSetHandler

在`createResultObject()`方法中，根据解析XML文件得来的返回类型创建对应的MetaObject元对象，然后通过反射来进行属性填充。

在`applyAutomaticMappings()`方法中，对元对象的属性进行填充。

```java
public class DefaultResultSetHandler implements ResultSetHandler {

    private final Configuration configuration;
    private final MappedStatement mappedStatement;
    private final RowBounds rowBounds;
    private final ResultHandler resultHandler;
    private final BoundSql boundSql;
    private final TypeHandlerRegistry typeHandlerRegistry;
    private final ObjectFactory objectFactory;

    public DefaultResultSetHandler(Executor executor, MappedStatement mappedStatement, ResultHandler resultHandler, RowBounds rowBounds, BoundSql boundSql) {
        this.configuration = mappedStatement.getConfiguration();
        this.rowBounds = rowBounds;
        this.boundSql = boundSql;
        this.mappedStatement = mappedStatement;
        this.resultHandler = resultHandler;
        this.objectFactory = configuration.getObjectFactory();
        this.typeHandlerRegistry = configuration.getTypeHandlerRegistry();
    }

    @Override
    @SuppressWarnings("unchecked")
    public List<Object> handleResultSets(Statement stmt) throws SQLException {
        final List<Object> multipleResults = new ArrayList<>();

        int resultSetCount = 0;
        ResultSetWrapper rsw = new ResultSetWrapper(stmt.getResultSet(), configuration);

        List<ResultMap> resultMaps = mappedStatement.getResultMaps();
        while (rsw != null && resultMaps.size() > resultSetCount) {
            ResultMap resultMap = resultMaps.get(resultSetCount);
            handleResultSet(rsw, resultMap, multipleResults, null);
            rsw = getNextResultSet(stmt);
            resultSetCount++;
        }

        return multipleResults.size() == 1 ? (List<Object>) multipleResults.get(0) : multipleResults;
    }

    private ResultSetWrapper getNextResultSet(Statement stmt) throws SQLException {
        // Making this method tolerant of bad JDBC drivers
        try {
            if (stmt.getConnection().getMetaData().supportsMultipleResultSets()) {
                // Crazy Standard JDBC way of determining if there are more results
                if (!((!stmt.getMoreResults()) && (stmt.getUpdateCount() == -1))) {
                    ResultSet rs = stmt.getResultSet();
                    return rs != null ? new ResultSetWrapper(rs, configuration) : null;
                }
            }
        } catch (Exception ignore) {
            // Intentionally ignored.
        }
        return null;
    }

    private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
        if (resultHandler == null) {
            // 1. 新创建结果处理器
            DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
            // 2. 封装数据
            handleRowValuesForSimpleResultMap(rsw, resultMap, defaultResultHandler, rowBounds, null);
            // 3. 保存结果
            multipleResults.add(defaultResultHandler.getResultList());
        }
    }

    private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
        DefaultResultContext resultContext = new DefaultResultContext();
        while (resultContext.getResultCount() < rowBounds.getLimit() && rsw.getResultSet().next()) {
            Object rowValue = getRowValue(rsw, resultMap);
            callResultHandler(resultHandler, resultContext, rowValue);
        }
    }

    private void callResultHandler(ResultHandler resultHandler, DefaultResultContext resultContext, Object rowValue) {
        resultContext.nextResultObject(rowValue);
        resultHandler.handleResult(resultContext);
    }

    /**
     * 获取一行的值
     */
    private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap) throws SQLException {
        // 根据返回类型，实例化对象
        Object resultObject = createResultObject(rsw, resultMap, null);
        if (resultObject != null && !typeHandlerRegistry.hasTypeHandler(resultMap.getType())) {
            final MetaObject metaObject = configuration.newMetaObject(resultObject);
            applyAutomaticMappings(rsw, resultMap, metaObject, null);
        }
        return resultObject;
    }

    private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix) throws SQLException {
        final List<Class<?>> constructorArgTypes = new ArrayList<>();
        final List<Object> constructorArgs = new ArrayList<>();
        return createResultObject(rsw, resultMap, constructorArgTypes, constructorArgs, columnPrefix);
    }

    /**
     * 创建结果
     */
    private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix) throws SQLException {
        final Class<?> resultType = resultMap.getType();
        final MetaClass metaType = MetaClass.forClass(resultType);
        if (resultType.isInterface() || metaType.hasDefaultConstructor()) {
            // 普通的Bean对象类型
            return objectFactory.create(resultType);
        }
        throw new RuntimeException("Do not know how to create an instance of " + resultType);
    }

    private boolean applyAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix) throws SQLException{
        final List<String> unmappedColumnNames = rsw.getUnmappedColumnNames(resultMap, columnPrefix);
        boolean foundValues = false;
        for (String columnName : unmappedColumnNames) {
            String propertyName = columnName;
            if (columnPrefix != null && !columnPrefix.isEmpty()) {
                // When columnPrefix is specified,ignore columns without the prefix.
                if (columnName.toUpperCase(Locale.ENGLISH).startsWith(columnPrefix)) {
                    propertyName = columnName.substring(columnPrefix.length());
                } else {
                    continue;
                }
            }
            final String property = metaObject.findProperty(propertyName, false);
            if (property != null && metaObject.hasSetter(property)) {
                final Class<?> propertyType = metaObject.getSetterType(property);
                if (typeHandlerRegistry.hasTypeHandler(propertyType)) {
                    final TypeHandler<?> typeHandler = rsw.getTypeHandler(propertyType, columnName);
                    // 使用 TypeHandler 取得结果
                    final Object value = typeHandler.getResult(rsw.getResultSet(), columnName);
                    if (value != null) {
                        foundValues = true;
                    }
                    if (value != null || !propertyType.isPrimitive()) {
                        // 通过反射工具类设置属性值
                        metaObject.setValue(property, value);
                    }
                }
            }
        }
        return foundValues;
    }
}
```

#### ResultHandler

功能就是将ResultSetHandler处理好的单个记录对应的对象，加入到结果集合中。

```java
public class DefaultResultHandler implements ResultHandler {

    private final List<Object> list;

    public DefaultResultHandler() {
        this.list = new ArrayList<>();
    }

    /**
     * 通过 ObjectFactory 反射工具类，产生特定的 List
     */
    @SuppressWarnings("unchecked")
    public DefaultResultHandler(ObjectFactory objectFactory) {
        this.list = objectFactory.create(List.class);
    }

    @Override
    public void handleResult(ResultContext context) {
        list.add(context.getResultObject());
    }

    public List<Object> getResultList() {
        return list;
    }

}
```

## 第十二章 通过注解配置执行SQL语句

Mybatis支持通过给方法添加注解的方式来绑定方法对应的SQL语句。如果通过注解的方式来配置SQL语句，那么在`datasource.xml`文件中需要通过`<mapper>`标签的`class=""`属性来指定被注解配置的类的全路径，即Dao接口的全路径。

于是，我们解析XML文件时就需要分别处理通过`Mapper.xml`和通过注解配置SQL的逻辑。

### Mapper XML 解析调用

#### XMLConfigBuilder

在XMLConfigBuilder解析`datasource.xml`文件时，通过读取`<mapper>`标签的`resource`和`class`属性，来判断应该调用哪部分逻辑。如果定义了`resouce`说明应该调用XMLMapperBuilder来解析`Mapper.xml`文件；如果定义了`class`属性，应该调用`MapperRegistry#addMapper`(该方法被封装在了Configuration中)。

```java
public class XMLConfigBuilder extends BaseBuilder {
    // ... 省略部分代码
    /*
    <mappers>
        <mapper resource="mapper/User_Mapper.xml"/>
        <mapper class="cn.bugstack.mybatis.test.dao.IUserDao"/>
    </mappers>
     */
    private void mapperElement(Element mappers) throws Exception {
        List<Element> mapperList = mappers.elements("mapper");
        for (Element e : mapperList) {
            String resource = e.attributeValue("resource");
            String mapperClass = e.attributeValue("class");
            // XML 解析
            if (resource != null && mapperClass == null) {
                InputStream inputStream = Resources.getResourceAsStream(resource);
                // 在for循环里每个mapper都重新new一个XMLMapperBuilder，来解析
                XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource);
                mapperParser.parse();
            } else if (resource == null && mapperClass != null) { // Annotation 注解解析
                Class<?> mapperInterface = Resources.classForName(mapperClass);
                configuration.addMapper(mapperInterface);
            }
        }
    }
}
```

#### MapperRegistry

在`addMapper()`方法中，判断给出的被注解修饰的类是否为接口，然后调用`MapperAnnotationBuilder`来接口文件中所有解析方法上的注解。

```java
public <T> void addMapper(Class<T> type) {
    /* Mapper 必须是接口才会注册 */
    if (type.isInterface()) {
        if (hasMapper(type)) {
            // 如果重复添加了，报错
            throw new RuntimeException("Type " + type + " is already known to the MapperRegistry.");
        }
        // 注册映射器代理工厂
        knownMappers.put(type, new MapperProxyFactory<>(type));
        // 解析注解类语句配置
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
    }
}
```

### 注解配置构建器

在Mybatis框架的实现中，有专门一个annotations注解包，来提供用于配置到DAO方法上的注解，这些注解包括了所有的增删改查操作，同时可以设定一些额外的返回参数等。

#### 注解定义

给出CRUD四种注解的定义。生命周期都是运行时存在，作用范围都是方法注解。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Insert {

    String[] value();

}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Delete {

    String[] value();
    
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Update {

    String[] value();

}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Select {

    String[] value();

}
```

#### 配置解析MapperAnnotationBuilder

在构造器中需要拿到四个注解的Class对象，便于后续解析时判断SQL类型。以及Dao接口的全路径名，作为命名空间，给SQL语句的ID添加前缀。

然后在`parse()`方法中，通过Dao接口的Class对象获取接口中定义的所有方法，一一进行解析。

```java
public MapperAnnotationBuilder(Configuration configuration, Class<?> type) {
    String resource = type.getName().replace(".", "/") + ".java (best guess)";
    this.assistant = new MapperBuilderAssistant(configuration, resource);
    this.configuration = configuration;
    this.type = type;
    
    sqlAnnotationTypes.add(Select.class);
    sqlAnnotationTypes.add(Insert.class);
    sqlAnnotationTypes.add(Update.class);
    sqlAnnotationTypes.add(Delete.class);
}

public void parse() {
    String resource = type.toString();
    if (!configuration.isResourceLoaded(resource)) {
        assistant.setCurrentNamespace(type.getName());
        Method[] methods = type.getMethods();
        for (Method method : methods) {
            if (!method.isBridge()) {
                // 解析语句
                parseStatement(method);
            }
        }
    }
}
```

解析方法的大致流程和解析XML文件中SQL标签类似，都是解析出ID，入参类型和返回结果类型，并通过语言驱动器来为SQL生成一个SqlSource对象，最后一起封装到MappedStatement中。

最大的不同是获取SqlSource是通过解析方法上的注解来获取SQL语句的，调用的是重载的`LanguageDriver#createSqlSource`方法。

```java
private void parseStatement(Method method) {
    Class<?> parameterTypeClass = getParameterType(method);
    LanguageDriver languageDriver = getLanguageDriver(method);
    SqlSource sqlSource = getSqlSourceFromAnnotations(method, parameterTypeClass, languageDriver);
    if (sqlSource != null) {
        final String mappedStatementId = type.getName() + "." + method.getName();
        SqlCommandType sqlCommandType = getSqlCommandType(method);
        boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
        String resultMapId = null;
        if (isSelect) {
            resultMapId = parseResultMap(method);
        }
        
        // 调用助手类
        assistant.addMappedStatement(
                mappedStatementId,
                sqlSource,
                sqlCommandType,
                parameterTypeClass,
                resultMapId,
                getReturnType(method),
                languageDriver
        );
    }
}
```

### 语言驱动器

#### XMLLanguageDriver

```java
@Override
public SqlSource createSqlSource(Configuration configuration, String script, Class<?> parameterType) {
    // 暂时不解析动态 SQL
    return new RawSqlSource(configuration, script, parameterType);
}
```

## 第十三章 解析和使用ResultMap映射参数配置

通常，在数据库的属性字段命名时，会采用小写字母加下户线分割的方式。但是，在Java程序中的Pojo类定义时，对应的字段会采用驼峰命名的方式。那么，再进行数据库查询时，我们需要将数据库的字段名称正确映射到Java代码中的驼峰字段，才能够将查询结果正确映射到返回对象上。虽然，我们可以在Mybatis中使用`employee_name as emmployeeName`这样的方式来实现，但是不够优雅。

本章节，我们通过ResultMap来实现命名的映射。

### 解析映射配置

我们可以在`Mapper.xml`文件中，添加`<resultMap>`标签来完成命名属性的映射。

#### 解析入口

为了能够解析`<resultMap>`标签，我们需要在XMLMapperBuilder类中，添加解析resultMap的操作。这部分操作需要放在解析select等标签之前。因为，select标签的返回值类型会调用resultMap标签。

```java
private void configurationElement(Element element) {
    // 1.配置namespace
    String namespace = element.attributeValue("namespace");
    if (namespace.equals("")) {
        throw new RuntimeException("Mapper's namespace cannot be empty");
    }
    builderAssistant.setCurrentNamespace(namespace);
    // 2. 解析resultMap
    resultMapElements(element.elements("resultMap"));
    // 3.配置select|insert|update|delete
    buildStatementFromContext(element.elements("select"),
            element.elements("insert"),
            element.elements("update"),
            element.elements("delete")
    );
}
```

#### 解析过程

对resultMaps中每一个`<resultMap>`分别进行解析。

```java
private void resultMapElements(List<Element> list) {
    for (Element element : list) {
        try {
            resultMapElement(element, Collections.emptyList());
        } catch (Exception ignore) {
        }
    }
}
```

根据`type`属性指定的全类名，获取到返回对象的的类Class对象。

然后，为每一条需要映射的属性，构建一个`ResultMapping`对象，并将所有的该类型对象记录在一个List集合中。

最后，和本条resultMap标签的id一起，封装到一个ResultMap对象中。便完成了一个返回类型的字段映射解析。

```java
private ResultMap resultMapElement(Element resultMapNode, List<ResultMapping> additionalResultMappings) throws Exception {
    String id = resultMapNode.attributeValue("id");
    String type = resultMapNode.attributeValue("type");
    Class<?> typeClass = resolveClass(type);

    List<ResultMapping> resultMappings = new ArrayList<>();
    resultMappings.addAll(additionalResultMappings);

    List<Element> resultChildren = resultMapNode.elements();
    for (Element resultChild : resultChildren) {
        List<ResultFlag> flags = new ArrayList<>();
        if ("id".equals(resultChild.getName())) {
            flags.add(ResultFlag.ID);
        }
        // 构建 ResultMapping
        resultMappings.add(buildResultMappingFromContext(resultChild, typeClass, flags));
    }

    // 创建结果映射解析器
    ResultMapResolver resultMapResolver = new ResultMapResolver(builderAssistant, id, typeClass, resultMappings);
    return resultMapResolver.resolve();
}

private ResultMapping buildResultMappingFromContext(Element context, Class<?> resultType, List<ResultFlag> flags) throws Exception {
    String property = context.attributeValue("property");
    String column = context.attributeValue("column");
    return builderAssistant.buildResultMapping(resultType, property, column, flags);
}
```

### 存放映射对象

#### 映射解析

ResultMapResolver 结果映射器是本章节新增加的内容，其实它的作用就是对解析结果内容的一个封装处理，最终调用的是还是 MapperBuilderAssistant 映射构建器助手，所提供 ResultMap 封装和保存操作。

```java
public class ResultMapResolver {
    private final MapperBuilderAssistant assistant;
    private String id;
    private Class<?> type;
    private List<ResultMapping> resultMappings;
	// ... 省略构造函数
    public ResultMap resolve() {
        return assistant.addResultMap(this.id, this.type, this.resultMappings);
    }
}
```

#### 封装ResultMap

```java
public class ResultMap {
    private String id;
    private Class<?> type;
    private List<ResultMapping> resultMappings;
    private Set<String> mappedColumns;

    private ResultMap() {}

    public static class Builder {
        private ResultMap resultMap = new ResultMap();

        public Builder(Configuration configuration, String id, Class<?> type, List<ResultMapping> resultMappings) {
            resultMap.id = id;
            resultMap.type = type;
            resultMap.resultMappings = resultMappings;
        }

        public ResultMap build() {
            resultMap.mappedColumns = new HashSet<>();
           
           // 添加 mappedColumns 字段
            for (ResultMapping resultMapping : resultMap.resultMappings) {
                final String column = resultMapping.getColumn();
                if (column != null) {
                    resultMap.mappedColumns.add(column.toUpperCase(Locale.ENGLISH));
                }
            }
            return resultMap;
        }
    }
}
```

#### 添加ResultMap

```java
public ResultMap addResultMap(String id, Class<?> type, List<ResultMapping> resultMappings) {
    // 补全ID全路径，如：cn.bugstack.mybatis.test.dao.IActivityDao + activityMap
    id = applyCurrentNamespace(id, false);

    ResultMap.Builder inlineResultMapBuilder = new ResultMap.Builder(
            configuration,
            id,
            type,
            resultMappings);

    ResultMap resultMap = inlineResultMapBuilder.build();
    configuration.addResultMap(resultMap);
    return resultMap;
}
```

## 第十九章 整合Spring

整合Spring，就需要将ORM框架交给Spring来管理。而我们手写的MyBatis框架将多边的服务都封装在了SqlSession中，而且我们实现了SqlSessionFactory，向外提供SqlSession对象。

于是，我们需要将工厂对象交给IoC容器来管理。Spring提供了FactoryBean接口，实现该接口的类对象会被注入到容器中，因此我们只需实现一个SqlSessionFactoryBean类，实现该接口，同时将工厂对象封装进去，就可以让Spring IoC容器来管理工厂对象。同时，我们实现Spring中的InitializingBean接口，在其中启动MyBatis的流程(加载解析XML文件)。

另外，框架中Dao接口的代理类也需要交给Spring容器来管理，实现依赖注入。我们定义一个MapperScannerConfigurer，将Dao接口全部扫描出来，完成代理并将代理对象全部注入到Spring容器中。即，扫描包路径下的class文件，从中获取对应的class类，然后通过Spring提供的注册接口，将代理类注册到容器中。

接着，我们需要实现MapperFactoryBean类，前面已经将Dao接口的代理类对象注册进了Spring容器，我们就需要在这个类中实现获取该代理对象的逻辑，即通过Dao接口的class对象来直接从SqlSession中获取。


## 第二十章 整合SpringBoot

SpringBoot的核心主要是自动配置类，我们需要为MyBatis实现一个自动配置类。

首先，SpringBoot中我们一般不通过XML文件的方式来配置项目，而是通过yaml文件。于是，我们需要定义一个MyBatisProperties类，负责从yaml文件中读取出定义的数据源信息和mapper路径等。

然后，再定义MyBatisAutoConfiguration类，来实现自动配置。在其中，将MyBatisProperties中的信息封装到一个Document对象中，交给SqlSessionFactoryBuilder来构建工厂对象。并将返回值注册为Bean。

然后将自动配置类的全路径名写到spring.factories中，开启自动配置。