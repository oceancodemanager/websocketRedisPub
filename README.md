# websocketRedisPub
分布式环境使用websocket，利用redis
集群环境中使用websocket时session分布在不同的server，在某个server中进行的操作如何调用到另外一个server中的session进行sendMessage，利用redis的发布/订阅实现。

1. InterviewQueueWebsocket
 
 @ServerEndpoint(value = "/wsocket/xxx/{id}", configurator = HttpSessionConfigurator.class)
public class MyWebsocket {
   WebsocketListener           listener;

    /**
     * onOpen的时候添加监听
     */
    @OnOpen
    public void onOpen(@PathParam("id") Integer id, Session session, EndpointConfig config) {
       RedisMessageListenerContainer redisMessageListenerContainer = (RedisMessageListenerContainer) ApplicationContext
                    .getBean("redisListenerContainer");
            StringRedisTemplate stringRedisTemplate = (StringRedisTemplate) ApplicationContext
                    .getBean("stringRedisTemplate");
            logger.debug("ws:" + this.toString());
            logger.debug("listener:" + listener);
            listener = new WebsocketListener(session, stringRedisTemplate);
            logger.debug("redisMessageListenerContainer：" + redisMessageListenerContainer);
            redisMessageListenerContainer.addMessageListener(listener, new ChannelTopic(getRedisTopic(id)));
    }
   @OnClose
    public void onClose(Session session) {

        RedisMessageListenerContainer redisMessageListenerContainer = (RedisMessageListenerContainer) ApplicationContext
                .getBean("redisListenerContainer");
        if (listener != null) {
            redisMessageListenerContainer.removeMessageListener(listener);
        }
   }
   
   /**
   其他操作需要有session操作的时候触发的方法
   */
       public static void sendMessageToQueuePage(int type,Integer id) {

        
        String messageStr = JSON.toJSONString(message);
        PublishService publishService = (PublishService) ApplicationContext.getBean("publishService");
        publishService.publish(getRedisTopic(id), messageStr);

    }
    
}

2. WebsocketListener redis监听

public class WebsocketListener implements MessageListener {

    private static final Logger logger = LoggerFactory.getLogger(WebsocketListener.class);

    private Session             session;

    StringRedisTemplate         stringRedisTemplate;

    public WebsocketListener() {

        // TODO Auto-generated constructor stub
    }

    public WebsocketListener(Session session, StringRedisTemplate stringRedisTemplate) {

        this.session = session;
        this.stringRedisTemplate = stringRedisTemplate;
    }

    // public StringRedisTemplate getRedisTemplate() {
    // return redisTemplate;
    // }
    //
    // public void setRedisTemplate(StringRedisTemplate redisTemplate) {
    // this.redisTemplate = redisTemplate;
    // }
    public Session getSession() {

        return session;
    }

    public void setSession(Session session) {

        this.session = session;
    }

    @Override
    public void onMessage(Message message, byte[] pattern) {

        if (message != null && message.getBody() != null) {
            String msg = stringRedisTemplate.getStringSerializer().deserialize(message.getBody());
            logger.info(new String(pattern) + "主题发布：" + msg);
            if (null != session && session.isOpen()) {
                try {
                    session.getBasicRemote().sendText(msg);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }

    }

    public StringRedisTemplate getStringRedisTemplate() {

        return stringRedisTemplate;
    }

    public void setStringRedisTemplate(StringRedisTemplate stringRedisTemplate) {

        this.stringRedisTemplate = stringRedisTemplate;
    }


3.PublishService

@Component
public class PublishService {

    @Autowired(required = false)
    private StringRedisTemplate redisTemplate;

    public void publish(String topic, Object message) {

        // 该方法封装的 connection.publish(rawChannel, rawMessage);
        redisTemplate.convertAndSend(topic, message);
    }
}

4.配置文件
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:redis="http://www.springframework.org/schema/redis"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                        http://www.springframework.org/schema/beans/spring-beans.xsd
                        http://www.springframework.org/schema/context
                        http://www.springframework.org/schema/context/spring-context.xsd
http://www.springframework.org/schema/redis http://www.springframework.org/schema/redis/spring-redis.xsd">
	<!-- 
	<bean id="jedisPoolConfig" class="redis.clients.jedis.JedisPoolConfig">
        <property name="testOnBorrow" value="true" />
    </bean>
    <bean id="jedisConnectionFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory"
          p:host-name="127.0.0.1" p:port="6379" p:password="password"
          p:pool-config-ref="jedisPoolConfig" p:usePool="true"/>
    -->
    <!-- Redis连接-->
    <!--<bean id="jedisConnectionFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory"
          p:host-name="192.168.19.129" p:port="6379" p:password="123456">
        <constructor-arg ref="jedisPoolConfig" />
    </bean>-->
    <bean id="stringRedisTemplate" class="org.springframework.data.redis.core.StringRedisTemplate" p:connection-factory-ref="myRedisConnectionFactory"/>
    <!-- 缓存序列化方式 -->
    <bean id="keySerializer" class="org.springframework.data.redis.serializer.StringRedisSerializer" />
    <bean id="valueSerializer" class="org.springframework.data.redis.serializer.StringRedisSerializer" />
    <bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
        <property name="connectionFactory" ref="myRedisConnectionFactory" />
        <property name="keySerializer" ref="keySerializer" />
        <property name="valueSerializer" ref="valueSerializer" />
        <property name="hashKeySerializer" ref="keySerializer" />
        <property name="hashValueSerializer" ref="valueSerializer" />
    </bean>
    <bean id="redisListenerContainer" class="org.springframework.data.redis.listener.RedisMessageListenerContainer">
        <property name="connectionFactory" ref="myRedisConnectionFactory"/>
    </bean>
   <!-- <bean id="serverEndpointExporter" class="org.springframework.web.socket.server.standard.ServerEndpointExporter"/>-->
    <!--序列化-->
    <bean id="jdkSerializer" class="org.springframework.data.redis.serializer.JdkSerializationRedisSerializer" />
 </beans>   
