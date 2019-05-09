---
title: Spring Boot 2实践系列(十二)：Spring Data Redis 集成详解和使用
comments: true
date: 2018-06-05 11:54:46
tags: [redis,jedis,redisTemplate,stringRedisTemplate]
categories: [Spring Boot 2实践系列]
---
　　Redis 是基于**key-value**键值对开源内存数据存储系统，现在非常流行用作缓存存储。

　　**Spring Boot** 集成 Redis 非常简单，也容易使用。Spring Boot 自动注册了`RedisConnectionFactory`,并提供了`RedisTemplate`和`StringRedisTemplate`两个模板来操作数据。所以在Spring Boot环境，只需配置下 **Redis** 的连接参数就可以直接使用了。
<!-- more -->
## Spring Data Redis ##
Spring Boot 对 Redis 的支持依赖于 [**Sping Data Redis**](https://docs.spring.io/spring-data/data-redis/docs/current/reference/html/)。Spring Data Redis 将数据操作抽象出了统一的方法便于使用。

Spring Data Redis提供了两种方式来连接 `Redis`，分别是`LettuceConnectionFactory`和`RedisConnectionFactory`。**Lettuce**是基于`Netty`的开源连接器,性能更高,在多线程并发情况下,连接是线程安全的; 而 Jedis 在多线程并发下不是线程安全的，就需要使用线程池来给每个线程创建物理连接。

在Spring Boot项目里不需要人为配置这两个工厂 **Bean**,默认集成的是**Lettuce**依赖，使用的是**LettuceConnectionFactory**创建连接。
### LettuceConnectionFactory ###
``` java
@Configuration
class AppConfig {

  @Bean
  public LettuceConnectionFactory redisConnectionFactory() {

    return new LettuceConnectionFactory(new RedisStandaloneConfiguration("server", 6379));
  }
}
```
### RedisConnectionFactory ###
``` java
@Configuration
class AppConfig {

  @Bean
  public RedisConnectionFactory redisConnectionFactory(){
        return new JedisConnectionFactory();
    }
}
```
### RedisTemplate ###
Spring Data Redis 对操作提供了高层的抽象，可以自定义序列化和连接管理，提供了丰富的操作接口,下面列有是常用的操作接口：

|Interface          |Description         |
|:------------------|:-------------------|
|HashOperations|Redis hash operations|
|ListOperations|Redis list operations|
|SetOperations|Redis set operations|
|ValueOperations|Redis string (or value) operations|
|ZSetOperations|Redis zset (or sorted set) operations|
该模板是线程安全的，可在并发多线程重复使用。
``` java
public class Example {

  // inject the actual template
  @Autowired
  private RedisTemplate<String, String> template;

  // inject the template as ListOperations
  @Resource(name="redisTemplate")
  private ListOperations<String, String> listOps;

  public void addLink(String userId, URL url) {
    listOps.leftPush(userId, url.toExternalForm());
  }
}
```
RedisTemplate 默认的的序列化方式是采用 JDK的二进制来序列化，键值用户不可读。

### StringRedisTemplate  ###
由于存储在 `Redis` 中的**键**和**值**通常是 String 类型，**Redis**模块为 `RedisConnection` 和 `RedisTemplate` 提供了两个扩展，分别为 `StringRedisConnection`（及其DefaultStringRedisConnection实现）和 `StringRedisTemplate`。为大量的字符串操作提供了便 捷的操作。 除了绑定到 **String** 键之外，模板和连接使用 `StringRedisSerializer`来序列化，这样存储的键和值是用户是可读的（前提是存在 Redis 和代码中都使用相同的编码）。
``` java
public class Example {

  @Autowired
  private StringRedisTemplate redisTemplate;

  public void addLink(String userId, URL url) {
    redisTemplate.opsForList().leftPush(userId, url.toExternalForm());
  }
}
```

### redis resporites ###
Spring-data-redis 提供了 Redis Repositories 来支持 Redis 缓存操作，可以通过继承 CrudRepository 或 JpaRepository 接口调用已提供的方法来操作传统数据库，再结合缓存注解来对数据进行缓存到 redis 的操作。
1. 在java 配置类上添加注解(@EnableRedisRepositories)开启Redis Repositories 的支持
	``` java
	@Configuration
	@EnableRedisRepositories
	public class ApplicationConfig {
	
	  @Bean
	  public RedisConnectionFactory connectionFactory() {
	    return new JedisConnectionFactory();
	  }
	
	  @Bean
	  public RedisTemplate<?, ?> redisTemplate() {
	
	    RedisTemplate<?, ?> template = new RedisTemplate<>();
	    return template;
	  }
	}
	```
2. 实体类的 Repository 接口继承 CrudRepository 或 JpaRepository
	``` java
	@Repository
	public interface UserRepository extends CrudRepository<User, Long> {
	
	}
	```
3. 业务层注入实体类的 Repository，调用 Repository 的方法来对数据库进行操作,在方法上添加缓存注解来对数据进行缓存操作
	``` java
	@Service
	public class UserServiceImpl implements UserService {
	
	    //注入实体类的Repository
	    @Autowired
	    private UserRepository userRepository;
	
	    /**
	     * 使用Cache注解
	     * 先查缓存,没有再查库,再写入缓存
	     */
	    @Override
	    @Cacheable(key = "#userId",value = "user")
	    public User queryByUserId(Long userId) {
	        User user = userRepository.findById(userId).get();
	        return user;
	    }
	}
	```
4. Spring Boot 的自动配置里默认就开启了对  Redis Repositories 的支持
在配置文件里可通过以下属性来设置是否开启
> spring.data.redis.repositories.enabled=true

**注(个人理解):** 
`RedisTemplate` 相当于 Redis 的客户端，是把 Redis 作为纯数据库来操作，这个数据库是在计算机物理内存中，自身具有缓存的特性；
而 `Redis Repositories` 结合缓存注解来使用，是把 Redis 作为 传统数据库的缓存来操作，相当于传统数据库上加了缓存层；这两者存储的数据在 Redis 中的表现也不同, Redis Repositories存入 Redis 的数据是有与实体类映射关系的,有对象的概念，每条数据相当于一个实例。而 RedisTemplate 存入 Redis 的数据就单纯是条数据。


## Spring Boot Redis ##
1. 添加依赖 **spring-boot-starter-data-redis**
	``` xml
	<!-- redis -->
	<dependency>
	    <groupId>org.springframework.boot</groupId>
	    <artifactId>spring-boot-starter-data-redis</artifactId>
	</dependency>
	```
	若需要使用 Jedis 替换 Lettuce,则使用下面依赖
	``` xml
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-data-redis</artifactId>
		<exclusions>
			<exclusion>
				<groupId>io.lettuce</groupId>
				<artifactId>lettuce-core</artifactId>
			</exclusion>
		</exclusions>
	</dependency>
	<dependency>
		<groupId>redis.clients</groupId>
		<artifactId>jedis</artifactId>
	</dependency>
	```
	若使用默认的 Lettuce ,配置连接池需要依赖 apache commons-pool2
	```
	<dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-pool2</artifactId>
        <version>2.6.0</version>
    </dependency>
	```
2. redis参数配置
	``` 
	spring.redis.database=0
	spring.redis.host=192.168.220.128
	spring.redis.password=123456
	spring.redis.port=6379
	spring.redis.jedis.pool.max-idle=8
	spring.redis.jedis.pool.min-idle=0
	spring.redis.jedis.pool.max-active=8
	spring.redis.jedis.pool.max-wait=-1ms
	```
3. 自定义序列化
	**添加依赖**
	``` xml
	<!-- fastjson -->
	<dependency>
	    <groupId>com.alibaba</groupId>
	    <artifactId>fastjson</artifactId>
	    <version>1.2.47</version>
	</dependency>
	<!-- jackson serializer msgpack -->
	<dependency>
	    <groupId>org.msgpack</groupId>
	    <artifactId>jackson-dataformat-msgpack</artifactId>
	    <version>0.8.16</version>
	</dependency>
	<!--kryo-->
	<dependency>
	    <groupId>com.esotericsoftware</groupId>
	    <artifactId>kryo</artifactId>
	    <version>4.0.2</version>
	</dependency>
	```
	**自定义Serializer：StringRedisSerializer**
	``` java
	/**
	 * @name: StringRedisSerializer
	 * @desc: 重写StringRedisSerializer,支持对象数据序列化,默认只支持String数据
	 * @author: gxing
	 * @date: 2018-11-05 16:15
	 **/
	public class StringRedisSerializer implements RedisSerializer<Object> {
	
	    private final Charset charset;
	
	    private final String target = "\"";
	
	    private final String replacement = "";
	
	    public StringRedisSerializer() {
	        this(Charset.forName("UTF8"));
	    }
	
	    public StringRedisSerializer(Charset charset) {
	        Assert.notNull(charset, "Charset must not be null!");
	        this.charset = charset;
	    }
	
	    @Override
	    public String deserialize(byte[] bytes) {
	        return (bytes == null ? null : new String(bytes, charset));
	    }
	
	    @Override
	    public byte[] serialize(Object object) {
	        String string = JSON.toJSONString(object);
	        if (string == null) {
	            return null;
	        }
	        string = string.replace(target, replacement);
	        return string.getBytes(charset);
	    }
	}
	```
	**自定义Serializer：MsgpackRedisSerializer**
	``` java
	/**
	 * @name: MsgpackRedisSerializer
	 * @desc: TODO
	 * @author: gxing
	 * @date: 2018-11-05 14:57
	 **/
	public class MsgpackRedisSerializer<T> implements RedisSerializer<Object> {
	
	    static final byte[] EMPTY_ARRAY = new byte[0];
	    private final ObjectMapper mapper;
	
	
	    public MsgpackRedisSerializer() {
	        this.mapper = new ObjectMapper(new MessagePackFactory());
	        this.mapper.registerModule((new SimpleModule()).addSerializer(new NullValueSerializer(null)));
	        this.mapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL.NON_FINAL, JsonTypeInfo.As.PROPERTY.PROPERTY);
	    }
	
	    @Override
	    public byte[] serialize(@Nullable Object source) throws SerializationException {
	        if (source == null) {
	            return EMPTY_ARRAY;
	        } else {
	            try {
	                return this.mapper.writeValueAsBytes(source);
	            } catch (JsonProcessingException var3) {
	                throw new SerializationException("Could not write JSON: " + var3.getMessage(), var3);
	            }
	        }
	    }
	
	    @Override
	    public Object deserialize(@Nullable byte[] source) throws SerializationException {
	        return this.deserialize(source, Object.class);
	    }
	
	    @Nullable
	    public <T> T deserialize(@Nullable byte[] source, Class<T> type) throws SerializationException {
	        Assert.notNull(type, "Deserialization type must not be null! Pleaes provide Object.class to make use of Jackson2 default typing.");
	        if (source == null || source.length == 0) {
	            return null;
	        } else {
	            try {
	                return this.mapper.readValue(source, type);
	            } catch (Exception var4) {
	                throw new SerializationException("Could not read JSON: " + var4.getMessage(), var4);
	            }
	        }
	    }
	
	    private class NullValueSerializer extends StdSerializer<NullValue> {
	        private static final long serialVersionUID = 2199052150128658111L;
	        private final String classIdentifier;
	
	        NullValueSerializer(@Nullable String classIdentifier) {
	            super(NullValue.class);
	            this.classIdentifier = StringUtils.hasText(classIdentifier) ? classIdentifier : "@class";
	        }
	
	        @Override
	        public void serialize(NullValue value, JsonGenerator jgen, SerializerProvider provider) throws IOException {
	            jgen.writeStartObject();
	            jgen.writeStringField(this.classIdentifier, NullValue.class.getName());
	            jgen.writeEndObject();
	        }
	    }
	}
	```
	**自定义Serializer：KryoRedisSerializer**
	``` java
	/**
	 * @name: KryoRedisSerializer
	 * @desc: Kryo序列化和反序列化
	 * @author: gxing
	 * @date: 2018-11-05 17:08
	 **/
	public class KryoRedisSerializer<T> implements RedisSerializer<T> {
	    Logger logger = LoggerFactory.getLogger(KryoRedisSerializer.class);
	
	    public static final byte[] EMPTY_BYTE_ARRAY = new byte[0];
	
	    private static final ThreadLocal<Kryo> kryos = ThreadLocal.withInitial(Kryo::new);
	
	    private Class<T> clazz;
	
	    public KryoRedisSerializer(Class<T> clazz) {
	        super();
	        this.clazz = clazz;
	    }
	
	    @Override
	    public byte[] serialize(T t) throws SerializationException {
	        if (t == null) {
	            return EMPTY_BYTE_ARRAY;
	        }
	
	        Kryo kryo = kryos.get();
	        kryo.setReferences(false);
	        kryo.register(clazz);
	
	        try (ByteArrayOutputStream baos = new ByteArrayOutputStream();
	             Output output = new Output(baos)) {
	            kryo.writeClassAndObject(output, t);
	            output.flush();
	            return baos.toByteArray();
	        } catch (Exception e) {
	            logger.error(e.getMessage(), e);
	        }
	
	        return EMPTY_BYTE_ARRAY;
	    }
	
	    @Override
	    public T deserialize(byte[] bytes) throws SerializationException {
	        if (bytes == null || bytes.length <= 0) {
	            return null;
	        }
	
	        Kryo kryo = kryos.get();
	        kryo.setReferences(false);
	        kryo.register(clazz);
	
	        try (Input input = new Input(bytes)) {
	            return (T) kryo.readClassAndObject(input);
	        } catch (Exception e) {
	            logger.error(e.getMessage(), e);
	        }
	
	        return null;
	    }
	}
	```
4. 序列化配置
``` java
@Configuration
@EnableRedisRepositories//Spring Boot 默认已开启对Redis Repository的支持，此注解可省略
public class RedisConfig {

    /**
     * SpringBoot已自动注册了这两个Bean
     */
    /*@Bean
    public RedisConnectionFactory redisConnectionFactory(){
        return new JedisConnectionFactory();
    }

    @Bean
    public RedisTemplate<Object, Object> redisTemplate(){
        RedisTemplate<Object, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        return template;
    }*/

    /**
     * redis默认使用jdk的二进制数据来序列化
     * 以下自定义使用jackson来序列化
     *
     * @param redisConnectionFactory
     * @return
     * @throws UnknownHostException
     */
    @Bean
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<Object, Object> template = new RedisTemplate<Object, Object>();
        template.setConnectionFactory(redisConnectionFactory);

        /* 序列化10000个对象数据,在Redis 所占用空间
         * 根据最终测试, String 和 FastJson 占用较少
         * */
//        Jackson2JsonRedisSerializer serializer = new Jackson2JsonRedisSerializer(Object.class);//2.5M,若开启类型检测是2.96M
//        StringRedisSerializer serializer = new StringRedisSerializer();//2.33M
//        FastJsonRedisSerializer serializer = new FastJsonRedisSerializer(Object.class);//2.35M
//        KryoRedisSerializer serializer = new KryoRedisSerializer(Object.class);//2.35M
        MsgpackRedisSerializer serializer = new MsgpackRedisSerializer();//2.96M

//        ObjectMapper om = new ObjectMapper();
//        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
//        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
//        serializer.setObjectMapper(om);

        template.setKeySerializer(new StringRedisSerializer()); //1
        template.setValueSerializer(serializer); //2

        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(serializer);

		//开启事务支持
		template.setEnableTransactionSupport(true);

        template.afterPropertiesSet();
        return template;
    }
}
```
5. Redis 操作方法
``` java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.ValueOperations;
import org.springframework.stereotype.Repository;

import javax.annotation.Resource;

@Repository
public class ActorRedisDao{

    @Autowired
    private RedisTemplate<Object,Object> redisTemplate;

    @Resource(name = "redisTemplate")
    ValueOperations<Object, Object> objValueOperations;

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    @Resource(name = "stringRedisTemplate")
    ValueOperations<String, String> stringValueOperations;

    public void stringRedisTemplateDmoe(String key, String value){
        stringValueOperations.set(key, value);
    }

    public void save(String key, Object obj){
        objValueOperations.set(key, obj);
    }

    public String getStr(String key){
        return stringValueOperations.get(key);
    }

    public Object getActor(String key){
       return objValueOperations.get(key);
    }
}
```
6. 在业务层注入 RedisDao 就可使用
``` java
import com.springboot.cache.dao.ActorRedisDao;
import com.springboot.cache.entity.Actor;
import com.springboot.cache.repository.ActorRepository;
import com.springboot.cache.service.ActorService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class ActorServiceImpl implements ActorService {

    @Autowired
    private ActorRepository actorRepository;

    @Autowired
    private ActorRedisDao actorRedisDao;

    /**
     * 先查Redis缓存,没有从库里查再写入缓存,手动写入Redis库
     * @param actorId
     * @return
     */
    public Actor queryById(Long actorId) {
        Actor actor = (Actor) actorRedisDao.getActor(String.valueOf(actorId));
//        return actor != null ? actor : actorRepository.findById(actorId).get();
        if(actor == null){
            actor = actorRepository.findById(actorId).get();
            actorRedisDao.save(String.valueOf(actor.getActorId()), actor);
        }
        return actor;
    }

    /**
     * 使用@Cache注解
     * 先查缓存,没有再查库,再写入缓存
     * 缓存注解使用:http://112.74.59.39/2018/06/01/spring-cache-annotation/
     * @param actorId
     * @return
     */
    @Override
    @Cacheable(key = "#actorId",value = "actor")
    public Actor queryByActorId(Long actorId) {
        Actor actor = actorRepository.findById(actorId).get();;
        return actor;
    }
}
```
**注意：**使用 redis 接口方法操作数据, 自定义的序列化方式可以起效；使用注解方式将数据写入缓存,自定义序列化的数据格式不会起效,写入到缓存的数据是二进制格式。
[文中原码 -> Github](https://github.com/gxing19/Spring-Boot-Example/tree/master/spring-boot-cache-redis)