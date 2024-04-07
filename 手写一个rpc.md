# 手写一个rpc

## rpc总体框架

### 架构图

简略版：

<img src="C:\Users\汪思敏\AppData\Roaming\Typora\typora-user-images\image-20240328160620485.png" alt="image-20240328160620485" style="zoom:67%;" />

详细版：

<img src="C:\Users\汪思敏\AppData\Roaming\Typora\typora-user-images\image-20240328160033563.png" alt="image-20240328160033563" style="zoom: 80%;" />

服务提供端 Server 向注册中心注册服务，服务消费者 Client 通过注册中心拿到服务相关信息，然后再通过网络请求服务提供端 Server

实现一个最基本的 RPC 框架应该至少包括下面几部分:

1. **注册中心** ：注册中心负责服务地址的注册与查找，相当于目录服务。
2. **网络传输** ：既然我们要调用远程的方法，就要发送网络请求来传递目标类和方法的信息以及方法的参数等数据到服务提供端。
3. **序列化和反序列化** ：要在网络传输数据就要涉及到**序列化**。
4. **动态代理** ：屏蔽远程方法调用的底层细节。
5. **负载均衡** ： 避免单个服务器响应同一请求，容易造成服务器宕机、崩溃等问题。
6. **传输协议** ：这个协议是客户端（服务消费方）和服务端（服务提供方）交流的基础。

## 实现

### 注册中心

注册中心负责服务**地址的注册与查找**，相当于目录服务。本项目使用zookeeper作为注册中心，常用的还有Nacos和Redis。

ZooKeeper 通常被用于实现诸如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master 选举、分布式锁和分布式队列等功能。并且，ZooKeeper 将数据保存在`内存`中，性能是非常棒的。 在`“读”多于“写”`的应用程序中尤其地高性能，因为“写”会导致所有的服务器间同步状态。（“读”多于“写”是协调服务的典型场景）。

#### ZooKeeper 

##### 客户端 Curator

通过 `CuratorFrameworkFactory` 创建 `CuratorFramework` 对象，然后再调用  `CuratorFramework` 对象的 `start()` 方法即可

```java
private static final int BASE_SLEEP_TIME = 1000;
private static final int MAX_RETRIES = 3;

// Retry strategy. Retry 3 times, and will increase the sleep time between retries.
RetryPolicy retryPolicy = new ExponentialBackoffRetry(BASE_SLEEP_TIME, MAX_RETRIES);
CuratorFramework zkClient = CuratorFrameworkFactory.builder()
    // the server to connect to (can be a server list)
    .connectString("127.0.0.1:2181")
    .retryPolicy(retryPolicy)
    .build();
zkClient.start();
```

##### 节点

1. **节点类型**

   - **持久（PERSISTENT）节点** ：一旦创建就一直存在即使 ZooKeeper 集群宕机，直到将其删除。

   - **临时（EPHEMERAL）节点** ：临时节点的生命周期是与 **客户端会话（session）** 绑定的，**会话消失则节点消失** 。并且，临时节点 **只能做叶子节点** ，不能创建子节点。

   - **持久顺序（PERSISTENT_SEQUENTIAL）节点** ：除了具有持久（PERSISTENT）节点的特性之外， 子节点的名称还具有顺序性。比如 `/node1/app0000000001` 、`/node1/app0000000002` 。

   - **临时顺序（EPHEMERAL_SEQUENTIAL）节点** ：除了具备临时（EPHEMERAL）节点的特性之外，子节点的名称还具有顺序性。

2. **创建节点**

父节点不存在的时候自动创建父节点：

```java
zkClient.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath("/node1/00001");
```

创建临时节点

```java
zkClient.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL).forPath("/node1/00001");
```

创建节点并指定数据内容

```java
zkClient.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL).forPath("/node1/00001","java".getBytes());
zkClient.getData().forPath("/node1/00001");//获取节点的数据内容，获取到的是 byte数组
```

