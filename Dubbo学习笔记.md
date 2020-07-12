#### Dubbo基础篇

###### 初始dubbo

Dubbo是阿里巴巴开发的一个开源的高性能RPC调用框架，致力于提供高性能和透明化的RPC远程调用服务解决方案

Dubbo架构图：

![image-20200708145915527](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200708145915527.png)

节点角色说明：

| 节点角色  | 节点说明                                                   |
| --------- | ---------------------------------------------------------- |
| Provider  | 服务提供者。负责暴露提供的服务，并将服务注册到服务注册中心 |
| Consumer  | 服务消费者。通过RPC远程调用服务提供者提供的服务            |
| Registry  | 服务注册中心。负责服务注册与发现                           |
| Monitor   | 监控中心。统计服务的调用次数和调用时间                     |
| Container | 服务运行容器。                                             |

调用关系说明：

   0.start：container服务容器负责启动、加载、运行服务提供者

1. register：Provider服务提供者在启动时会将自己提供的服务注册到服务注册中心
2. subscribe：Consumer服务消费方在启动时会去服务注册中心订阅自己需要的服务的地址列表，然后服务注册中心异步把消费者需要的地址列表返回，服务消费者根据路由规则和负载均衡算法选择一个服务提供者进行调用
3. notify：当消费者订阅的服务发生变更时，会触发通知
4. invoke：具体调用
5. count：在内存中统计调用次数和调用时间，每隔一分钟将统计数据发送到监控中心

#### Dubbo高级篇

###### Dubbo分层架构概述

分层：每一层各司其职，专注于实现本层的业务逻辑；上层依赖下层提供的功能，下层的改变对上层透明；每一层都是可被替换的组件

Dubbo框架整体设计图：

![image-20200709115009851](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200709115009851.png)

分层说明：

- Service和Config为API接口层：用于让Dubbo使用方方便地发布服务和引用服务

- Proxy服务代理层：主要对消费端使用的接口进行代理，把本地服务透明的转为远程服务；对提供方的接口实现类进行代理，把实现类转换为Wrapper类(目的：减少反射调用)

- Registry服务注册中心层：主要功能是封装服务地址的注册与发现逻辑

- Cluster路由层：封装服务提供者的路由规则、负载均衡、集群容错的实现，并桥接服务中心

  负载均衡扩展接口LoadBalance对应的实现类有：RandomLoadBalance(随机)、RoundRobinLoadBalance(轮询)、LeastActiveLoadBalance(最小活跃数)、ConsistentHashLoadBalance(一致性Hash)等

  集群容错扩展接口Cluster对应的实现类有：FailoverCluster(失败重试)、FailbackCluster(失败自动恢复)、FailfastCluster(快速失败)、FailsafeCluster(失败安全)、ForkingCluster(并行调用)等
  
- Monitor监控层：用来统计调用的次数和消耗时长

  扩展接口为MonitorFactory，实现类为DubboMonitorFactory

  可以实现MonitorFactory扩展接口，实现自定义监控统计策略

- Protocol远程调用层：封装RPC调用逻辑

  扩展接口为Protocol，实现类有RegistryProtocol、DubboProtocol、InjvmProtocol

- Exchange信息交换层：封装请求响应模式，同步转异步

  扩展接口为Exchanger，实现类有HeaderExchanger等

- Transport网络传输层：将Mina和Netty抽象为统一接口

  扩展接口Channel，实现类有NettyChannel、MinaChannel

  扩展接口Transport，实现类有NettyTransport、MinaTransport等

  扩展接口Codec2，实现类有DubboCodec、ThriftCodec等

- Serialize数据序列化层：提供可复用的一些工具

  扩展接口Serialization，实现类有DubboSerialization、FastJsonSerialization、Hessian2Serialization、JavaSerialization等

  扩展接口ThreadPool，实现类有FixedThreadPool、CachedThreadPool、LimitedThreadPool等

###### 适配器类原理

一个扩展接口可以有多个实现类，具体选择哪一个实现类，就是适配器类要做的事情
以Dubbo提供的扩展接口Protocal为例：Dubbo会使用动态编译技术为Protocal生成适配器类Protocal$Adaptive的对象实例。当需要使用Protocol的实例时，实际是通过调用Protocal$Adaptive实例对象的export()方法并根据URL中的协议类型参数来获取具体的实现类

