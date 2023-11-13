# SpringBoot + Redis + Redission + HuTool 实现布隆过滤器功能

### 一、布隆过滤器简介


#### 1.1 什么是布隆过滤器
布隆过滤器（Bloom Filter）是1970年由布隆提出的。它实际上是一个很长的二进制向量和一系列随机映射函数。布隆过滤器可以用于检索一个元素是否在一个集合中。它的优点是空间效率和查询时间都比一般的算法要好的多，缺点是有一定的误识别率和删除困难。


上面这句介绍比较全面的描述了什么是布隆过滤器，如果还是不太好理解的话，就可以把布隆过滤器理解为一个set集合，我们可以通过add往里面添加元素，通过contains来判断是否包含某个元素。由于本文讲述布隆过滤器时会结合Redis来讲解，因此类比为Redis中的Set数据结构会比较好理解，而且Redis中的布隆过滤器使用的指令与Set集合非常类似（后续会讲到）。


学习布隆过滤器之前有必要先聊下它的优缺点，因为好的东西我们才想要嘛！
##### 布隆过滤器的优点：

时间复杂度低，增加和查询元素的时间复杂为O(N)，（N为哈希函数的个数，通常情况比较小）
保密性强，布隆过滤器不存储元素本身
存储空间小，如果允许存在一定的误判，布隆过滤器是非常节省空间的（相比其他数据结构如Set集合）
##### 布隆过滤器的缺点：

有点一定的误判率，但是可以通过调整参数来降低
无法获取元素本身
很难删除元素
#### 1.2 布隆过滤器的使用场景
布隆过滤器可以告诉我们 “某样东西一定不存在或者可能存在”，也就是说布隆过滤器说这个数不存在则一定不存，布隆过滤器说这个数存在可能不存在（误判，后续会讲），**利用这个判断是否存在的特点可以做很多有趣的事情。

解决Redis缓存穿透问题（面试重点）
邮件过滤，使用布隆过滤器来做邮件黑名单过滤
对爬虫网址进行过滤，爬过的不再爬
解决新闻推荐过的不再推荐(类似抖音刷过的往下滑动不再刷到)
HBase\RocksDB\LevelDB等数据库内置布隆过滤器，用于判断数据是否存在，可以减少数据库的IO请求

#### 1.3 布隆过滤器的原理
##### 1.3.1 数据结构
布隆过滤器它实际上是一个很长的二进制向量和一系列随机映射函数。以Redis中的布隆过滤器实现为例，Redis中的布隆过滤器底层是一个大型位数组（二进制数组）+多个无偏hash函数。
一个大型位数组（二进制数组）：


多个无偏hash函数：
无偏hash函数就是能把元素的hash值计算的比较均匀的hash函数，能使得计算后的元素下标比较均匀的映射到位数组中。

如下就是一个简单的布隆过滤器示意图，其中k1、k2代表增加的元素，a、b、c即为无偏hash函数，最下层则为二进制数组。


##### 1.3.2 空间计算
在布隆过滤器增加元素之前，首先需要初始化布隆过滤器的空间，也就是上面说的二进制数组，除此之外还需要计算无偏hash函数的个数。布隆过滤器提供了两个参数，分别是预计加入元素的大小n，运行的错误率f。布隆过滤器中有算法根据这两个参数会计算出二进制数组的大小l，以及无偏hash函数的个数k。
它们之间的关系比较简单：

错误率越低，位数组越长，控件占用较大
错误率越低，无偏hash函数越多，计算耗时较长


### 二、布隆过滤器的实现
#### 2.1 引入所需jar包
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.15.4</version>
</dependency>
<dependency>
    <groupId>cn.hutool</groupId>
    <artifactId>hutool-core</artifactId>
    <version>5.8.12</version>
</dependency>

<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```
#### 2.2 配置redisson RedissonConfig
```java
@Configuration
public class RedissonConfig {
    @Value("${spring.redis.host}")
    private String host;
    @Value("${spring.redis.port}")
    private String port;
    @Value("${spring.redis.password:}")
    private String password;
    @Value("${spring.redis.database}")
    private Integer database;
    @Value("${spring.redis.cluster.nodes:}")
    private String clusterNodes;

    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        if (!CharSequenceUtil.isEmpty(clusterNodes)) {
            //集群cluster
            String[] nodeAddr = Stream.of(clusterNodes.split(",")) .map(url -> String.format("redis://%s", url)).toArray(String[]::new);
            config.useClusterServers().setScanInterval(2000).addNodeAddress(nodeAddr);
            config.useClusterServers().setPassword(CharSequenceUtil.emptyToDefault(password, null));
        } else {
            //单节点
            config.useSingleServer().setAddress("redis://" + host + ":" + port);
            config.useSingleServer().setDatabase(database);
            config.useSingleServer().setPassword(CharSequenceUtil.emptyToDefault(password, null));
        }
        return Redisson.create(config);
    }
}
```

#### 2.3 配置布隆过滤器 BloomConfig
```java
@Configuration
@ConfigurationProperties(prefix = "bloom")
@Getter
@Setter
public class BloomConfig {
    /**
     * 是否开启布隆过滤器
     */
    private Boolean enable;
    /**
     * 布隆过滤器误判率
     */
    private Double missRate;
    /**
     * 布隆过滤器大小
     */
    private Long size;
    /**
     * 布隆过滤器过期时间
     */
    private Long expireTime;
    /**
     * 布隆过滤器复位时间(以天为单位，且需大于expireTime)
     */
    private Long reset;
}
```


#### 2.4 实现布隆过滤器
```java
@Component
@RequiredArgsConstructor
public class BloomFilter implements DisposableBean {

