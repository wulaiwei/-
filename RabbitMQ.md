### RabbitMQ
1. RabbitMQ是基于AMQP协议 高级消息队列协议。它是应用层协议的一个开放标准，为面向消息的中间件设计，基于此协议的客户端与消息中间件可传递消息，并不受产品、开发语言等条件的限制。
2. RabbitMQ常见名词
   1. Producer **生成者** 消息的产生者
   2. Consumer **消费者** 消息的处理者
   3. Connection **连接**  生产者/消费者和RabbitMQ服务器之间建立的TCP连接
   4. Channel **信道** 是TCP里面的虚拟连接。例如：Connection相当于电缆，Channel相当于独立光纤束，一条TCP连接中可以创建多条信道，增加连接效率。无论是发布消息、接收消息、订阅队列都是通过信道完成的。
   5. Broker **消息队列服务器实体** 即RabbitMQ服务器
   6. Virtual Host **虚拟主机** 出于多租户和安全因素设计的，把AMQP的基本组件划分到一个虚拟的分组中。每个vhost本质上就是一个mini版的RabbitMQ服务器，拥有自己的队列、交换机、绑定和权限机制。当多个不同的用户使用同一个RabbitMQ服务器时，可以划分出多个虚拟主机。RabbitMQ默认的虚拟主机路径是 /
   7. Exchange **交换机**  用来接收生产者发送的消息，并根据分发规则，将这些消息分发给服务器中的队列中。不同的交换机有不同的分发规则。
   8. Queue **队列** 用来保存消息直到发送给消费者。它是消息的容器，也是消息的终点。消息一直在队列里面，等待消费者链接到这个 队列将其取走
   9. Binding **绑定** 消息队列和交换机之间的虚拟连接，绑定中包含路由规则，绑定信息保存到交换机的路由表中，作为消息的分发依据
