# MoroBatis

手动实现Mybatis框架，摸清“为什么在使用Mybatis的时候，只需定义一个Dao接口，不用写实现类就能使用XML中或者注解上配置好的SQL语句，就能完成对数据库CRUD的操作呢？”这样的问题。

Mybatis的作用：

不使用Mybatis之前，需要通过JDBC来手动连接数据库，Java代码和SQL代码耦合在一起，不利于后期修改维护。同时Mybatis免除了JDBC参数设置和获取结果集的工作。


## 第一章 创建简单的映射器代理工厂

首先，为什么Mybatis框架中，能够只定义一个接口，其内部只进行方法声明而不实现方法，就可以通过XML文件来执行SQL语句，与数据库进行操作呢。

Mybatis就是通过反射和代理的方式，为我们定的接口实现了一个代理类Mapper，代理类来实现该接口。从而，可以通过Mapper对象来调用接口中定义的方法，去实现数据库的CRUD操作。

### MapperProxy

于是，我们先创建一个代理类`MapperProxy`，通过反射(`java.lang.reflect`)的方式，在`invoke()`方法中，实现Dao接口中抽象方法的执行逻辑。

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

然后，再使用工厂模式，创建一个`MapperProxyFactory`代理工厂，来为我们生成代理类对象。生成代理对象的做法为，通过`java.lang.reflect`包中提供的API，去生成指定Dao接口的代理对象并返回。

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

代理工厂给我们返回的代理对象就是Dao接口的实现类对象，即Mapper对象，我们可以直接通过Mapper对象调用Dao接口中的方法。


## 第二章 实现映射器的注册和使用

在第一章的代码中，工厂创建代理对象时，需要传入一个Map集合(在代码中参数名为`SqlSession`)，这个Ma集合是用来告知工厂需要对哪个接口进行代理，这里存在硬编码问题。同时，虽然名为`SqlSession`但却用的是Map，并不是真正的SqlSession对象。

参考SpringMVC的做法，在项目启动时，用户只需要指定路径即可完成对需要代理的Dao的扫描和注册。于是，我们需要对映射器的注册提供注册器来处理，同时还需要对SqlSession进行规范化处理，让它可以将我们的映射器代理和方法调用进行包装，建立一个生命周期模型，方便后期扩展。

经上述分析，我们需要一个注册器类，它可以扫描包路径完成Dao接口与代理类的映射。以及一个SqlSession接口的定义。

### MapperRegistry注册器类

注册器类，其功能顾名思义就是将Dao接口与代理工厂一一对应起来，相当于完成了注册。则其应该实现如下方法：
- `getMapper`：根据Dao接口，返回对应的代理类实例。
- `addMapper`：存储Dao接口与其代理工厂的映射关系，完成注册。
- `hasMapper`：根据Dao接口，查询是否已经注册了与代理工厂的映射关系。

于是，可以通过一个Map集合来存储映射关系。

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

目前，先拿最简单的`select`语句作为基础功能。SqlSession应该提供如下功能：
- `getMapper`：根据传入的Dao接口类型，返回对应的代理类实例。
- `selectOne`：传入参数为`String statement`，SQL语句的ID，方法应该根据该ID，执行对应的SQL，返回数据库中的一条记录。

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

对于SqlSession，既然要能够返回Dao接口的代理对象，那么就应该在其内部包含一个注册器类`MapperRegistry`。

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

Mybatis的核心逻辑就是：提供一个Dao接口的代理类Mapper，Mapper中需要对XML文件进行解析和处理。

在第二章，我们已经通过MapperRegistry类来扫描包路径对Mapper进行注册。如果，我们将命名空间、SQL描述、映射信息统一维护到一个XML文件中，那么我们解析XML文件即可Mapper的注册和SQL管理。

所以，本章的重点就是如何解析XML文件。在本章节只对`<mappers>`标签的内容解析，获取其中的SQL语句。

### SqlSessionFactoryBuilder

首先，需要定义一个`SqlSessionFactoryBuilder`工厂建造者模式类，作为整个MoroBatis的入口。通过入口I/O的方式，对XML文件进行解析。在目前阶段，主要解析SQL语句，并注册Mapper，串联出整个核心流程的脉络。

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

在代码中，使用I/O创建了一个`XMLConfigBuilder`对象，其内部会对XML文件进行解析，将命名空间、SQL描述、资源等等信息解析出来，放置在`Configuration`对象中。然后调用`XMLConfigBuilder.parse()`方法返回这个Configuration对象。

SqlSessionFactoryBuilder再利用获取的Configuration对象来创建SqlSessionFactory工厂对象并返回。

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

Configuration类是一个配置类，其对MapperRegistry的功能再做了一次封装。配置类将XMLConfigBuilder解析出来的XML文件的信息存储在内部。

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

有了事务的定义，我们给出一个实现。

可以看出，如果是通过数据源创建的事务，获取连接时是通过dataSource来实现的。即从数据库连接池中拿到一个连接返回。

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

该类型的每一个实例记录一个数据库类型和Java类型的对应关系。

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

### 执行器的创见与使用

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