    private final RedissonClient redissonClient;

    private volatile long curTimeStamp;

    private RBloomFilter<String> cur;

    private RBloomFilter<String> feature;

    private static String PRI = "%s:bloom:%s";

    private static final String KEY = "keys:bloom:";


    @Resource
    private BloomConfig bloomConfig;


    public void init(String prefix) {

        long timeMillis = System.currentTimeMillis();
        if (cur == null) {
            RMapCache<Object, Object> mapCache = redissonClient.getMapCache(KEY);
            // 当前过滤器
            String random = RandomUtil.randomString(8);
            String key = String.format(PRI, prefix, random);

            mapCache.put(key, timeMillis);
            cur = redissonClient.getBloomFilter(key);
            // 使用最多允许 size 个元素的布隆过滤器
            cur.tryInit(bloomConfig.getSize(), bloomConfig.getMissRate());
            curTimeStamp = timeMillis;
        }
        if (timeMillis - curTimeStamp >= bloomConfig.getExpireTime() && feature == null) {
            RMapCache<Object, Object> mapCache = redissonClient.getMapCache(KEY);
            // T + 2 过滤器
            String randomNext = RandomUtil.randomString(8);
            String nextKey = String.format(PRI, prefix, randomNext);

            mapCache.put(nextKey, timeMillis);
            feature = redissonClient.getBloomFilter(nextKey);
            // 使用最多允许 size 个元素的布隆过滤器
            feature.tryInit(bloomConfig.getSize(), bloomConfig.getMissRate());
            clearBitMap(mapCache, timeMillis);
        }

    }

    private void clearBitMap(RMapCache<Object, Object> mapCache, long timeMillis) {
        final int three = 3;
        final int second = 2;
        Set<Object> keySet = mapCache.keySet();
        if (keySet.size() <= second) {
            return;
        }
        for (Object o : keySet) {
            String key = (String) o;
            if (timeMillis - (Long) mapCache.get(key) > three * bloomConfig.getExpireTime()) {
                RBloomFilter<String> delete = redissonClient.getBloomFilter(key);
                delete.deleteAsync();
                mapCache.removeAsync(key);
            }
        }
    }


    public boolean filter(String prefix, String value) {
        init(prefix);
        final int multiple = 2;
        if (System.currentTimeMillis() - curTimeStamp < bloomConfig.getExpireTime()) {
            boolean contains = cur.contains(value);
            if (!contains) {
                cur.add(value);
            }
            return contains;
        } else if (System.currentTimeMillis() - curTimeStamp < multiple * bloomConfig.getExpireTime()) {
            boolean contains = cur.contains(value);
            if (!contains) {
                cur.add(value);
                feature.add(value);
            }
            return contains;
        } else {
            curTimeStamp = System.currentTimeMillis();
            cur.deleteAsync();
            cur = feature;
            feature = null;
            boolean contains = cur.contains(value);
            if (!contains) {
                cur.add(value);
            }
            return contains;
        }

    }

    @Override
    public void destroy() throws Exception {
        if (cur != null) {
            cur.deleteAsync();
        }
        if (feature != null) {
            feature.deleteAsync();
        }
    }
}
```

#### 2.5 配置 yaml 参数
```yaml
spring:
    redis:
        host: 10.10.2.217
        # 端口，默认为6379
        port: 32426
        password: Parav1ew
        # 连接超时时间
        database: 0
        timeout: 5000
    jedis:
        pool:
            # 连接池中的最小空闲连接
            min-idle: 5
            # 连接池中的最大空闲连接
            max-idle: 10
            # 连接池的最大数据库连接数
            max-active: 20
            # #连接池最大阻塞等待时间（使用负值表示没有限制）
            max-wait: 2000
            testOnCreate: true
            testOnBorrow: true
            testOnReturn: true
            testWhileIdle: true
    lettuce:
        cluster:
            refresh:
                adaptive: true
            period: 60000
        pool:
            # 连接池中的最小空闲连接
            min-idle: 5
            # 连接池中的最大空闲连接
            max-idle: 10
            # 连接池的最大数据库连接数
            max-active: 20
            # #连接池最大阻塞等待时间（使用负值表示没有限制）
            max-wait: 2000
            testOnCreate: true
            testOnBorrow: true
            testOnReturn: true
            testWhileIdle: true

bloom:
    enable: true
    missRate: 0.03
    size: 10
    expireTime: 60000
    reset: 180
```


#### 2.6 引入BloomController
```java
@RestController
@Slf4j
@RequiredArgsConstructor
@RequestMapping("/bloom")
public class BloomFilterController {

    private final BloomFilter bloomFilter;

    private final BloomConfig bloomConfig;

    @GetMapping("/testBloom")
    public Boolean existBloom(String id) {
        String reqId = UUID.randomUUID().toString();
        return checkBloomFilter(id, reqId);
    }

    /**
     * 校验布隆过滤器
     */
    private Boolean checkBloomFilter(String id, String reqId) {
        // 布隆过滤器是否开启
        if (!bloomConfig.getEnable()) {
            log.info("布隆过滤器未开启");
            return false;
        }

        final String prefix = "filter";
        boolean flag = bloomFilter.filter(prefix, ObjectUtil.defaultIfBlank(id , reqId));
        if (flag) {
            log.info("布隆过滤器判定数据可能已经存在，不进行后续操作");
            return true;
        }
        return false;
    }
}

```
