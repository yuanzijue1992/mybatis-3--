[TOC]

本节讲述MyBatis中的DataSource。

### 设计模式

此部分使用了==工厂方法==设计模式，具体的数据源由具体的数据源工厂方法创建。

其中==DataSource==代表了产品接口，**PooledDataSource**和**UnpooledDataSource**代表了两个具体产品类。

==DataSourceFactory==代表了工厂接口，**PooledDataSourceFactory**和**UnpooledDataSourceFactory**代表了两个具体工厂类。

### DataSource

```java
public interface DataSource  extends CommonDataSource, Wrapper {
  // 获取链接
  Connection getConnection() throws SQLException;
  // 获取链接，通过用户名和密码
  Connection getConnection(String username, String password)
    throws SQLException;
}
```

### DataSourceFactory

```java
public interface DataSourceFactory {

  //设置属性,被XMLConfigBuilder所调用
  void setProperties(Properties props);

  //获取相应的DataSource
  DataSource getDataSource();

}
```

MyBatis中的DataSourceFactory具体实现类

![DataSourceFactory](/DataSourceFactoryImps.png)

###1. UnpooleDataSource

#### 1.1 UnpooleDataSource

首先是字段：

```java
  // driver类加载器
  private ClassLoader driverClassLoader;
  // driver属性
  private Properties driverProperties;
  // 存放已经注册了的driver
  private static Map<String, Driver> registeredDrivers = new ConcurrentHashMap<>();

  // 分别是数据库的四个必要参数
  private String driver;
  private String url;
  private String username;
  private String password;

  // 是否自动提交事务
  private Boolean autoCommit;
  // 设置事务隔离级别
  private Integer defaultTransactionIsolationLevel;
```

接着是静态代码块：

```java
// 从DriverManager中获取所有driver，并放入registeredDrivers
static {
    Enumeration<Driver> drivers = DriverManager.getDrivers();
    while (drivers.hasMoreElements()) {
      Driver driver = drivers.nextElement();
      registeredDrivers.put(driver.getClass().getName(), driver);
    }
  }
```

实现==getConnection()==方法 

```java
 @Override
  public Connection getConnection() throws SQLException {
    // 实际调用的是 doGetConnection方法 
    return doGetConnection(username, password);
  }
```

接着看doGetConnection()方法

```java
// 如果没有用户信息，则填充，实际调用的是重载方法  
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
 // 实际的获取connection方法
 private Connection doGetConnection(Properties properties) throws SQLException {
    // 初始化driver 
    initializeDriver();
    // 通过DriverManager获取链接
    Connection connection = DriverManager.getConnection(url, properties);
    // 加工链接，主要是设置自动提交事务，和事务等级
    configureConnection(connection);
    return connection;
  }
```

#### 1.2 UnpooledDataSourceFactory

首先是字段部分：

```java
  private static final String DRIVER_PROPERTY_PREFIX = "driver.";
  private static final int DRIVER_PROPERTY_PREFIX_LENGTH = DRIVER_PROPERTY_PREFIX.length();
  // 相应的datasource
  protected DataSource dataSource;

```

构造函数：

```java
// 初始化工厂时，直接创建一个UnpooledDataSource
public UnpooledDataSourceFactory() {
    this.dataSource = new UnpooledDataSource();
  }
```

实现的方法：

==setProperties==

```java
  // 通过反射的方法来设置datasource
  // 逐个查看properties中的数据，如果是driver类型的参数，则存入driverProperties中，如果是其他类型的参数，则存入相应的字段中。
  @Override
  public void setProperties(Properties properties) {
    Properties driverProperties = new Properties();
    MetaObject metaDataSource = SystemMetaObject.forObject(dataSource);
    for (Object key : properties.keySet()) {
      String propertyName = (String) key;
      if (propertyName.startsWith(DRIVER_PROPERTY_PREFIX)) {
        String value = properties.getProperty(propertyName);
        driverProperties.setProperty(propertyName.substring(DRIVER_PROPERTY_PREFIX_LENGTH), value);
      } else if (metaDataSource.hasSetter(propertyName)) {
        String value = (String) properties.get(propertyName);
        Object convertedValue = convertValue(metaDataSource, propertyName, value);
        metaDataSource.setValue(propertyName, convertedValue);
      } else {
        throw new DataSourceException("Unknown DataSource property: " + propertyName);
      }
    }
    if (driverProperties.size() > 0) {
      metaDataSource.setValue("driverProperties", driverProperties);
    }
  }
```

==getDataSource()==

```java
  直接返回dataSource即可
  @Override
  public DataSource getDataSource() {
    return dataSource;
  }

```

### 2.PooledDataSource

这是一个==线程安全==的，并且==有链接池==的一个实现类。

它并不直接管理链接，而是由==PooledConnection==来管理，首先来看一下这个类。

#### 2.1PooledConnection

首先是类中的字段：

