[id]:2018-08-25
[type]:javaee
[tag]:java,spring,websocket

<h3 id="#一、背景">一、背景</h3>

&emsp;&emsp;我们都知道http协议只能浏览器单方面向服务器发起请求获得响应，服务器不能主动向浏览器推送消息。想要实现浏览器的主动推送有两种主流实现方式：

- 轮询：缺点很多，但是实现简单
- websocket：在浏览器和服务器之间建立tcp连接，实现全双工通信

&emsp;&emsp;springboot使用websocket有两种方式，一种是实现简单的websocket，另外一种是实现**STOMP**协议。这一篇实现简单的websocket，STOMP下一篇在讲。

**注意：如下都是针对使用springboot内置容器**

<h3 id="二、实现">二、实现</h3>

<h4 id="1、依赖引入">1、依赖引入</h4>

&emsp;&emsp;要使用websocket关键是`@ServerEndpoint`这个注解，该注解是javaee标准中的注解,tomcat7及以上已经实现了,如果使用传统方法将war包部署到tomcat中，只需要引入如下javaee标准依赖即可：
```xml
<dependency>
  <groupId>javax</groupId>
  <artifactId>javaee-api</artifactId>
  <version>7.0</version>
  <scope>provided</scope>
</dependency>
```
如使用springboot内置容器,无需引入，springboot已经做了包含。我们只需引入如下依赖即可：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
    <version>1.5.3.RELEASE</version>
    <type>pom</type>
</dependency>
```

<h4 id="2、注入Bean">2、注入Bean</h4>

&emsp;&emsp;首先注入一个**ServerEndpointExporter**Bean,该Bean会自动注册使用@ServerEndpoint注解申明的websocket endpoint。代码如下：
```java
@Configuration
public class WebSocketConfig {
    @Bean
    public ServerEndpointExporter serverEndpointExporter(){
        return new ServerEndpointExporter();
    }
}
```

<h4 id="3、申明endpoint">3、申明endpoint</h4>

&emsp;&emsp;建立**MyWebSocket.java**类，在该类中处理websocket逻辑
```java
@ServerEndpoint(value = "/websocket") //接受websocket请求路径
@Component  //注册到spring容器中
public class MyWebSocket {


    //保存所有在线socket连接
    private static Map<String,MyWebSocket> webSocketMap = new LinkedHashMap<>();

    //记录当前在线数目
    private static int count=0;

    //当前连接（每个websocket连入都会创建一个MyWebSocket实例
    private Session session;

    private Logger log = LoggerFactory.getLogger(this.getClass());
    //处理连接建立
    @OnOpen
    public void onOpen(Session session){
        this.session=session;
        webSocketMap.put(session.getId(),this);
        addCount();
        log.info("新的连接加入：{}",session.getId());
    }

    //接受消息
    @OnMessage
    public void onMessage(String message,Session session){
        log.info("收到客户端{}消息：{}",session.getId(),message);
        try{
            this.sendMessage("收到消息："+message);
        }catch (Exception e){
            e.printStackTrace();
        }
    }

    //处理错误
    @OnError
    public void onError(Throwable error,Session session){
        log.info("发生错误{},{}",session.getId(),error.getMessage());
    }

    //处理连接关闭
    @OnClose
    public void onClose(){
        webSocketMap.remove(this.session.getId());
        reduceCount();
        log.info("连接关闭:{}",this.session.getId());
    }

    //群发消息

    //发送消息
    public void sendMessage(String message) throws IOException {
        this.session.getBasicRemote().sendText(message);
    }

    //广播消息
    public static void broadcast(){
        MyWebSocket.webSocketMap.forEach((k,v)->{
            try{
                v.sendMessage("这是一条测试广播");
            }catch (Exception e){
            }
        });
    }

    //获取在线连接数目
    public static int getCount(){
        return count;
    }

    //操作count，使用synchronized确保线程安全
    public static synchronized void addCount(){
        MyWebSocket.count++;
    }

    public static synchronized void reduceCount(){
        MyWebSocket.count--;
    }
}
```

<h4 id="4、客户的实现">4、客户的实现</h4>

&emsp;&emsp;客户端使用h5原生websocket，部分浏览器可能不支持。代码如下：
```html
<html>

<head>
    <title>websocket测试</title>
    <meta charset="utf-8">
</head>

<body>
    <button onclick="sendMessage()">测试</button>
    <script>
        let socket = new WebSocket("ws://localhost:8080/websocket");
        socket.onerror = err => {
            console.log(err);
        };
        socket.onopen = event => {
            console.log(event);
        };
        socket.onmessage = mess => {
            console.log(mess);
        };
        socket.onclose = () => {
            console.log("连接关闭");
        };

        function sendMessage() {
            if (socket.readyState === 1)
                socket.send("这是一个测试数据");
            else
                alert("尚未建立websocket连接");
        }
    </script>
</body>

</html>
```

<h3 id="三、测试">三、测试</h3>

&emsp;&emsp;建立一个controller测试群发，代码如下：
```java
@RestController
public class HomeController {

    @GetMapping("/broadcast")
    public void broadcast(){
        MyWebSocket.broadcast();
    }
}
```
然后打开上面的html，可以看到浏览器和服务器都输出连接成功的信息：
```
浏览器：
Event {isTrusted: true, type: "open", target: WebSocket, currentTarget: WebSocket, eventPhase: 2, …}

服务端：
2018-08-01 14:05:34.727  INFO 12708 --- [nio-8080-exec-1] com.fxb.h5websocket.MyWebSocket          : 新的连接加入：0
```
点击测试按钮，可在服务端看到如下输出：
```
2018-08-01 15:00:34.644  INFO 12708 --- [nio-8080-exec-6] com.fxb.h5websocket.MyWebSocket          : 收到客户端2消息：这是一个测试数据
```
再次打开html页面，这样就有两个websocket客户端，然后在浏览器访问[localhost:8080/broadcast](localhost:8080/broadcast)测试群发功能,每个客户端都会输出如下信息：
```
MessageEvent {isTrusted: true, data: "这是一条测试广播", origin: "ws://localhost:8080", lastEventId: "", source: null, …}
```
<br/>
&emsp;&emsp;源码可在[github]()上下载，记得点赞，star哦