检测节点是否创建成功

```java
zkClient.checkExists().forPath("/node1/00001");//不为null的话，说明节点创建成功
```

3. **删除节点**

删除一个子节点

```java
zkClient.delete().forPath("/node1/00001");
```

删除一个节点以及其下的所有子节点

```java
zkClient.delete().deletingChildrenIfNeeded().forPath("/node1");
```

获取/更新节点数据内容

```java
zkClient.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL).forPath("/node1/00001","java".getBytes());
zkClient.getData().forPath("/node1/00001");//获取节点的数据内容
zkClient.setData().forPath("/node1/00001","c++".getBytes());//更新节点数据内容
```

获取某个节点的所有子节点路径

```java
List<String> childrenPaths = zkClient.getChildren().forPath("/node1");
```

##### 监听器

注册了监听器之后，这个节点的子节点发生变化比如增加、减少或者更新的时候，可以自定义回调操作。

```java
String path = "/node1";
PathChildrenCache pathChildrenCache = new PathChildrenCache(zkClient, path, true);
PathChildrenCacheListener pathChildrenCacheListener = (curatorFramework, pathChildrenCacheEvent) -> {
    // do something
};
pathChildrenCache.getListenable().addListener(pathChildrenCacheListener);
pathChildrenCache.start();
```

- **节点事件类型：**

```java
public static enum Type {
        CHILD_ADDED,//子节点增加
        CHILD_UPDATED,//子节点更新
        CHILD_REMOVED,//子节点被删除
        CONNECTION_SUSPENDED,
        CONNECTION_RECONNECTED,
        CONNECTION_LOST,
        INITIALIZED;
   }
```

### 网络传输

网络传输即**发送网络请求来传递目标类和方法的信息以及方法的参数等数据到服务提供端**，本项目使用Netty做网络传输，常用的还有Socket和NIO。

#### **Socket 网络通信**

##### 什么是Socket

Socket 是一个抽象概念，应用程序可以通过它发送或接收数据。套接字是 IP 地址与端口的组合：
$$
Socket=（IP 地址：端口号）
$$
在 Java 开发中使用 Socket 时会常用到两个类，都在 `java.net` 包中：

1. `Socket`：一般用于客户端
2. `ServerSocket` ：用于服务端

##### Socket 网络通信过程

- Socket 网络通信过程简单来说分为下面 4 步：

1. 建立服务端并且监听客户端请求
2. 客户端请求，服务端和客户端建立连接
3. 两端之间可以传递数据
4. 关闭资源

Socket 网络通信过程如下图所示：

<img src="C:\Users\汪思敏\AppData\Roaming\Typora\typora-user-images\image-20240329150944811.png" alt="image-20240329150944811" style="zoom:67%;" />

- 对应到服务端和客户端的话，是下面这样的。

**服务器端：**

1. 创建 `ServerSocket` 对象并且绑定地址（ip）和端口号(port)：`server.bind(new InetSocketAddress(host, port))`
2. 通过 `accept()`方法监听客户端请求
3. 连接建立后，通过输入流读取客户端发送的请求信息
4. 通过输出流向客户端发送响应信息
5. 关闭相关资源

**客户端：**

1. 创建`Socket` 对象并且连接指定的服务器的地址（ip）和端口号(port)：`socket.connect(inetSocketAddress)`
2. 连接建立后，通过输出流向服务器端发送请求信息
3. 通过输入流获取服务器响应的信息
4. 关闭相关资源

##### 局限

1. `ServerSocket` 的 accept () 方法是**阻塞方法**，也就是说 `ServerSocket` 在调用accept ()  等待客户端的连接请求时会阻塞，直到收到客户端发送的连接请求才会继续往下执行代码。
2. 只能同时处理一个客户端的连接，如果需要管理多个客户端的话，就需要为我们请求的客户端单独创建一个线程。每次使用都创建线程会造成资源浪费，可以使用**线程池**，创建和回收的成本较低，并且可以指定最大线程数量。