3. RabbitMQ 架构 ![架构](架构.png)
4. Queue 详解
   1. 声明队列的方法 `channel.QueueDeclare(queueName,durable,exclusive,autoDelete);`
   2. queueName 队列名称
   3. durable 持久化
   4. exclusive 独占 当配置为true则为当前channel独占，其他channel无法访问，进程结束后队列自动删除
   5. autoDelete 自动删除 [当最后一个消费者取消订阅时，至少有一个消费者的队列将被删除](https://stackoverflow.com/questions/52399134/rabbit-mq-queue-not-auto-deleted-after-consumers-zero)
5. Exchange 详解
   1. 声明Exchange的方便 `channel.ExchangeDeclare(exchangeName,type,durable,autoDelete)`
   2. exchangeName 交换器名称
   3. type 交换器类型
      1. Direct 直连
      2. Fanout 扇型
      3. Topics  主题
      4. Headers 头部
   4. durable 持久化
   5. autoDelete 自动删除 规则和队列类似
6. Direct 模式
   1. 声明 **Exchange** 类型为 **Direct** `channel.ExchangeDeclare(exchangeName,ExchangeType.Direct,durable,autoDelete)`
   2. ![direct](direct.png)
   3. direct通过 routekey 进行匹配 类似1对1
   4. 设置队列信息 `channel.QueueDeclare(queueName,durable,exclusive,autoDelete)`
   5. 绑定队列 `channel.QueueBind(queueName, exchangeName, routingKey)`
   6. 发布消息 `channel.BasicPublish(exchange: exchangeName, routingKey: routingKey, basicProperties: null, body: body)`
7. Fanout 模式
   1. 声明 **Exchange** 类型为 **Fanout** `channel.ExchangeDeclare(exchangeName,ExchangeType.Fanout,durable,autoDelete)`
   2. ![fanout](fanout.png)
   3. fanout为 1对多  不需要路由键routingKey
   ```
   channel.QueueDeclare("wk-fanout-q1", false, false, true);
   channel.QueueDeclare("wk-fanout-q2", false, false, true);
   channel.QueueBind("wk-fanout-q1", "wk-fanout", "");
   channel.QueueBind("wk-fanout-q2", "wk-fanout", "");
   for (int i = 0; i < 5; i++)
   {
       var body = Encoding.UTF8.GetBytes("消息");
       channel.BasicPublish(exchange: "wk-fanout",
          routingKey: "",
          basicProperties: null,
          body: body);
   } 
   ```
8. Topics 模式
   1. 声明 **Exchange** 类型为 **Topic** `channel.ExchangeDeclare(exchangeName,ExchangeType.Topic,durable,autoDelete)`
   2. ![toopic](topic.png)
   3. *(star) can substitute for exactly one word.
   4. #(hash) can substitute for zero or more words.
   5. 设置队列绑定关系 `channel.QueueBind("a1", "wk-topic", "*.orange.*")`
   6. 发送消息时指定 Exchange 与 routingKey `channel.BasicPublish(exchange: "wk-topic", routingKey: ".orange.", basicProperties: null, body: body1)`
   7. *.orange.* 
      1. s.orange.s match 
      2. s.orange. match
      3. .orange. match
      4. 1.1.orange. not match
      5. 1.orange.1.1 not match
   8. #.orange.#
      1. s.orange.s match
      2. s.orange. match
      3. .orange. match
      4. 1.1.orange. match
      5. 1.orange.1.1 match
      6. .orange1. not match 
9. Headers 模式
   1. 声明 **Exchange** 类型为 **Headers** `channel.ExchangeDeclare("wk-header", ExchangeType.Headers, true, true)`
   2. 设置队列信息 `channel.QueueDeclare(queueName,durable,exclusive,autoDelete)`
   3. 对于header 模式并不依赖routingKey 而是依赖队列与exchange 绑定时设置的header匹配模式 `x-match`
      1. 用法如下 
          ``` 
          channel.QueueBind("any-header","wk-header","",new Dictionary<string, object>
          {
             {"x-match","any"},
             {"k1","1"},
             {"k2","2"},
          }); 
          ```
      2. x-match 存在any 或者all 
         1. any 代表任意匹配 即send发送的header头中携带 k1:1 或者k2:2中 任意一个即可匹配
         2. all 代表全量匹配 即send发送的header头中只有携带 k1:1与k2:2才能匹配，多余不受影响
10. 消息确认
    1. 消息确认存在两个来源 
       1. 生产端发送消息给RabbitMq服务器
       2. 消费者从RabbitMq服务器取消息
    2. 生产者确认 默认情况下生成者确认模式关闭,如果需要启用则需要手动设置 `channel.ConfirmSelect()`
       1. 同步模式 支持普通Confirm模式 与 批量Confirm模式两种方式 同步确认会造成程序阻塞
          1. WaitForConfirms 每发送一条消息，立即调用WaitForConfirms就等待MQ服务器的ack响应也可以全量发送消息后调用 `channel.WaitForConfirms()`
          2. WaitForConfirmsOrDie 一旦消息被确认，该方法就会返回。如果消息在超时时间内没有得到确认或者被 nack-ed（意味着代理由于某种原因无法处理它），该方法将抛出异常。异常的处理通常包括记录错误消息和/或重试发送消息。 `channel.WaitForConfirmsOrDie(TimeSpan.FromHours(0.01))`
       2. 异步模式 
          1. BasicAcks 消息发送成功的时候进入到这个事件：即RabbitMq服务器告诉生产者，我已经成功收到了消息
          2. BasicNacks 息发送失败的时候进入到这个事件：即RabbitMq服务器告诉生产者，你发送的这条消息我没有成功的投递到Queue中，或者说我没有收到这条消息。
    3. 消费者确认
       1. 自动确认模式（automatic acknowledgement model） 
          1. autoAck设为true为自动应答，false为手动ack `channel.BasicConsume(queue: queueName, autoAck: true, consumer: consumer)`
       2. 显式确认模式（explicit acknowledgement model）
          1. 不需要设置 channel.ConfirmSelect() 消费端是不需要去指定消息的确认应答模式的，消费端本身就是监听
          2. BasicAck:肯定确认 `channel.BasicAck(deliveryTag:ea.DeliveryTag, multiple: false)`
             1. delivery tag: the sequence number identifying the confirmed or nack-ed message. We will see shortly how to correlate it with the published message.
             2. multiple：是否批量.true:将一次性拒绝所有小于deliveryTag的消息
          3. BasicNack:否定确认 `channel.BasicNack(deliveryTag: ea.DeliveryTag, multiple: false, requeue: false)`
             1. multiple：是否批量.true:将一次性拒绝所有小于deliveryTag的消息
             2. requeue:被拒绝的是否重新入队列；true：重新进入队列 fasle：抛弃此条消息
          4. BasicReject 拒绝 `channel.BasicReject(deliveryTag: ea.DeliveryTag,  requeue: false)`
             1. requeue表示是否将消息重新投递到队列中。 与basicNack的区别在于，basicReject只能拒绝单个消息，而basicNack可以拒绝多个消息。
          5. BasicRecover 恢复消息  `channel.basicRecover(true)`
             1. basic.recover是否恢复消息到队列，参数是是否requeue，true则重新入队列，并且尽可能的将之前recover的消息投递给其他消费者消费，而不是自己再次消费。false则消息会重新被投递给自己。
11. BasicQos 消费端限流 ![consumer-prefetch](https://www.rabbitmq.com/consumer-prefetch.html)
    1. 消费者设置 `channel.BasicQos(prefetchSize: 0, prefetchCount: 10, global: false)`
    2. prefetchSize：服务器将传送的最大内容量（以八位字节为单位），如果无限制则为 0。
    3. prefetchCount：会告诉RabbitMQ不要同时给一个消费者推送多于N个消息，即一旦有N个消息还没有ack，则该Consumer将block（阻塞）掉，直到有消息ack
    4. Global：true\false是否将上面设置应用于Channel；简单来说，就是上面限制是Channel级别的还是Consumer级别
12. TTL
    1. Queue TTL 
       1. 创建队列时绑定x-message-ttl 即可 单位为ms   
          ``` csharp
              channel.QueueDeclare("queue-ttl", true, false, false,new Dictionary<string, object>
             {
                  { "x-message-ttl", 10000 },
                  { "x-expires", 20000 }
             }) 
            ```
       2. 基于队列设置后 进入队列的消息默认有效期为 10s,到期后消息自动丢弃
       3. Queue 过期时间 ` x-expires `  设置队列的空闲存活时间（如该队列根本没有消费者，一直没有使用，队列可以存活多久和是否存在Messagee并无关联）
    2. Message TTL
       1. 发送消息时指定过期时间
          ```csharp
          var basicProperties = channel.CreateBasicProperties();
          basicProperties.DeliveryMode = 2;
          basicProperties.Expiration = "10000";
          channel.BasicPublish(exchange: "wk-direct-ttl-m", routingKey: "q1", basicProperties: basicProperties, body: body);
          ```
       2. 当消息到期时如果消息未消费，消息丢弃
       3. Message TTL 并不要求Queue也必须设置TTL
       4. Message TTL依旧遵顼 Queue先进显出的原则，假定A1消息过期时间未10s  A2为5s A2消息等待A1过期后立即过期，即过期时间是按发送消息的时间点
13. 死信队列 Dead Letter Exchanges ![Link](https://www.rabbitmq.com/dlx.html#using-optional-queue-arguments)
    1. 队列中的消息可能是“死信”的，这意味着当发生以下任何事件时，这些消息将重新发布到交换器。
       1. 消费者使用basic.reject或basic.nack否定确认消息，并将requeue参数设置为false。
       2. 消息由于每条消息的 TTL而过期
       3. 由于队列超出长度限制，消息被丢弃
       4. 特别注意：请注意，如果队列过期***(x-expires)***，队列中的消息不会“死信”。 
    2. 死信交换（DLX）是正常的交换。它们可以是任何常见类型并被声明为正常类型。
    3. 队列配置死信路由 
       ``` csharp
         channel.QueueDeclare("queue-ttl", true, false, false, new Dictionary<string, object>
         {
             { "x-message-ttl", 30000 },
             { "x-dead-letter-exchange", "wk-direct-ttl-dlx" },
             { "x-dead-letter-routing-key", "k1" }
         });
          channel.QueueDeclare("queue-ttl-tlx", true, false, false)
       ```
    4. 当消息过期后，这边消息会路由到exchange为 `wk-direct-ttl-dlx` routingKey 为 `x-dead-letter-routing-key` 的 queue
    5. 当未设置 ***x-dead-letter-routing-key*** 时，会默认将进入queue-ttl 自动转发过来，即如果原始routingKey为 Q1 则这边消息会路由到exchange为 `wk-direct-ttl-dlx` routingKey 为 Q1 的 queue，匹配不到则丢弃消息
    6. 死信消息会添加部分描述信息，详解文档
14. 日志追踪
    1. 启动Tracing插件 `rabbitmq-plugins enable rabbitmq_tracing`
    2. 重启RabbitMQ
    3. 添加Trace 匹配规则 
15. 消费者取消通知
    1. 当队列被删除或变得不可用时，支持消费者取消通知的客户端将始终收到通知。当复制队列的领导者发生变化时，消费者可能会请求取消它们。
    2. 客户端在同一通道上发出 basic.cancel ，这将导致消费者被取消并且服务器回复 basic.cancel -ok
    3. 接受消费取消 
         ``` csharp
           consumer.ConsumerCancelled += (s, e) =>
           {
                Trace.TraceInformation("Consumer Cancelled");
           };
       ```
16. 消费者优先级 !(Consumer Priorities)[https://www.rabbitmq.com/consumer-priority.html]
    1. 正常情况下，即不给消费者配置优先级或者消费者具有相同优先级时，监听同一个队列的多个消费者之间轮流消费消息。
    2. 如果我们给消费者配置了优先级，那么，高优先级的消费者会优先消费消息。
    3. 当高优先级消费者阻塞时，队列不会等待高优先级的消费者重新变为活跃状态，而是将消息推送给低优先级的消费者。
    4. 活跃消费者: 所谓活动的消费者，是指消费者可以在不等待的情况下，继续消费消息。
    5. 阻塞消费者: 所谓阻塞的消费者，是指消费者所在信道中未确认的消息数量达到了最大值，或者网络拥塞了，或者其他原因，导致消费者没法消费消息
    6. 消费者优先级的配置方式: 优先级默认为0，数值越大优先级越高，可以使用正数和负数。 
        ``` csharp
       channel.BasicConsume(queue: "wk-direct-q", autoAck: true, consumer: consumer, arguments: new Dictionary<string, object>
       {
           { "x-priority", 10 }
       });    
       ```
17. 最大队列长度 ![max length](https://www.rabbitmq.com/maxlength.html)
    1. 可以设置队列的最大长度。最大长度限制可以设置为消息数，或设置为字节数（所有消息正文长度的总和，忽略消息属性和任何开销），或两者兼而有之
    2. x-max-length 声明参数提供非负整数值来设置最大消息数 。
    3. x-max-length-byte 声明参数提供非负整数值来设置最大长度（ 以字节为单位）。
    4. 可以通过为x-overflow队列声明参数提供字符串值来设置溢出行为 。可能的值为drop-head（默认）、 reject-publish或 reject-publish-dlx。
       1. drop-head 删除头部数据
       2. reject-publish 拒绝发布 消息无法发送成功，开启生产端 ConfirmSelect 后可以收到
          1. channel.WaitForConfirms() 结果为false
          2. CallBack NACK (.NET客户端未能监控到NACK回调)
       3. reject-publish-dlx 路由到死信队列
       