###### 动态编译原理

动态编译：在JVM运行过程中把源文件编译为字节码文件
Dubbo提供了一个扩展接口Compiler，对应的实现类有JavassistCompiler、JdkCompiler

```java
@SPI("javassist")
public interface Compiler {
    Class<?> compile(String var1, ClassLoader var2);
}
```

首先从如何使用动态编译生成扩展接口对应的适配器类入手，打开ExtensionLoader的createAdaptiveExtensionClass()，该方法将源文件动态编译成Class，然后通过Class.newInstance()生成对象实例

```java
private Class<?> createAdaptiveExtensionClass() {
    	//生成扩展接口对应的适配器类的源码，其返回结果是一个字符串
        String code = (new AdaptiveClassCodeGenerator(this.type, this.cachedDefaultName)).generate();
    	//选择类加载器
        ClassLoader classLoader = findClassLoader();
    	//得到compiler的一个具体实现类
        Compiler compiler = (Compiler)getExtensionLoader(Compiler.class).getAdaptiveExtension();
    	//编译器编译源码，生成适配器类的Class对象
        return compiler.compile(code, classLoader);
    }
```

小结：Dubbo为每个扩展接口生成对应的适配器类的源码，然后选择具体的动态编译器类对源码进行编译以生成适配器类的Class对象，然后就可以通过Class.newInstance()生成对象实例

###### Dubbo中的SPI

Dubbo中的SPI功能是从JDK的标准SPI演化而来的，所以我们先了解一下JDK中的标准SPI

SPI（Service Provider Interface）服务提供者接口，是JDK内置的一种服务发现机制。JDK中的SPI

![image-20200710164122011](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200710164122011.png)

java中如果要使用SPI，首先需要提供服务接口，然后再提供服务实现类，也就是面向接口编程。这样就可以通过SPI机制来定位具体的实现类

SPI遵循如下约定：

1. 当服务提供者提供了扩展接口的某一具体实现后，需要再META-INF/services目录下创建一个名为“接口全限定名”的文件，其内容为实现类的全限定类名
2. 接口实现类所在的jar包放在主程序的classpath路径中
3. 主程序通过java.util.ServiceLoader动态装载实现类。通过扫描META-INF/services目录下的文件得到类的全限定类名，然后根据全限定类名将类加载到JVM中
4. SPI的实现类必须携带一个无参构造方法

Dubbo提供的SPI的好处：

1. JDK标准SPI会一次性实例化扩展点的所有实现，那如果有些扩展点初始化很耗时但又没有用上这样就浪费了系统资源
2. JDK标准SPI中如果扩展点加载失败，用户不会收到友好的异常通知
3. 增强了对扩展点的IOC和AOP的支持，一个扩展点可以通过setter()注入其他扩展点，也可以使用Wrapper类进行功能增强

###### Dubbo框架的线程模型

Dubbo默认的底层网络通信是netty，服务提供方NettyServer使用两级线程池，其中EventLoopGroup(boss)主要用于接收客户端的连接请求，并将TCP三次握手操作交给EventLoopGroup(worker)来处理，boss和worker线程组称为I/O线程

如果服务提供者的逻辑处理能快速完成，且不会发起新的I/O请求，那么请求直接在I/O线程上处理，这样的好处是减少线程池调度与上下文切换的开销。但如果处理逻辑较慢，或者需要发起新的I/O请求，则I/O线程会将请求派发到新的线程池进行处理，否则I/O线程会被阻塞

Dubbo提供了下面几种线程模型，判断请求是被I/O线程处理还是被业务线程池处理

- all（AllDispatcher类）：所有消息都派发到业务线程池，这些消息包括请求、响应、连接事件、断开事件等，如下图所示：

  ![image-20200712142428161](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200712142428161.png)

- direct（DirectDispatcher类）：所有消息都在I/O线程上执行。如下图所示：

  ![image-20200712142644409](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200712142644409.png)

- message（MessageOnlyDispatcher类）：请求响应消息派发到业务线程池，其他消息如连接事件、断开事件、心跳事件等直接I/O线程上执行，如下图所示：

  ![image-20200712142907287](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200712142907287.png)

- execution（ExecutionDispatcher类）：请求消息派发到业务线程池，其他消息直接在I/O上执行，如下图所示：

  ![image-20200712143052636](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200712143052636.png)

