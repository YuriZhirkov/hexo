title: Spring Boot WebSocket Vue 前后端分离实现消息通讯
author: whsoftinc
tags:
  - SpringBoot
  - Vue
  - WebSocket
categories:
  - Jave Web
  - Vue
  - WebSocket
toc: true
date: 2019-10-19 11:44:00
---
Spring Boot 2.0 WebSocket Vue 前后端分离实现消息通讯

<Excerpt in index>
# 一、前言

目前项目中要实现一个消息推送的功能，类似于空间点赞，朋友圈新消息的提醒功能，然后马上想到websocket长连接实现的实时通讯效果，引入到了项目之中，我也做了一个小的demo,大家可以参考一下。
<!-- more -->

# 二、快速开始

1. 在**springboot**项目中添加**websocket**所需要的依赖；

   <!-- websocket -->
   	<dependency>
   		<groupId>org.springframework.boot</groupId>
   		<artifactId>spring-boot-starter-websocket</artifactId>
   	</dependency>

2. 添加**websocket**配置类；

   ```java
   @Configuration
   @EnableWebSocketMessageBroker
   public class WebSocketConfig implements  WebSocketMessageBrokerConfigurer {
   
       /**
        * 配置信息代理
        */
       @Override
       public void configureMessageBroker(MessageBrokerRegistry config) {
           // 订阅Broker名称,接受消息用户必须以这个开头的路径才能收到消息
           config.enableSimpleBroker("/user", "/topic", "/queue");
           // 全局使用的消息前缀（客户端订阅路径上会体现出来），客户端主动发送消息会以这里配置的前缀访问 @MessageMapping 配置的路径
           config.setApplicationDestinationPrefixes("/app");
           // 点对点使用的订阅前缀（客户端订阅路径上会体现出来），不设置的话，默认也是/user/
           config.setUserDestinationPrefix("/user");
       }
   
       /**
        * 注册stomp的端点
        */
       @Override
       public void registerStompEndpoints(StompEndpointRegistry registry) {
           // 允许使用socketJs方式访问，endpointSang，允许跨域
           // 在网页上我们就可以通过这个链接
           // http://localhost:8080/endpointSang
           // 来和服务器的WebSocket连接
           registry.addEndpoint("/endpointSang")
                  // .addInterceptors(new HttpSessionHandshakeInterceptor())
                   .setAllowedOrigins("*") // 允许跨域设置
                   .withSockJS();
       }
   }
   ```

3. 写一个控制器进行**广播通讯**（一个人发送消息，多个人收消息）,**点对点通讯**（一人发一人收），可以用 **SimpMessagingTemplate** 或者 **@SendTo("/topic/greetings")** 2种方式都可以进行推送消息。

   ```java	
   @RestController
   public class WsController {
   
       private  final Logger logger = LoggerFactory.getLogger(WsController.class);
   
       @Autowired
       private SimpMessagingTemplate messageTemplate;
   
       /**
        * 广播
        * @param message
        * @return
        */
       @MessageMapping("/welcome")
       @SendTo("/topic/greetings")
       public Map say(Map message) {
           logger.info(message.get("chatType").toString());
           Map res = new HashMap();
           res.put("topic", message.get("chatType").toString());
           return res;
       }
   	
   	/**
        * 请求接口推送消息-（广播）
        * @param message
        */
       @GetMapping("/sentMes")
       public void sentMes(String message) {
           logger.info(message + "xiaixixi");
           this.messageTemplate.convertAndSend("/queue/msg", message);
       }
       
       /**
        * 点对点通信
        * @param message
        */
       @MessageMapping(value = "/point")
       @SendToUser("/topic/point")
       public String point(Map message) {
           logger.info(message.get("test") + "******");
           return "dd";
       }
   
       /**
        * 点对点通信
        * @param message
        */
       @MessageMapping(value = "/points")
       public void point1(Map message) {
           logger.info(message.get("name") + "******" );
           //发送消息给指定用户, 最后接受消息的路径会变成 /user/admin/queue/points
           messageTemplate.convertAndSendToUser("admin", "/queue/points", message);
       }
   }		
   ```

   > 备注：如果是前端主动推送消息，可以用控制器请求进行处理，也可以用websocket，特有的@MessageMapping("/welcome")，第二种可以实现点对点的通讯效果。