#### Netty

##### 基础知识

**介绍：**

1. **Netty 是一个基于 NIO 的 client-server(客户端服务器)框架，使用它可以快速简单地开发网络应用程序。**
2. 它极大地简化并简化了 TCP 和 UDP 套接字服务器等网络编程,并且性能以及安全性等很多方面甚至都要更好。
3. 支持多种协议如 FTP，SMTP，HTTP 以及各种二进制和基于文本的传统协议

**使用场景：**

- 作为 RPC 框架的网络通信工具
- 实现一个自己的 HTTP 服务器 
- 实现一个即时通讯系统
- 消息推送系统

**管道和通道：**

- Channel 表示一个网络通道，它负责在客户端和服务器之间进行数据的读写操作
- ChannelPipeline 是一个处理器链，它由一系列的 `ChannelHandler` （可以自定义）组成，用于处理进出 `Channel` 的事件和数据。每个 `Channel` 都有自己的ChannelPipeline ，可以对数据进行解码、编码、处理等操作。

`.handler()`方法用于设置一些针对于整个Channel的处理器，这些处理器通常在Channel的生命周期中只需要设置一次。因此，Netty提供了两个方法用于设置处理器，分别是`.handler()`和`.childHandler()`。其中，`.handler()`方法设置的处理器是针对ServerBootstrap所创建的ServerChannel的，而`.childHandler()`方法设置的处理器是针对ServerBootstrap所接受的连接的Channel的。

##### 工作原理

Netty入门：https://blog.csdn.net/S1124654/article/details/125489407

<img src="https://img-blog.csdnimg.cn/5acd384830574aa69f7a2e57fe3f6867.webp" alt="img" style="zoom:67%;" />

说明如下：

1. Netty抽象出两组线程池： BossGroup 专门负责接收客户端的连接, WorkerGroup 专门负责网络的读写。BossGroup 和 WorkerGroup 类型都是 NioEventLoopGroup
2. NioEventLoopGroup 相当于一个事件循环组, 这个组中含有多个事件循环 ，每一个事件循环是 NioEventLoop
3. NioEventLoop 表示一个不断循环的执行处理任务的线程， 每个NioEventLoop 都有一个selector , 用于监听绑定在其上的socket的网络通讯
4. NioEventLoopGroup 可以有多个线程, 即可以含有多个NioEventLoop
5. 每个Boss NioEventLoop 循环执行的步骤有3步
   - 轮询accept 事件
   - 处理accept 事件 , 与client建立连接 , 生成NioScocketChannel , 并将其注册到某个worker NIOEventLoop 上的 selector
   - 处理任务队列的任务 ， 即 runAllTasks
6. 每个 Worker NIOEventLoop 循环执行的步骤
   - 轮询read, write 事件
   - 处理i/o事件， 即read , write 事件，在对应NioScocketChannel 处理
   - 处理任务队列的任务 ， 即 runAllTasks
7. 每个Worker NIOEventLoop 处理业务时，会使用pipeline(管道), pipeline 中包含了 channel , 即通过pipeline 可以获取到对应通道, 管道中维护了很多的 处理器
   

##### ByteBuf类

**工作原理：**

ByteBuf维护了两个不同的索引：一个用于**读取**，一个用于**写入**。当你从ByteBuf读取时，它的readerIndex将会被递增已经被读取的字节数。同样地，当你写入ByteBuf时，它的writerIndex也会被递增。

<img src="https://img2020.cnblogs.com/blog/780676/202008/780676-20200828161154230-1122648307.png" alt="img" style="zoom:80%;" />

名称以 `set `或者 `get `开头的ByteBuf方法，将会推进其对应的索引，而名称以set或者get开关的操作则不会。

#### 传输协议

通过设计协议，我们定义需要传输哪些类型的数据， 并且还会规定每一种类型的数据应该占多少字节。这样我们在接收到二级制数据之后，就可以正确的解析出我们需要的数据。

