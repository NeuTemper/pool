## 字符串优化
- 主动用StringBuilder拼接替换+	
    ```java
        String str ＝ "hello";
        // 间隔的其他执行语句
        str = str ＋ user.getName();
    ```
    每一条语句都会生成一个新的`StringBuilder`对象
- Log记录主动用StringBuilder拼接替换{}的方式
    ```java
    public final class MessageFormatter {
        public static final FormattingTuple arrayFormat(String messagePattern, Object[] argArray, Throwable throwable) {
            if (messagePattern == null) {
                return new FormattingTuple((String)null, argArray, throwable);
            } else if (argArray == null) {
                return new FormattingTuple(messagePattern);
            } else {
                int i = 0;
                StringBuilder sbuf = new StringBuilder(messagePattern.length() + 50);
    
                for(int L = 0; L < argArray.length; ++L) {
                    int j = messagePattern.indexOf("{}", i);
                    if (j == -1) {
                        if (i == 0) {
                            return new FormattingTuple(messagePattern, argArray, throwable);
                        }
    
                        sbuf.append(messagePattern, i, messagePattern.length());
                        return new FormattingTuple(sbuf.toString(), argArray, throwable);
                    }
    
                    if (isEscapedDelimeter(messagePattern, j)) {
                        if (!isDoubleEscaped(messagePattern, j)) {
                            --L;
                            sbuf.append(messagePattern, i, j - 1);
                            sbuf.append('{');
                            i = j + 1;
                        } else {
                            sbuf.append(messagePattern, i, j - 1);
                            deeplyAppendParameter(sbuf, argArray[L], new HashMap());
                            i = j + 2;
                        }
                    } else {
                        sbuf.append(messagePattern, i, j);
                        deeplyAppendParameter(sbuf, argArray[L], new HashMap());
                        i = j + 2;
                    }
                }
    
                sbuf.append(messagePattern, i, messagePattern.length());
                return new FormattingTuple(sbuf.toString(), argArray, throwable);
            }
        }
    }
    ```
    Log日志使用循环的方式去替换`{}`，可以通过主动使用`StringBuilder`拼接的方式优化掉循环逻辑