```java
  private static final String CLOSE = "close";
  private static final Class<?>[] IFACES = new Class<?>[] { Connection.class };

  private int hashCode = 0;
  // 使用的datasource
  private PooledDataSource dataSource;
  // 真正的连接
  private Connection realConnection;
  // 代理的连接
  private Connection proxyConnection;
  // 从链接池中取出时的时间
  private long checkoutTimestamp;
  // 该链接创建的时间
  private long createdTimestamp;
  // 最后一次使用的时间戳
  private long lastUsedTimestamp;
  // 由数据库URL、用户名、密码计算出来的hash值，可用于标识该链接池所在的链接池
  private int connectionTypeCode;
  // 是否有效
  private boolean valid;
```

接着是构造函数：

```java
// 使用链接和数据源来创建一个PooledConnection，并初始化其中的字段
public PooledConnection(Connection connection, PooledDataSource dataSource) {
    this.hashCode = connection.hashCode();
    this.realConnection = connection;
    this.dataSource = dataSource;
    this.createdTimestamp = System.currentTimeMillis();
    this.lastUsedTimestamp = System.currentTimeMillis();
    this.valid = true;
    this.proxyConnection = (Connection) Proxy.newProxyInstance(Connection.class.getClassLoader(), IFACES, this);
  }
```

这个类其实是一个代理类，因为我们在关闭链接时，需要将它加入链接池而不是真正的关闭它，所以我们需要对close方法做一些特殊处理：

```java
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    String methodName = method.getName();
    //如果调用close的话，忽略它，将这个connection加入到池中
    if (CLOSE.hashCode() == methodName.hashCode() && CLOSE.equals(methodName)) {
      dataSource.pushConnection(this);
      return null;
    } else {
      try {
        if (!Object.class.equals(method.getDeclaringClass())) {        
        // 检查链接是否是有效的
          checkConnection();
        }
        //其他的方法，则交给真正的connection去调用
        return method.invoke(realConnection, args);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    }
  }
```

#### 2.2PoolState

这个类是用来管理链接池的一个类。

首先看它的字段：

```java
  // 
  protected PooledDataSource dataSource;
  // 空闲链接池
  protected final List<PooledConnection> idleConnections = new ArrayList<>();
  // 活跃链接池
  protected final List<PooledConnection> activeConnections = new ArrayList<>();
  // 请求的次数
  protected long requestCount = 0;
  // 获取链接的累计时间
  protected long accumulatedRequestTime = 0;
  // 所有的链接使用时间总和
  protected long accumulatedCheckoutTime = 0;
  // 超时链接的个数
  protected long claimedOverdueConnectionCount = 0;
  // 累计超时时间
  protected long accumulatedCheckoutTimeOfOverdueConnections = 0;
  // 类继等待时间
  protected long accumulatedWaitTime = 0;
  // 类继等待次数
  protected long hadToWaitCount = 0;
  // 无效的链接数量
  protected long badConnectionCount = 0;
```

#### 2.3 PooledDataSource

````java
  // 链接池
  private final PoolState state = new PoolState(this);
  //实际使用的是UnpooledDataSource
  private final UnpooledDataSource dataSource;
  //正在使用连接的数量
  protected int poolMaximumActiveConnections = 10;
  //空闲连接数
  protected int poolMaximumIdleConnections = 5;
  //在被强制返回之前,池中连接被检查的时间
  protected int poolMaximumCheckoutTime = 20000;
  //在无法获得链接时，最长的等待时间
  protected int poolTimeToWait = 20000;
  //在检测数据库是否可用时，给数据库发送的一个测试SQL语句
  protected String poolPingQuery = "NO PING QUERY SET";
  //开启或禁用侦测查询
  protected boolean poolPingEnabled = false;
  //用来配置 poolPingQuery 多次时间被用一次
  protected int poolPingConnectionsNotUsedFor = 0;
  //url、用户名、密码形成的一个hash值
  private int expectedConnectionTypeCode;
````

获取一个connection时：