以下是设计的传输协议：

<img src="C:\Users\汪思敏\AppData\Roaming\Typora\typora-user-images\image-20240407205445929.png" alt="image-20240407205445929" style="zoom:80%;" />

- **魔法数** ： 通常是 4 个字节。这个魔数主要是为了筛选来到服务端的数据包，有了这个魔数之后，服务端首先取出前面四个字节进行比对，能够在第一时间识别出这个数据包并非是遵循自定义协议的，也就是无效数据包，为了安全考虑可以直接关闭连接以节省资源。
- **序列化器类型** ：标识序列化的方式，比如是使用 Java 自带的序列化，还是 json，kyro 等序列化方式。
- **消息长度** ： 运行时计算出来。



#### 实现

1. 定义 **RpcRequest.java**
   - 包含了要调用的目标方法和类的名称、参数等数据

2. 定义 **RpcResponse.java**
   - 调用结果就通过 RpcResponse 返回给客户端

3. 在Netty框架下实现。

   - Netty 客户端主要实现了 **NettyClient.java**:

     - doConnect() : 用于连接服务端（目标方法所在的服务器）并返回对应的 Channel。当我们知道了服务端的地址之后，我们就可以通过 NettyClient 成功连接服务端了。（有了 Channel 之后就能发送数据到服务端了）

     - sendRpcRequest() : 用于传输 rpc 请求(RpcRequest) 到服务端。
     - NettyClientHandler类：用于处理服务器发送的数据。

   - Netty 服务端主要实现了 **NettyRpcServer.java**:
     - NettyServerHandler类：用于处理客户端发送的数据。



### 序列化和反序列化

**为什么需要序列化和反序列化呢？** 
因为网络传输的数据必须是二进制的。因此，我们的 Java 对象没办法直接在网络中传输。为了能够让 Java 对象在网络中传输我们需要将其**序列化**为二进制的数据。我们最终需要的还是目标 Java 对象，因此我们还要将二进制的数据“解析”为目标 Java 对象，也就是对二进制数据再进行一次**反序列化**。

<img src="C:\Users\汪思敏\AppData\Roaming\Typora\typora-user-images\image-20240328161302687.png" alt="image-20240328161302687" style="zoom:80%;" />

现在比较常用序列化的有 hessian、**kryo**、protostuff，JSON 和 XML 这种属于文本类序列化方式。虽然可读性比较好，但是性能较差

#### Java自带序列化方式

JDK 自带的序列化，只需实现 `java.io.Serializable`接口即可。

1. **serialVersionUID 有什么作用？**

序列化号 `serialVersionUID` 属于版本控制的作用。反序列化时，会检查 `serialVersionUID` 是否和当前类的 `serialVersionUID` 一致。

2. **serialVersionUID 不是被 static 变量修饰了吗？为什么还会被“序列化”？**

static修饰的变量是静态变量，位于方法区，本身是不会被序列化的。static变量是属于类的而不是对象。反序列之后，static变量的值就像是默认赋予给了对象一样，看着就像是static变量被序列化，实际只是假象罢了。

3. **有些字段不想进行序列化怎么办？**

使用 `transient` 关键字修饰。

- transient 只能修饰 `变量`，不能修饰类和方法。
- transient 修饰的变量，在反序列化后变量值将会被置成类型的默认值。例如，如果是修饰int 类型，那么反序列后结果就是0。
- static 变量因为不属于任何对象(Object)，所以无论有没有 transient 关键字修饰，均不会被序列化。

4. **为什么不推荐使用 JDK 自带的序列化？**

- **不支持跨语言调用** : 如果调用的是其他语言开发的服务的时候就不支持了。
- **性能差** ：相比于其他序列化框架性能更低，主要原因是序列化之后的字节数组体积较大，导致传输成本加大。
- **存在安全问题** ：序列化和反序列化本身并不存在问题。但当输入的反序列化的数据可被用户控制，那么攻击者即可通过构造恶意输入，让反序列化产生非预期的对象，在此过程中执行构造的任意代码。

