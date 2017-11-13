### Netty是什么  
Netty是一个使用Java语言编写的提供异步的、事件驱动的网络应用程序的开源框架，用以快速开发高性能、高可靠性的服务器和客户端网络应用程序。它封装了网络编程的复杂性。Netty不只是一个接口和类的集合，它还定义了一种架构模型以及一套丰富的设计模式。简单的将，Netty就是一个jar包，用于快速开发高性能的网络应用程序。  

### Netty特性  
* **性能**：统一的API，支持多种传输类型，简单而强大的线程模型，真正的无连接数据报套接字支持，链接逻辑组件以支持复用。
* **易于使用**：详实的Javadoc和大量的示例集。
* **性能**：拥有比Java的核心API更高的吞吐量以及更低的延迟，得益于池化和复用，拥有更低的资源消耗，最少的内存复制。
* **健壮性**：不会因为慢速、快速或者超载的连接而导致OutOfMemoryError，消除在高速网络中NIO应用程序常见的不公平读/写比率。
* **安全性**：完整的SSL/TLS以及StartTLS支持，可用于受限环境下，如Applet和OSGI。
* **社区驱动**：发布快速而且频繁。  

### Netty核心组件  
为了理解理解Netty是如何工作的，对Netty的内部设计有一个认识是非常必要的。对于Netty，有几个核心的组件必须掌握。如下：  
* AbstractBootstrap
* EventLoop
* EventLoopGroup
* Channel
* SocketChannel
* ChannelInitializer
* ChannelFuture
* ChannelPipeline
* ChannelHandler
* ChannelHandlerContext  

下图是几个核心组件的关系图：  
![image](/Netty/Images/netty-components.png)  

* **AbstractBootstrap**  
    在Netty中AbstractBootstrap类负责更方便的引导Netty中的Channel，引导包括启动线程，获取socket连接等。它支持链式调用以支持更方便进行配置AbstractBootstrap。通俗的说该组件是进行服务器配置和启动的类。至少，它可以将服务器绑定到它要监听请求的端口上。AbstractBootstrap类是一个抽象类，它有两个子类Bootstrap和ServerBootstrap，其中Bootstrap类是用来引导客户端程序的，ServerBootstrap用来引导服务器端程序。

* **EventLoopGroup**  
    EventLoopGroup是EventLoop的一个组。多个EventLoop可以通过EventLoopGroup组织在一起。以这种方式EventLoop可以像线程一样共享资源。

* **EventLoop**  
    EventLoop是一个持续监听新事件的程序，比如：从Socket写入的数据(从SocketChannel实例)。当一个事件出现，事件会传递到一个适当的事件处理器上，比如一个ChannelHandler实例。EventLoop用于处理连接的生命周期中所发生的事件。

* **Channel**  
    相当于Socket。基本的I/O操作(bind()、connect()、read()、write())，依赖于底层网络传输提供的原语。Netty的Channel接口所提供的API，大大降低了直接使用Socket类的复杂性。

* **SocketChannel**  
    Netty的SocketChannel代表了通过网络到另一台计算器的一个TCP连接。无论你使用Netty作为客户端还是服务器端，计算机之间的数据交换都是通过代表了计算机之间TCP连接的SocketChannel进行传递的。  
    一个SocketChannel通过一个EventLoop进行管理，并且总是通过一个相同的EventLoop。因为一个EventLoop总是被相同的线程执行，SocketChannel实例也仅仅只能被相同的线程所访问。因此你不必担心从SocketChannel读取数据的同步性。

* **ChannelInitializer**  
    ChannelInitializer是特殊的ChannelHandler，当SocketChannel被创建时会被添加到SocketChannel中的ChannelPipeline中。ChannelInitializer之后会被调用用于初始化SocketChannel。在初始化SocketChannel之后，ChannelInitializer会从ChannelPipeline中将自己移除。

* **ChannelPipeline**  
    每一个SocketChannel中都有一个ChannelPipeline。ChannelPipeline中包含了一些ChannelHandler实例。当EventLoop从SocketChannel中读取数据时，数据会传递给ChannelPipeline中的第一个ChannelHandler。第一个ChannelHandler会处理数据并且可以选择将数据转发给ChannelPipeline中的下一个ChannelHandler，同样也是负责处理数据和选择将数据转发给下一个ChannelHandler等等。
    当写出数据到SocketChannel，写出数据在最终写入到SocketChannel时同样通过ChannelPipeline。
* **ChannelHandler**  
    ChannelHandler负责处理从SocketChannel中接受的数据。ChannelHandler同样也可以处理写出到SocketChannel的数据。它充当了所有处理入站和出站数据的应用程序逻辑的容器。通俗的说就是该组件实现了服务器对从客户端接受的数据的处理，即它的业务逻辑，在Netty应用程序中，所有的数据处理逻辑都包含在这些核心抽象的实现中。每个Netty服务器中至少要有一个ChannelHandler。

* **ChannelFuture**
    Netty中所有的I/O操作都是异步的。因为一个操作可能不会立刻返回，我们需要一个方法在稍后的时间来判定它的结果。因此，Netty提供了ChannelFuture，它的addListener()方法注册一个ChannelFutureListener，当操作完成时可以收到通知（无论成功与否）。

* **ChannelHandlerContext**
    ChannelHandlerContext代表了ChannelHandler和ChannelPipeline之间的关联，每当有ChannelHandler添加到ChannelPipeline中时，都会创建ChannelHandlerContext。ChannelHandlerContext的主要功能是管理它所关联的ChannelHandler和在同一个ChannelPipeline中的其他ChannelHandler之间的交互。

### Netty基本  







**参考链接**：  
http://www.linkedkeeper.com/detail/blog.action?bid=141  
http://tutorials.jenkov.com/netty/overview.html  