4. 前端**vue**进行代码处理；
    	
   可以在所需要的项目中直接下载依赖，如果没有的话可以直接在新建一个工程。  
   如果不会的这里有: [vue如何新建一个项目](https://www.jianshu.com/p/02b12c600c7b).  
   下载依赖：

   ```javascript
   	npm install sockjs-client --save
   	npm install stompjs  --save
   	npm install axios --save
   ```

5. 创建vue页面引入依赖，编写主要方法；  

	**请求接口收消息**
   ```vue
   <template>
    <div class="hello">
      <h1>{{ msg }}</h1>
      请求参数：<input type="text" v-model="params"/>
      <button @click="axiosGet">axios请求</button>
      <ul>
          <li v-for="(item,i) in listData">{{item}}</li>
      </ul>
      <p><router-link to="/ws">自动发消息，收消息（广播）</router-link></p>
      <p><router-link to="/wsp">点对点，收消息发消息</router-link></p>
    </div>
  </template>

  <script>
    import SockJS from 'sockjs-client'
    import Stomp from 'stompjs'
    import axios from 'axios'
    export default {
      name: 'HelloWorld',
      data() {
        return {
          msg: '广播',
          params: '你好',
          listData: ['whdoftinc'],
          stompClient: '',
          timer: ''
        }
      },
      methods: {
          init() {
              // 建立连接对象
              let socket = new SockJS('http://127.0.0.1:8080/endpointSang')
              // 获取STOMP子协议的客户端对象
              this.stompClient = Stomp.over(socket)
              // 向服务器发起websocket连接
              this.stompClient.connect({}, () => {
                  this.stompClient.subscribe('/queue/msg', (msg) => { // 订阅服务端提供的某个queue
                  console.log('******** 成功收到消息 *****', msg)
                  console.log(msg.body) // msg.body存放的是服务端发送给我们的信息
                  this.listData.push(msg.body)
                  })
              }, (err) => {
                  // 连接发生错误时的处理函数
                  console.log('失败')
                  console.log(err);
              })
          },
          axiosGet() {
              axios.get('http://localhost:8080/sentMes?message='+this.params)
              .then(function (response) {
                  console.log(response);
              })
              .catch(function (error) {
                  console.log(error);
              })
          }
      },
      // 初始化连接
      mounted() {
        this.init();
      },
    }

  </script>
 	``` 
   

 	
 **点对点，收消息发消息** 
   ```vue
   <template>
    <div>
        <div>{{msg}}</div>
        <p><router-link to="/ws">自动发消息，收消息(广播)</router-link></p>
	    <p><router-link to="/">请求接口收消息</router-link></p>
    </div>
</template>
script>
import SockJS from  'sockjs-client';
import  Stomp from 'stompjs';
  export default {
    name: 'HelloWorld',
    data() {
      return {
        msg: "你好",  
        stompClient:'',
        timer:''
      }
    },
    methods: {
        initWebSocket() {
            this.connection()
            let that= this
            // 断开重连机制,尝试发送消息,捕获异常发生时重连
            this.timer = setInterval(() => {
                try {
                    that.stompClient.send("/app/points",
                    {},JSON.stringify({name: 'admin'}),)   //用户加入接口 
                } catch (err) {
                    console.log("断线了: " + err)
                    that.connection();
                }
            }, 5000);
        },
        // 连接服务器
        connection() {
            // 建立连接对象
            let socket = new SockJS('http://127.0.0.1:8080/endpointSang')
            // 获取STOMP子协议的客户端对象
            this.stompClient = Stomp.over(socket);
            // 定义客户端的认证信息,按需求配置
            let headers = {
                name:'admin'
            }
            let that = this
            // 向服务器发起websocket连接
            this.stompClient.connect(headers,() => {
                // 这里的路径需要和后台 发送人保持一致
                this.stompClient.subscribe('/user/admin/queue/points',  function(data) { //订阅消息
                    console.log(data, "******收到消息了********")
                    that.msg += data.body
                },headers);
            }, (err) => {
                // 连接发生错误时的处理函数
                console.log('失败')
                console.log(err)
            });
        },
        // 断开连接
        disconnect() {
            if (this.stompClient) {
                this.stompClient.disconnect();
            }
        }
    },
    // 初始化连接
    mounted(){
        this.initWebSocket()
    },
    beforeDestroy: function () {
        // 页面离开时断开连接,清除定时器
        this.disconnect()
        clearInterval(this.timer)
    }
  }
</script>
   ```
    	
> 以上是主要核心代码，效果大家可以自己尝试一下。

## 三、注意事项

#### 讲一下过程中踩得坑

- 前端发送请求的时候以@MessageMapping(value = "/point") 要以websocket中配置的config.setApplicationDestinationPrefixes("/app");
  路径则为/app/point进行请求。
- 前端接受消息的时候一定要以配置的this.messageTemplate.convertAndSend("/queue/msg", message);
  开头路径的这个/queue,
  config.enableSimpleBroker("/user", "/topic", "/queue"); 这里面一定要存在。可以配置多个。
- 广播通讯开始的很顺利，然后进行点对点通讯发消息的时候，始终接受不到消息，搞了半天才发现，一定要遵循上述所提出的config.enableSimpleBroker("/user", "/topic", "/queue");  所配置的路径才可以收到消息，发给指定用户默认会在路径加上/user/
  config.setUserDestinationPrefix("/user"); 这里可以自定义配置，但是如果配置了其它路径，一定要设置到config.enableSimpleBroker 中。

> 例如：
> config.enableSimpleBroker("/user", "/topic", "/queue");
> config.setUserDestinationPrefix("/user");
> messageTemplate.convertAndSendToUser("admin", "/queue/points", message);
>
> 最后前端接受消息的时候路径会变成/user/admin/queue/points
> user 这个路径一定要出现在config.enableSimpleBroker 中，否则收不到消息

四、代码demo参考

> 本文代码地址：https://github.com/wuhengc/springboot-websocket-vue
> 个人博客地址：https://www.whsoftinc.com
> 如有疑问请联系我：33993155（微信同号）