#### Kryo

Kryo 是一个高性能的序列化/反序列化工具，由于其变长存储特性并使用了字节码生成机制，拥有较高的运行速度和较小的字节码体积。

1.  Kryo 不是线程安全的，所以使用 `ThreadLocal` 来确保每个线程都有自己的 `Kryo` 实例
2. 使用kryo.writeObject(output, obj) 和kryo.readObject(input, clazz) 分别实现序列化和反序列化
3. Output 或 Input 对象在退出 try 块后会自动关闭，而在关闭之前，需要将 Kryo 实例从 ThreadLocal 中移除（kryoThreadLocal.remove()），防止内存泄漏

### 代理模式

#### 定义

使用代理对象来代替对真实对象的访问，这样就可以在**不修改原目标对象**的前提下，提供额外的功能操作，扩展目标对象的功能。

**主要作用：**

1. 提供一种间接访问方式，以便于控制对真实对象的访问。
2. 扩展目标对象的功能，比如说在目标对象的某个方法执行前后你可以增加一些自定义的操作（重写invoke时添加代码）。

使用场景：

|                     |                                                              |
| :------------------ | :----------------------------------------------------------- |
| AOP（面向切面编程） | Spring的AOP功能也是基于代理模式实现的。通过定义切点和切面，代理对象可以在目标对象的方法执行前后插入额外的横切逻辑，如日志记录、性能监控、安全验证等。 |
| 事务管理            | Spring的事务管理功能通常使用代理模式来实现。通过在业务方法前后添加事务管理的逻辑，代理对象可以控制事务的开始、提交或回滚，并提供了对事务的管理和控制。 |

#### 静态代理

静态代理中，对目标对象的每个方法的增强都是手动完成的，比如接口一旦新增加方法，目标对象和代理对象都要进行修改，需要对每个目标类都单独写一个代理类。从 JVM 层面来说， 静态代理在**编译时**就将接口、实现类、代理类这些都变成了一个个实际的 class 文件。

**实现步骤：**

1. 定义一个接口及其实现类；
2. 创建一个代理类同样实现这个接口
3. 将目标对象注入进代理类，然后在代理类的对应方法调用目标类中的对应方法。

#### 动态代理

动态代理不需要针对每个目标类都单独创建一个代理类，并且也不需要必须实现接口，可以直接代理实现类。从 JVM 角度来说，动态代理是在运行时动态生成类字节码，并加载到 JVM 中的。

##### **JDK 动态代理**

1. **介绍**

在 Java 动态代理机制中 `InvocationHandler` 接口和 `Proxy` 类是核心。

`Proxy` 类中使用频率最高的方法是：`newProxyInstance()` ，这个方法主要用来生成一个代理对象。

```java
    public static Object newProxyInstance(ClassLoader loader, //接口的类加载器
                                          Class<?>[] interfaces, //接口
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        ......
    }
```

因此，还需要实现InvocationHandler 来自定义处理逻辑。 当我们的动态代理对象调用一个方法时候，这个方法的调用就会被转发到实现InvocationHandler 接口类的 `invoke()`来调用。也就是说：**通过**Proxy **类的** newProxyInstance() **创建的代理对象在调用方法的时候，实际会调用到实现**InvocationHandler 接口的类的 `invoke()`**方法。**

2. **使用步骤**
   1. 定义一个接口及其实现类；
   2. 实现 InvocationHandler 并重写`invoke`方法，该方法在代理对象调用方法时**被触发**，在 `invoke` 方法中我们会调用原生方法（被代理类的方法）并自定义一些处理逻辑；
   3. 通过 `Proxy.newProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler h)` 方法创建代理对象；

- **整体实例：**

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

// 定义接口
interface UserService {
    void addUser(String username);
}