```java
private PooledConnection popConnection(String username, String password) throws SQLException {	
    boolean countedWait = false;
    PooledConnection conn = null;
    long t = System.currentTimeMillis();
    int localBadConnectionCount = 0;

    //最外面是while死循环，如果一直拿不到connection，则不断尝试
    while (conn == null) {
      synchronized (state) {
        if (!state.idleConnections.isEmpty()) {
          //如果有空闲的连接的话,返回第一个空闲的链接
          conn = state.idleConnections.remove(0);
          if (log.isDebugEnabled()) {
            log.debug("Checked out connection " + conn.getRealHashCode() + " from pool.");
          }
        } else {//如果没有空闲连接
          if (state.activeConnections.size() < poolMaximumActiveConnections) {
        	// 如果没有达到最大连接数，则创建新的
            conn = new PooledConnection(dataSource.getConnection(), this);
            if (log.isDebugEnabled()) {
              log.debug("Created connection " + conn.getRealHashCode() + ".");
            }
          } else {
        	  //如果activeConnections已经很多了，那不能再new了
        	  //取得activeConnections列表的第一个（最老的）
            PooledConnection oldestActiveConnection = state.activeConnections.get(0);
            long longestCheckoutTime = oldestActiveConnection.getCheckoutTime();
            if (longestCheckoutTime > poolMaximumCheckoutTime) {
            	//如果checkout时间过长，则这个connection标记为overdue（过期）
              state.claimedOverdueConnectionCount++;
              state.accumulatedCheckoutTimeOfOverdueConnections += longestCheckoutTime;
              state.accumulatedCheckoutTime += longestCheckoutTime;
              state.activeConnections.remove(oldestActiveConnection);
              if (!oldestActiveConnection.getRealConnection().getAutoCommit()) {
                oldestActiveConnection.getRealConnection().rollback();
              }
              //删掉最老的连接，然后再new一个新连接
              conn = new PooledConnection(oldestActiveConnection.getRealConnection(), this);
              oldestActiveConnection.invalidate();
              if (log.isDebugEnabled()) {
                log.debug("Claimed overdue connection " + conn.getRealHashCode() + ".");
              }
            } else {
            	//无空闲链接，无法创建新的链接且无超时时间，只能阻塞
              try {
                if (!countedWait) {
                	//统计信息：等待+1
                  state.hadToWaitCount++;
                  countedWait = true;
                }
                if (log.isDebugEnabled()) {
                  log.debug("Waiting as long as " + poolTimeToWait + " milliseconds for connection.");
                }
                long wt = System.currentTimeMillis();
                state.wait(poolTimeToWait); //阻塞
                state.accumulatedWaitTime += System.currentTimeMillis() - wt;
              } catch (InterruptedException e) {
                break;
              }
            }
          }
        }
        if (conn != null) {
        	//如果已经拿到connection，则返回
          if (conn.isValid()) {
            if (!conn.getRealConnection().getAutoCommit()) {
              conn.getRealConnection().rollback();
            }
            conn.setConnectionTypeCode(assembleConnectionTypeCode(dataSource.getUrl(), username, password));
            //记录checkout时间
            conn.setCheckoutTimestamp(System.currentTimeMillis());
            conn.setLastUsedTimestamp(System.currentTimeMillis());
            state.activeConnections.add(conn);
            state.requestCount++;
            state.accumulatedRequestTime += System.currentTimeMillis() - t;
          } else {
            if (log.isDebugEnabled()) {
              log.debug("A bad connection (" + conn.getRealHashCode() + ") was returned from the pool, getting another connection.");
            }
            //如果没拿到，统计信息：坏连接+1
            state.badConnectionCount++;
            localBadConnectionCount++;
            conn = null;
            if (localBadConnectionCount > (poolMaximumIdleConnections + 3)) {
            	//如果好几次都拿不到，就放弃了，抛出异常
              if (log.isDebugEnabled()) {
                log.debug("PooledDataSource: Could not get a good connection to the database.");
              }
              throw new SQLException("PooledDataSource: Could not get a good connection to the database.");
            }
          }
        }
      }

    }
```

归还链接：

```java
protected void pushConnection(PooledConnection conn) throws SQLException {
    synchronized (state) {
      //先从activeConnections中删除此connection
      state.activeConnections.remove(conn);
      if (conn.isValid()) {
        if (state.idleConnections.size() < poolMaximumIdleConnections && conn.getConnectionTypeCode() == expectedConnectionTypeCode) {
      	  //如果空闲的连接太少，
          state.accumulatedCheckoutTime += conn.getCheckoutTime();
          if (!conn.getRealConnection().getAutoCommit()) {
            conn.getRealConnection().rollback();
          }
          //new一个新的Connection，加入到idle列表
          PooledConnection newConn = new PooledConnection(conn.getRealConnection(), this);
          state.idleConnections.add(newConn);
          newConn.setCreatedTimestamp(conn.getCreatedTimestamp());
          newConn.setLastUsedTimestamp(conn.getLastUsedTimestamp());
          conn.invalidate();
          if (log.isDebugEnabled()) {
            log.debug("Returned connection " + newConn.getRealHashCode() + " to pool.");
          }
          //通知其他线程可以来抢connection了
          state.notifyAll();
        } else {
        	//否则，即空闲的连接已经足够了
          state.accumulatedCheckoutTime += conn.getCheckoutTime();
          if (!conn.getRealConnection().getAutoCommit()) {
            conn.getRealConnection().rollback();
          }
          //那就将connection关闭就可以了
          conn.getRealConnection().close();
          if (log.isDebugEnabled()) {
            log.debug("Closed connection " + conn.getRealHashCode() + ".");
          }
          conn.invalidate();
        }
      } else {
        if (log.isDebugEnabled()) {
          log.debug("A bad connection (" + conn.getRealHashCode() + ") attempted to return to the pool, discarding connection.");
        }
        state.badConnectionCount++;
      }
    }
  }
```

#### 2.4 PooledDataSourceFactory

这个很简单，只是将数据源换成了PooledDataSource

```java
public class PooledDataSourceFactory extends UnpooledDataSourceFactory {
  //数据源换成了PooledDataSource
  public PooledDataSourceFactory() {
    this.dataSource = new PooledDataSource();
  }
}
```