- 合理使用字符串常量池intern（注意jdk1.6 字符串常量池容量1009）
    深入解析参考[internal](https://tech.meituan.com/in_depth_understanding_string_intern.html)
## StringBuilder使用优化
- 使用合适的初始长度初始长度
    ```java
    public final class StringBuilder{
        public StringBuilder() {
            super(16);
        }
        
        public StringBuilder(int capacity) {
            super(capacity);
        }
        
        public StringBuilder(String str) {
            super(str.length() + 16);
            append(str);
        }
        
        public StringBuilder(CharSequence seq) {
            this(seq.length() + 16);
            append(seq);
        }
    }
    abstract class AbstractStringBuilder{
        AbstractStringBuilder(int capacity) {
            value = new char[capacity];
        }
        
        public AbstractStringBuilder append(CharSequence s, int start, int end) {
            if (s == null)
                s = "null";
            if ((start < 0) || (start > end) || (end > s.length()))
                throw new IndexOutOfBoundsException(
                    "start " + start + ", end " + end + ", s.length() "
                    + s.length());
            int len = end - start;
            //计算容量
            ensureCapacityInternal(count + len);
            for (int i = start, j = count; i < end; i++, j++)
                value[j] = s.charAt(i);
            count += len;
            return this;
        }
        
        private void ensureCapacityInternal(int minimumCapacity) {
            // overflow-conscious code
            if (minimumCapacity - value.length > 0)
                expandCapacity(minimumCapacity);
        }
        
        void expandCapacity(int minimumCapacity) {
            int newCapacity = value.length * 2 + 2;
            if (newCapacity - minimumCapacity < 0)
                newCapacity = minimumCapacity;
            if (newCapacity < 0) {
                if (minimumCapacity < 0) // overflow
                    throw new OutOfMemoryError();
                newCapacity = Integer.MAX_VALUE;
            }
            value = Arrays.copyOf(value, newCapacity);
        }
    }
    ```
- 每次扩容都会创建新的数组，`toString()`也会创建新的数组，浪费内存空间
    ```java
    public final class StringBuilder{
        @Override
        public String toString() {
            // Create a copy, don't share the array
            return new String(value, 0, count);
        }
    }
    public final class String{
        public String(char value[], int offset, int count) {
            if (offset < 0) {
                throw new StringIndexOutOfBoundsException(offset);
            }
            if (count <= 0) {
                if (count < 0) {
                    throw new StringIndexOutOfBoundsException(count);
                }
                if (offset <= value.length) {
                    this.value = "".value;
                    return;
                }
            }
            // Note: offset or count might be near -1>>>1.
            if (offset > value.length - count) {
                throw new StringIndexOutOfBoundsException(offset + count);
            }
            this.value = Arrays.copyOfRange(value, offset, offset+count);
        }
    }
    ```
- 重用StringBuilder
    ```java
    public class StringBuilderHolder {
    
        private final StringBuilder stringBuilder;
    
        public StringBuilderHolder(int capacity) {
            stringBuilder = new StringBuilder(capacity);
        }
    
        public StringBuilder resetAndGet(){
            //重置count指针，char[]则会继续使用
            stringBuilder.setLength(0);
            return stringBuilder;
        }
    }
    ```
    循环中使用`setLength(0)`代替 `new StringBuilder()`
    ```java
    StringBuilder builder = new StringBuilder();
    for (int i = 0 ; 1 < 10 ; i++) {
        builder.setLength(0);
        builder.append("mumu");  
    }
   ```
- 避免并发冲突
    ```java
    private static final ThreadLocal<StringBuilderHolder> stringBuilderHolder = 
        new ThreadLocal<StringBuilderHolder>(){
            @Override
            protected StringBuilderHolder initialValue() {
                return new StringBuilderHolder(64);
            }
        };
    ```
## 集合类使用优化
- 集合对象初始化指定容量
    ```java
        // 注意map的冲突因子 0.75
        Map<String, Integer> numbers = new HashMap<String, Integer>(6);
        numbers.put("1", 1);
        numbers.put("two", 2);
        numbers.put("three", 3);
        numbers.put("fore", 4);
    ```
   每次使用集合计大小比较麻烦，推荐使用`guava`的集合工具类，以`hashMap`的创建为例，工具类会根据期望大小计算需要初始化的大小，建议多看`guava`源码了解·各种工具类的实现
    ```java
        Map<String, Integer> numbers = Maps.newHashMapWithExpectedSize(4);
    
        public static <K, V> HashMap<K, V> newHashMapWithExpectedSize(int expectedSize) {
            return new HashMap(capacity(expectedSize));
        }
        
        static int capacity(int expectedSize) {
            if (expectedSize < 3) {
                CollectPreconditions.checkNonnegative(expectedSize, "expectedSize");
                return expectedSize + 1;
            } else {
                return expectedSize < 1073741824 ? (int)((float)expectedSize / 0.75F + 1.0F) : 2147483647;
            }
        }    
    ```
- 使用轻量级、高性能集合类，比如`netty`中`LongObjectMap`、`IntObjectMap`

## 减少对象创建与销毁，池化资源
- 线程池(绝不允许使用`new Thread()`的方式创建线程)
    ```java
    public class ThreadPoolExecutorFactory {  
        private static volatile ThreadPoolExecutor threadPoolExecutor;
        // 核心池数量
        private static final int CORE_POOL_SIZE = 10;
        // 最大线程数量
        private static final int MAX_POOL_SIZE = 30;
    
        public static ThreadPoolExecutor getThreadPoolExecutor(){
            if (null == threadPoolExecutor) {
                synchronized (ThreadPoolExecutorFactory.class){
                    if (null == threadPoolExecutor){
                        threadPoolExecutor = new ThreadPoolExecutor(CORE_POOL_SIZE, MAX_POOL_SIZE,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<>());
                    }
                }
            }
            return threadPoolExecutor;
        }
    }

    ```
    使用多线程时需要对线程进行管理，根据业务类型合理选择不同类型的线程池参数，可以实现避免多余线程对象被创建及限制线程数量
- `HTTP`连接池
    ```java
    @Slf4j
    public class HttpUtils {
    
    
        // 请求超时2秒钟
        public static final int CONNECTION_TIMEOUT = 2000;
        // 等待数据超时时间2秒钟
        public static final int SO_TIMEOUT = 2000;
        // 连接不够时等待超时时间
        public static final int CONN_MANAGER_TIMEOUT = 2000;
        // 设置整个连接池最大连接数
        public static final int MAX_TOTAL = 200;
        private static final String HTTP = "http";
        private static final String HTTPS = "https";
    
        static PoolingHttpClientConnectionManager connectionManager;
        static Registry<ConnectionSocketFactory> socketFactoryRegistry;
        static HttpClientBuilder httpClientBuilder;
        static RequestConfig config;
        static SSLContext sslcontext;
    
        static {
            setRequestConfig();
            setSocketFactoryRegistry();
            setConnectionManager();
            setHttpClientBuilder();
        }
    
        public static CloseableHttpClient getCloseableHttpClient() {
            return httpClientBuilder.build();
        }
    
        public static void setRequestConfig() {
            config = RequestConfig.custom().setSocketTimeout(SO_TIMEOUT)
                    .setConnectionRequestTimeout(CONN_MANAGER_TIMEOUT)
                    .setConnectTimeout(CONNECTION_TIMEOUT)
                    .build();
        }
    
        public static void setSocketFactoryRegistry() {
            try {
                sslcontext = SSLContexts.custom().loadTrustMaterial(null, new TrustSelfSignedStrategy()).build();
            } catch (NoSuchAlgorithmException e) {
                handleException(e);
            } catch (KeyManagementException e) {
                handleException(e);
            } catch (KeyStoreException e) {
                handleException(e);
            }
            HostnameVerifier hostnameVerifier = SSLConnectionSocketFactory.ALLOW_ALL_HOSTNAME_VERIFIER;
            SSLConnectionSocketFactory sslConnectionSocketFactory = new SSLConnectionSocketFactory(sslcontext, hostnameVerifier);
            socketFactoryRegistry = RegistryBuilder.<ConnectionSocketFactory>create()
                    .register(HTTP, PlainConnectionSocketFactory.INSTANCE)
                    .register(HTTPS, sslConnectionSocketFactory)
                    .build();
        }
    
        public static void setHttpClientBuilder() {
            httpClientBuilder = HttpClients
                    .custom()
                    .setDefaultRequestConfig(config)
                    .setConnectionManager(connectionManager);
        }
    
        public static void setConnectionManager() {
            connectionManager = new PoolingHttpClientConnectionManager(socketFactoryRegistry);
            connectionManager.setMaxTotal(MAX_TOTAL);
            connectionManager.setDefaultMaxPerRoute(connectionManager.getMaxTotal());
            connectionManager.setValidateAfterInactivity(CONNECTION_TIMEOUT);
        }
    
        private static void handleException(Exception e) {
            // do something
        }
    
    }
    ```

- 常量对象复用，例如BigDecimal.ZERO
- log主动判断日志开启级别减少StringBuilder对象的创建
    ```java
        if (log.isInfoEnabled()) {
            log.error("log");
        }
    ```
## 降级开关
- 不允许从`redis`中直接获取，目前redis解决不了热键问题([热key问题的解决方案](https://github.com/a1525155750/share/blob/master/markdown/cache/cacheModel.md))
- 使用`Hystrix`要评估好合理的参数([hystrix熔断策略](https://github.com/a1525155750/share/blob/master/markdown/distributed/hystrix.md))
## 缓存
合理使用缓存优化接口是性能提升的关键

- 对于QPS要求很高的接口，考虑缓存方法执行结果 
- 需要对回源进行并发控制，当`key`失效，只有单一线程对`key`回源，避免缓存击穿([缓存击穿介绍](https://github.com/a1525155750/share/blob/master/markdown/cache/cache.md))	
- 基于二进制优于文本数据的原则，`JSON`的序列化方案较通用（使用`Jackson` 或 `FastJSON`，`GSON`性能较前两者为差，hessian序列化子父类同名字段存在问题）
    ```java
	public class User implements Serializable {    
	    private String username ;
	}
	public class UserInfo extends User {
	    private String username;
    }   
    public static void main(String[] args) {
        UserInfo user = new UserInfo();
        user.setUsername("mumu");
        ByteArrayOutputStream os = new ByteArrayOutputStream();
        //Hessian的序列化输出
        HessianOutput ho = new HessianOutput(os);
        ho.writeObject(user);
        byte[] userByte = os.toByteArray();
        ByteArrayInputStream is = new ByteArrayInputStream(userByte);
        //Hessian的反序列化读取对象
        HessianInput hi = new HessianInput(is);
        UserInfo u = (UserInfo) hi.readObject();  
    }  
    ```    
    序列化后username是null，如下源码hessian是基于Map进行序列化的，并且子类字段不会覆盖父类字段，所以会存在序列化问题    
    ```java
        protected HashMap<String, FieldDeserializer> getFieldMap(Class<?> cl) {
            HashMap<String, FieldDeserializer> fieldMap = new HashMap<String, FieldDeserializer>();
            for (; cl != null; cl = cl.getSuperclass()) {
                Field[] fields = cl.getDeclaredFields();
                for (int i = 0; i < fields.length; i++) {
                    Field field = fields[i];
                    if (Modifier.isTransient(field.getModifiers()) || Modifier.isStatic(field.getModifiers()))
                        continue;
                    else if (fieldMap.get(field.getName()) != null)
                        continue;
                    try {
                        field.setAccessible(true);
                    } catch (Throwable e) {
                        e.printStackTrace();
                    }
                    Class<?> type = field.getType();
                    FieldDeserializer deser;
                    if (String.class.equals(type)) {
                        deser = new StringFieldDeserializer(field);
                    } else if (byte.class.equals(type)) {
                        deser = new ByteFieldDeserializer(field);
                    }
                    fieldMap.put(field.getName(), deser);
                }
            }
            return fieldMap;
        }
    ```
### 多级缓存
- 根据就近原则，堆内缓存 > 堆外缓存 > 集中式缓存
- 堆外缓存与集中式缓存，需要序列化与反序列化，序列化的性能很重要	
- 集中式缓存，有网络传输成本，如果一次获取多个键时，在有分区的情况下，需要收发多个网络包，`mget`与`mset`要控制数量
### 堆内缓存
- 推荐`Guava Cache`
- 使用合理的并行度、`Capacity`、`expire`参数	
- 重载load()函数，实现单一线程回源([guava的阻塞load](https://github.com/a1525155750/share/blob/master/markdown/cache/guava.md))
### Redis
- 禁止使用大key(不超过256字节)、大`value`（不超过1个网络包（`MTU 1500byte`）），`value`中不存储无用的数据
- 读写分离
- 热键做拆分、本地化处理	
- `mget`与`mset`要控制数量
- 减少交互，多条操作使用`pipeline`
## JVM参数
- 加大Integer、Long Cache 
    ```java
    -XX:AutoBoxCacheMax=20000
    ```
- 分代回收可以设置最大回收年龄大小([分代回收介绍](https://github.com/a1525155750/share/blob/master/markdown/jvm/gc.md))
    ```java
    -XX:MaxTenuringThreshold=3
    ```
## 数据库性能瓶颈优化
往往接口优化的最大问题来自于数据库的瓶颈

- 避免循环调用数据库查询操作，使用批量查询
- 对SQL进行优化
- 使用ES
    
    考虑使用基于数据库binlog把数据同步到EslaticSearch集群中，实现数据库以主键进行业务操作，查询使用ES检索，实现读写分离。
    深入解析参考([ES查询优化](https://github.com/a1525155750/share/blob/master/markdown/elasticsearch/elasticsearch.md))
## 总结
对于TPS/QPS要求很高的接口进行优化，往往很小的细节会导致性能显著提升

	  