// 实现接口的具体类
class UserServiceImpl implements UserService {
    public void addUser(String username) {
        System.out.println("添加用户：" + username);
    }
}

// 实现InvocationHandler接口
class MyInvocationHandler implements InvocationHandler {
    private Object target;

    public MyInvocationHandler(Object target) {
        this.target = target;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("动态代理前置操作");
        Object result = method.invoke(target, args);
        System.out.println("动态代理后置操作");
        return result;
    }
}

public class DynamicProxyExample {
    public static void main(String[] args) {
        // 创建目标对象
        UserService userService = new UserServiceImpl();

        // 创建InvocationHandler实例
        MyInvocationHandler handler = new MyInvocationHandler(userService);

        // 创建动态代理对象
        UserService proxy = (UserService) Proxy.newProxyInstance(
                userService.getClass().getClassLoader(),
                userService.getClass().getInterfaces(),
                handler
        );

        // 通过代理对象调用方法
        proxy.addUser("Alice");
    }
}
```

3. **缺陷**

JDK 动态代理有一个最致命的问题是其只能代理**实现了接口的类**，可以用 CGLIB 动态代理机制来避免。例如，在Spring 中的 AOP 模块中：如果目标对象实现了接口，则默认采用 JDK 动态代理，否则采用 CGLIB 动态代理，JDK 动态代理的效率更高。

##### **CGLIB 动态代理**

1. **介绍**

在 CGLIB 动态代理机制中 `MethodInterceptor` 接口和 `Enhancer` 类是核心。

需要自定义 `MethodInterceptor` 并重写 `intercept` 方法，`intercept` 用于拦截增强被代理类的方法。可以通过 `Enhancer`类来动态获取被代理类，当代理类调用方法的时候，实际调用的是 `MethodInterceptor` 中的 `intercept` 方法。

```java
public interface MethodInterceptor extends Callback{
    // 拦截被代理类中的方法
    public Object intercept(Object obj, java.lang.reflect.Method method, Object[] args,
                               MethodProxy proxy) throws Throwable;
} // obj :被代理的对象（需要增强的对象）   method :被拦截的方法（需要增强的方法）  
  // args :方法入参     methodProxy :用于调用原始方法
```

2. **使用步骤**

   1. 定义一个类，添加依赖 cglib；

   2. 实现 `MethodInterceptor` 并重写 `intercept` 方法，`intercept` 用于拦截增强被代理类的方法，和 JDK 动态代理中的 `invoke` 方法类似；

   3. 通过 `Enhancer` 类的 `create()`创建代理类；

#### 对比

**静态代理和动态代理的对比：**

1. **灵活性** ：动态代理更加灵活，不需要必须实现接口，可以直接代理实现类，并且可以不需要针对每个目标类都创建一个代理类。另外，静态代理中，接口一旦新增加方法，目标对象和代理对象都要进行修改，这是非常麻烦的！
2. **JVM 层面** ：静态代理在编译时就将接口、实现类、代理类这些都变成了一个个实际的 class 文件。而动态代理是在运行时动态生成类字节码，并加载到 JVM 中的。

### 其他

#### 创造者模式

##### 定义

建造者模式可以使得对象的构建过程更加灵活，可以避免在对象的构造函数中传入大量的参数，并且可以提高代码的可读性和可维护性。通过链式调用的方式，可以直观地设置对象的属性，并且可以选择性地设置属性，而不需要考虑参数的顺序。

```java
RpcRequest rpcRequest = RpcRequest.builder()
  .interfaceName("interface")
  .methodName("hello").build();
```

##### 实现

1. **手写**

在目标类中定义一个静态内部类作为建造者类，建造者类中包含与目标类中相同的属性，并提供设置属性值的方法。建造者类还提供一个 `build()` 方法来构建目标类的实例，并且通常会在 `build()` 方法中进行参数的校验和初始化操作。

2. **注解**

Springboot中，只需要添加@Builder注解即可实现
