# springboot-rabbitmq-mqtt

[我也没想到 springboot + rabbitmq 做智能家居，会这么简单](https://mp.weixin.qq.com/s/udFE6k9pPetIWsa6KeErrA)

---

前一段有幸参与到一个智能家居项目的开发，由于之前都没有过这方面的开发经验，所以对智能硬件的开发模式和技术栈都颇为好奇。

智能可燃气体报警器
产品是一款可燃气体报警器，如果家中燃气泄露浓度到达一定阈值，报警器检测到并上传气体浓度值给后台，后台以电话、短信、微信等方式，提醒用户家中可能有气体泄漏。

用户还可能向报警器发一些关闭报警、调整音量的指令等。整体功能还是比较简单的，大致的逻辑如下图所示：图片

但当我真正的参与其中开发时，其实有一点小小的失望，因为在整个研发过程中，并没用到什么新的技术，还是常规的几种中间件，只不过换个用法而已。

技术选型用rabbitmq 来做核心的组件，主要考虑到运维成本低，组内成员使用的熟练度比较高。

下面和小伙伴分享一下如何用 springboot + rabbitmq 搭建物联网（IOT）平台，其实智能硬件也没想象的那么高不可攀！

很多小伙伴可能有点懵？rabbitmq 不是消息队列吗？怎么又能做智能硬件了？

其实rabbitmq有两种协议，我们平时接触的消息队列是用的AMQP协议，而用在智能硬件中的是MQTT协议。

## 一、什么是 MQTT协议？
MQTT 全称(Message Queue Telemetry Transport)：一种基于发布/订阅（publish/subscribe）模式的轻量级通讯协议，通过订阅相应的主题来获取消息，是物联网（Internet of Thing）中的一个标准传输协议。

该协议将消息的发布者（publisher）与订阅者（subscriber）进行分离，因此可以在不可靠的网络环境中，为远程连接的设备提供可靠的消息服务，使用方式与传统的MQ有点类似。图片

TCP协议位于传输层，MQTT 协议位于应用层，MQTT 协议构建于TCP/IP协议上，也就是说只要支持TCP/IP协议栈的地方，都可以使用MQTT协议。

## 二、为什么要用 MQTT协议？
MQTT协议为什么在物联网（IOT）中如此受偏爱？而不是其它协议，比如我们更为熟悉的 HTTP协议呢？

首先HTTP协议它是一种同步协议，客户端请求后需要等待服务器的响应。而在物联网（IOT）环境中，设备会很受制于环境的影响，比如带宽低、网络延迟高、网络通信不稳定等，显然异步消息协议更为适合IOT应用程序。

HTTP是单向的，如果要获取消息客户端必须发起连接，而在物联网（IOT）应用程序中，设备或传感器往往都是客户端，这意味着它们无法被动地接收来自网络的命令。

通常需要将一条命令或者消息，发送到网络上的所有设备上。HTTP要实现这样的功能不但很困难，而且成本极高。

## 三、MQTT协议介绍
前边说过MQTT是一种轻量级的协议，它只专注于发消息， 所以此协议的结构也非常简单。

MQTT数据包
在MQTT协议中，一个MQTT数据包由：固定头（Fixed header）、 可变头（Variable header）、 消息体（payload）三部分构成。

固定头（Fixed header），所有数据包中都有固定头，包含数据包类型及数据包的分组标识。
可变头（Variable header），部分数据包类型中有可变头。
内容消息体（Payload），存在于部分数据包类，是客户端收到的具体消息内容。
图片
在这里插入图片描述
### 1、固定头
固定头部，使用两个字节，共16位：图片(4-7)位表示消息类型，使用4位二进制表示，可代表如下的16种消息类型，不过 0 和 15位置属于保留待用，所以共14种消息事件类型。图片

DUP Flag（重试标识）

DUP Flag：保证消息可靠传输，消息是否已送达的标识。默认为0，只占用一个字节，表示第一次发送，当值为1时，表示当前消息先前已经被传送过。

QoS Level（消息质量等级）

QoS Level：消息的质量等级，后边会详细介绍

RETAIN（持久化）

值为1：表示发送的消息需要一直持久保存，而且不受服务器重启影响，不但要发送给当前的订阅者，且以后新加入的客户端订阅了此Topic，订阅者也会马上得到推送。注意：新加入的订阅者，只会取出最新的一个RETAIN flag = 1的消息推送。

值为0：仅为当前订阅者推送此消息。

Remaining Length（剩余长度）

在当前消息中剩余的byte(字节)数，包含可变头部和消息体payload。

### 2、可变头
固定头部仅定义了消息类型和一些标志位，一些消息的元数据需要放入可变头部中。可变头部内容字节长度 + 消息体payload = 剩余长度。

可变头部居于固定头部和payload中间，包含了协议名称，版本号，连接标志，用户授权，心跳时间等内容。

可变头存在于这些类型的消息：PUBLISH (QoS > 0)、PUBACK、PUBREC、PUBREL、PUBCOMP、SUBSCRIBE、SUBACK、UNSUBSCRIBE、UNSUBACK。

### 3、消息体payload
消息体payload只存在于CONNECT、PUBLISH、SUBSCRIBE、SUBACK、UNSUBSCRIBE这几种类型的消息：

CONNECT：包含客户端的ClientId、订阅的Topic、Message以及用户名和密码。
PUBLISH：向对应主题发送消息。
SUBSCRIBE：要订阅的主题以及QoS。
SUBACK：服务器对于SUBSCRIBE所申请的主题及QoS进行确认和回复。
UNSUBSCRIBE：取消要订阅的主题。
消息质量（QoS ）
消息质量（Quality of Service），即消息的发送质量，发布者（publisher）和订阅者（subscriber）都可以指定qos等级，有QoS 0、QoS 1、QoS 2三个等级。

下边分别说明一下这三个等级的区别。

1、Qos 0
Qos 0：At most once（至多一次）只发送一次消息，不保证消息是否成功送达，没有确认机制，消息可能会丢失或重复。

图片
图片源于网络，如有侵权联系删除
2、Qos 1
Qos 1：At least once（至少一次），相对于QoS 0而言Qos 1增加了ack确认机制，发送者（publisher）推送消息到MQTT代理（broker）时，两者自身都会先持久化消息，只有当publisher 或者 Broker分别收到 PUBACK确认时，才会删除自身持久化的消息，否则就会重发。

但有个问题，尽管我们可以通过确认来保证一定收到客户端 或 服务器的message，可我们却不能保证仅收到一次message，也就是当客户端publisher没收到Broker的puback或者 Broker没有收到subscriber的puback，那么就会一直重发。

publisher -> broker 大致流程：

publisher store msg -> publish ->broker （传递message）
broker -> puback -> publisher delete msg （确认传递成功）
图片
图片源于网络，如有侵权联系删除
3、Qos 2
Qos 2：Exactly once（只有一次），相对于QoS 1，QoS 2升级实现了仅接受一次message，publisher 和 broker 同样对消息进行持久化，其中 publisher 缓存了message和 对应的msgID，而 broker 缓存了 msgID，可以保证消息不重复，由于又增加了一个confirm 机制，整个流程变得复杂很多。

publisher -> broker 大致流程：

publisher store msg -> publish ->broker -> broker store
msgID（传递message） broker -> puberc （确认传递成功）
publisher -> pubrel ->broker delete msgID （告诉broker删除msgID）
broker -> pubcomp -> publisher delete msg （告诉publisher删除msg）
图片
图片源于网络，如有侵权联系删除
LWT（最后遗嘱）
LWT 全称为 Last Will and Testament，其实遗嘱是一个由客户端预先定义好的主题和对应消息，附加在CONNECT的数据包中，包括遗愿主题、遗愿 QoS、遗愿消息等。

当MQTT代理 Broker 检测到有客户端client非正常断开连接时，再由服务器主动发布此消息，然后相关的订阅者会收到消息。

举个栗子：聊天室中所有人都订阅一个叫talk的主题 ，但小富由于网络抖动突然断开了链接，这时聊天室中所有订阅主题 talk的客户端都会收到一个 “小富离开聊天室” 的遗愿消息。

遗嘱的相关参数：

Will Flag：是否使用 LWT，1 开启
Will Topic：遗愿主题名，不可使用通配符
Will Qos：发布遗愿消息时使用的 QoS
Will Retain：遗愿消息的 Retain 标识
Will Message：遗愿消息内容
那客户端Client 有哪些场景是非正常断开连接呢？

Broker 检测到底层的 I/O 异常；
客户端 未能在心跳 Keep Alive 的间隔内和 Broker 进行消息交互；
客户端 在关闭底层 TCP 连接前没有发送 DISCONNECT 数据包；
客户端 发送错误格式的数据包到 Broker，导致关闭和客户端的连接等。
注意：当客户端通过发布 DISCONNECT 数据包断开连接时，属于正常断开连接，并不会触发 LWT 的机制，与此同时Broker 还会丢弃掉当前客户端在连接时指定的相关 LWT 参数。

## 四、MQTT协议应用场景
MQTT协议广泛应用于物联网、移动互联网、智能硬件、车联网、电力能源等领域。使用的场景也是非常非常多，下边列举一些：

物联网M2M通信，物联网大数据采集
Android消息推送，WEB消息推送
移动即时消息，例如Facebook Messenger
智能硬件、智能家具、智能电器
车联网通信，电动车站桩采集
智慧城市、远程医疗、远程教育
电力、石油与能源等行业市场
## 五、代码实现
具体 rabbitmq 的环境搭建就不赘述了，网上教程比较多，有条件的用服务器，没条件的像我搞个Windows版的也很快乐嘛。

图片
在这里插入图片描述
### 1、启用 rabbitmq的mqtt协议
我们先开启 rabbitmq 的 mqtt协议，因为默认安装下是关闭的，命令如下：

rabbitmq-plugins enable rabbitmq_mqtt
### 2、mqtt 客户端依赖包
上一步中安装rabbitmq环境并开启 mqtt协议后，实际上mqtt 消息代理服务就搭建好了，接下来要做的就是实现客户端消息的推送和订阅。

这里使用spring-integration-mqtt、org.eclipse.paho.client.mqttv3两个工具包实现。
```
<!--mqtt依赖包-->
<dependency>
    <groupId>org.springframework.integration</groupId>
    <artifactId>spring-integration-mqtt</artifactId>
</dependency>
<dependency>
    <groupId>org.eclipse.paho</groupId>
       <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
    <version>1.2.0</version>
</dependency>
```

### 3、消息发送者
消息的发送比较简单，主要是应用到@ServiceActivator注解，需要注意messageHandler.setAsync属性，如果设置成false，关闭异步模式发送消息时可能会阻塞。
```
@Configuration
public class IotMqttProducerConfig {

    @Autowired
    private MqttConfig mqttConfig;

    @Bean
    public MqttPahoClientFactory mqttClientFactory() {
        DefaultMqttPahoClientFactory factory = new DefaultMqttPahoClientFactory();
        factory.setServerURIs(mqttConfig.getServers());
        return factory;
    }

    @Bean
    public MessageChannel mqttOutboundChannel() {
        return new DirectChannel();
    }

    @Bean
    @ServiceActivator(inputChannel = "iotMqttInputChannel")
    public MessageHandler mqttOutbound() {
        MqttPahoMessageHandler messageHandler = new MqttPahoMessageHandler(mqttConfig.getServerClientId(), mqttClientFactory());
        messageHandler.setAsync(false);
        messageHandler.setDefaultTopic(mqttConfig.getDefaultTopic());
        return messageHandler;
    }
}
```

MQTT 对外提供发送消息的API时，需要使用@MessagingGateway 注解，去提供一个消息网关代理，参数defaultRequestChannel 指定发送消息绑定的channel。

可以实现三种API接口，payload 为发送的消息，topic 发送消息的主题，qos 消息质量。
```
@MessagingGateway(defaultRequestChannel = "iotMqttInputChannel")
public interface IotMqttGateway {

    // 向默认的 topic 发送消息
    void sendMessage2Mqtt(String payload);
    // 向指定的 topic 发送消息
    void sendMessage2Mqtt(String payload,@Header(MqttHeaders.TOPIC) String topic);
    // 向指定的 topic 发送消息，并指定服务质量参数
    void sendMessage2Mqtt(@Header(MqttHeaders.TOPIC) String topic, @Header(MqttHeaders.QOS) int qos, String payload);
}
```


### 4、消息订阅
消息订阅和我们平时用的MQ消息监听实现思路基本相似，@ServiceActivator注解表明当前方法用于处理MQTT消息，inputChannel 参数指定了用于接收消息的channel。
```
/**
 * @Author: xiaofu
 * @Description: 消息订阅配置
 * @date 2020/6/8 18:24
 */
@Configuration
public class IotMqttSubscriberConfig {

    @Autowired
    private MqttConfig mqttConfig;

    @Bean
    public MqttPahoClientFactory mqttClientFactory() {
        DefaultMqttPahoClientFactory factory = new DefaultMqttPahoClientFactory();
        factory.setServerURIs(mqttConfig.getServers());
        return factory;
    }

    @Bean
    public MessageChannel iotMqttInputChannel() {
        return new DirectChannel();
    }

    @Bean
    public MessageProducer inbound() {
        MqttPahoMessageDrivenChannelAdapter adapter = new MqttPahoMessageDrivenChannelAdapter(mqttConfig.getClientId(), mqttClientFactory(), mqttConfig.getDefaultTopic());
        adapter.setCompletionTimeout(5000);
        adapter.setConverter(new DefaultPahoMessageConverter());
        adapter.setQos(1);
        adapter.setOutputChannel(iotMqttInputChannel());
        return adapter;
    }

    /**
     * @author xiaofu
     * @description 消息订阅
     * @date 2020/6/8 18:20
     */
    @Bean
    @ServiceActivator(inputChannel = "iotMqttInputChannel")
    public MessageHandler handlerTest() {

        return message -> {
            try {
                String string = message.getPayload().toString();
                System.out.println("接收到消息：" + string);
            } catch (MessagingException ex) {
                //logger.info(ex.getMessage());
            }
        };
    }
}
```
## 六、测试消息
额~ 由于本渣渣对硬件一窍不通，为了模拟硬件的发送消息，只能借助一下工具，其实硬件端实现MQTT协议，跟我们前边的基本没什么区别，只不过换种语言嵌入到硬件中而已。

这里选的测试工具为mqttbox，下载地址：http://workswithweb.com/mqttbox.html

1、测试消息发送
我们用先用mqttbox模拟向主题mqtt_test_topic发送消息，看后台是否能成功接收到。

图片看到后台成功拿到了向主题mqtt_test_topic发送的消息。图片

2、测试消息订阅
用mqttbox模拟订阅主题mqtt_test_topic，在后台向主题mqtt_test_topic发送一条消息，这里我简单的写了个controller调用API发送消息。

http://127.0.0.1:8080/fun/testMqtt?topic=mqtt_test_topic&message=我是后台向主题 mqtt_test_topic 发送的消息图片我们看mqttbox的订阅消息，已经成功的接收到了后台的消息，到此我们的MQTT通信环境就算搭建成功了。如果把mqttbox工具换成具体硬件设备，整个流程就是我们常说的智能家居了，其实真的没那么难。图片

## 七、应用注意事项
在我们实际的生产环境中遇到过的问题，这里分享一下让大家少踩坑。

clientId 要唯一
在客户端connect连接的时，会有一个clientId 参数，需要每个客户端都保持唯一的。但我们在开发测试阶段clientId直接在代码中写死了，而且服务都是单实例部署，并没有暴露出什么问题。

MqttPahoMessageDrivenChannelAdapter(mqttConfig.getClientId(), mqttClientFactory(), mqttConfig.getDefaultTopic());
然而在生产环境内侧的时候，由于服务是多实例集群部署，结果出现了下边的奇怪问题。同一时间内只能有一个客户端能拿到消息，其他客户端不但不能消费消息，而且还在不断的掉线重连：Lost connection: 已断开连接; retrying...。

图片这就是由于clientId相同导致客户端间相互竞争消费，最后将clientId获取方式换成从发号器中拿，问题就好了，所以这个地方是需要特别注意的。

平时程序在开发环境没问题，可偏偏到了生产环境就一大堆问题，很多都是因为服务部署方式不同导致的。所以多学习分布式还是很有必要的。

## 八、其他中间件
MQTT它只是一种协议，支持MQTT协议的消息中间件产品非常多，下边的也只是其中的一部分

Mosquitto
Eclipse Paho
RabbitMQ
Apache ActiveMQ
HiveMQ
JoramMQ
ThingMQ
VerneMQ
Apache Apollo
emqttd Xively
IBM Websphere .....
总结
我也是第一次做和硬件相关的项目，之前听到智能家居都会觉得好高大上，但实际上手开发后发现，技术嘛万变不离其宗，也只是换种用法而已。

双手奉上项目 demo 的github地址 ：https://github.com/chengxy-nds/springboot-rabbitmq-mqtt.git,感兴趣的小伙伴可以下载跑一跑，实现起来非常的简单。