- connection（ConnectionOrderedDispatcher类）：在I/O线程上将连接事件、断开事件放入队列，有序的逐个执行，其他消息派发到业务线程池处理，如下图所示：

  ![image-20200712143415518](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200712143415518.png)

###### Dubbo的线程池

Dubbo提供了线程池扩展接口ThreadPool，该扩展接口有如下一些实现：

- FixedThreadPool：创建一个固定个数线程的线程池

- LimitedThreadPool：创建一个线程池，池中的线程个数动态增加，但不会超过配置的最大值。

  注：空闲线程不会被回收，会一直存在

- EagerThreadPool：创建一个线程池，当所有核心线程都处于忙碌状态时，将创建新的线程来执行新任务

- CachedThreadPool：创建一个自适应线程池，当线程空闲1分钟时，线程会被回收；当有新请求到来时，会创建新线程

###### 服务降级策略

如果URL中包含mock字段，如果值以force开头，则说明设置了forck:return 降级策略，此策略会返回mock值，而不发起远程调用；如果其值以fail开发，则说明设置了fail:return降级策略，那么会先发起远程调用。如果远程调用成功，则返回远程调用结果，如果远程调用结果失败，则返回mock值

###### 集群容错

Dubbo提供了以下集群容错模式：

- Failover Cluster：失败重试

  当服务消费者调用服务提供者失败后，会自动切换到其他服务提供者进行重试，这通常用于读操作或者具有幂等的写操作。重试会带来更长的延迟，可以通过retries=2来设置重试次数

- Failfast Cluster：快速失败

  当服务消费者调用服务提供者失败后，立即报错。这种模式通常用于非幂等性的写操作

- Failsafe Cluster：安全失败

  当服务消费者调用服务出现异常时，直接忽略异常。这种模式通常用于写入审计日志等操作

- Failback Cluster：失败自动恢复

  当服务消费端调用服务出现异常后，在后台记录失败的请求，并按照一定的策略后期再进行重试。这种模式通常用于消息通知操作

- Forking Cluster：并行调用

  当消费者调用一个接口方法后，Dubbo Client会并行调用多个服务提供者，只要其中有一个成功即返回。这种模式通常用于实时性要求较高的读操作，但需要浪费更多服务资源，可通过forks="4"来设置最大并行数

- Broadcast Cluster：广播调用

  当消费者调用一个接口方法后，Dubbo Client会逐个调用所有服务提供者，任意一台服务器调用异常则标志这次调用失败。这种模式通常用于通知所有提供者更新缓存或日志等本地资源信息

###### 负载均衡策略

Dubbo提供了多种负载均衡策略，默认为random随机调用，提供的负载均衡策略如下：

- Random LoadBalance：随机策略。按照概率设置权重，比较均匀，且可以动态调节提供者的权重
- RoundRobin LoadBalance：轮询策略。按权重设置轮询比率。存在执行比较慢的服务提供者堆积请求的情况
- LeastActive LoadBalance：最少活跃调用数。如果每个提供者的活跃数相同，则随机选择一个。每个服务提供者维护着一个活跃数计数器，用来记录当前同时处理请求的个数，也就是并发处理任务的个数。这个值越小，说明提供者处理的速度越快或者当前机器的负载比较低，所以路由选择时就选择活跃度最低的提供者
- ConsistentHash LoadBalance：一致性Hash策略。一致性Hash可以保证相同参数的请求总是发给同一个提供者，且当某一台机器宕机时，原本发往这台机器的请求将基于虚拟节点平摊给其他提供者，这样不会引起大范围的变动

###### 隐式参数传递

Dubbo提供了隐式参数传递的功能，即服务调用方可以通过RpcContext.getContext().setAttachment()方法设置附加属性键值对，然后设置的键值对可在服务提供方方法内获取

###### Dubbo协议

![image-20200712200636999](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200712200636999.png)



![image-20200712200652276](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200712200652276.png)

header包含了16各字节的数，其中前两个字节为魔数，用来标识一个帧的开始，固定为0xdabb。魔数后面的一个字节是请求类型和序列化标记ID的组合结果，其中高四位是请求类型，低4位是序列化方式。

接下来的一个字节用于在响应报文里设置响应的结果码，其后一个字节是请求ID，最后4各字节是body内容的大小。