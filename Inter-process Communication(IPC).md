# 进程间通信（IPC）

## 概览

Chromium具有多进程架构，这意味着我们有许多进程相互通信。我们的主要进程间通信是命名管道。在Linux和OS X上，我们使用socketpair（）。为每个渲染器进程分配命名管道，以便与浏览器进程通信。管道以异步模式使用，以确保两端都不会被阻塞等待另一端。

有关如何编写安全IPC端点的建议，请参阅[IPC的安全提示](http://www.chromium.org/Home/chromium-security/education/security-tips-for-ipc)。

#### IPC 在浏览器中

在浏览器中，与渲染器的通信在单独的I / O线程中完成。然后，必须使用ChannelProxy将进出视图的消息代理到主线程。此方案的优点是最常见和性能关键消息的资源请求（对于网页等），可以完全在I / O线程上处理，而不是阻塞在用户界面。这些是通过使用ChannelProxy :: MessageFilter完成的，它由RenderProcessHost插入到通道中。此过滤器在I / O线程中运行，拦截资源请求消息，并将它们直接转发到资源调度程序主机。有关资源加载的详细信息，请参阅[多进程资源加载](http://dev.chromium.org/developers/design-documents/multi-process-resource-loading)。

#### IPC在渲染器中

每个渲染器还有一个管理通信的线程（在这种情况下，主线程），渲染和大多数处理发生在另一个线程上（参见多进程架构中的图表）。大多数消息通过主渲染器线程从浏览器发送到WebKit线程，反之亦然。这个额外的线程是支持同步渲染器到浏览器的消息（参见下面的“同步消息”）。



## 消息

#### 消息类型

我们有两种主要类型的消息：“routed”和“control”。Control消息由创建管道的类处理。有时，该类允许其他人通过拥有MessageRouter对象接收消息。其他侦听器可以使用该对象注册并接收使用其唯一（每个管道）ID发送的“路由”消息。

例如，在渲染时，Control消息不是特定于给定视图的，将由renderProcess（渲染器）或renderProcessHost（浏览器）处理。对资源或修改剪贴板的请求不是视图特定的，所以是Control消息。routed消息的一个示例是要求视图绘制区域的消息。

历史上，routed消息适用于将消息发送到特定的RenderViewHost。但是，从技术上讲，任何类都可以通过使用RenderProcessHost :: GetNextRoutingID并使用RenderProcessHost :: AddRoute注册自己来接收Routed消息。目前，RenderViewHost和RenderFrameHost实例都有自己的路由ID。

消息从浏览器发送到渲染器，还是从渲染器发送到浏览器，与消息类型是无关的。与从浏览器发送到渲染器的文档框架相关的消息称为Frame消息，因为它们被发送到RenderFrame。同样，从渲染器发送到浏览器的消息称为FrameHost消息，因为它们被发送到RenderFrameHost。您会注意到frame_messages.h中定义的消息是两个部分，一个用于Frame，另一个用于FrameHost消息。

插件也有单独的进程。与渲染消息一样，有PluginProcess消息（从浏览器发送到插件进程）和PluginProcessHost消息（从插件进程发送到浏览器）。这些消息都在plugin_process_messages.h中定义。自动化消息（用于从UI测试控制浏览器）以类似的方式完成。

同一组织适用于浏览器和渲染器之间交换的其他消息组，如在view_messages.h中定义的renderview和renderview之间交换的view和viewhost标记消息。



#### 声明消息

特殊宏用于声明消息。要声明从渲染器到浏览器的路由消息（例如，特定于Frame的FrameHost消息），其中包含URL和整数作为参数，请写入：

```c++
IPC_MESSAGE_ROUTED2(FrameHostMsg_MyMessage, GURL, int)
```

要声明从浏览器到渲染器的控制消息（例如，不是特定于一个Frame的Frame消息），不包含任何参数，请写入：

```c++
IPC_MESSAGE_CONTROL0(FrameMsg_MyMessage)
```



##### Pickling values

使用ParamTraits模板将参数序列化并反序列化为消息体。此模板的专门化是为ipc_message_utils.h中最常见的类型提供的。如果你要定义自己的类型，那你必须要为其定义自己的ParamTraits模板。

有时，消息中包含太多值，无法合理地放入消息中。在这种情况下，我们定义一个单独的结构来保存这些值。例如，对于FrameMsg_Navigate消息，在navigation_params.h中定义了CommonNavigationParams结构。 frame_messages.h使用IPC_STRUCT_TRAITS宏系列定义结构的ParamTraits特化。



#### 发送消息

您通过“channels”发送消息（见下文）。在浏览器中，RenderProcessHost包含用于将消息从浏览器的UI线程发送到渲染器的通道。RenderWidgetHost（RenderViewHost的基类）提供了一个方便使用的Send函数。

消息由指针发送，并在调度后由IPC层删除。因此，一旦找到合适的发送函数，只需使用新消息调用它：

```c++
Send(new ViewMsg_StopFinding(routing_id_));
```

请注意，您必须指定路由ID，以便将消息路由到接收端的正确View / ViewHost。RenderWidgetHost（RenderViewHost的基类）和RenderWidget（RenderView的基类）都有可以使用的GetRoutingID（）成员函数。



#### 处理消息

消息是通过实现ipc:：listener接口来处理的，该接口是onMessageReceived最重要的函数。我们有多种宏来简化此函数中的消息处理，这一点最好用示例来说明：

```c++
MyClass::OnMessageReceived(const IPC::Message& message) { 
    IPC_BEGIN_MESSAGE_MAP(MyClass, message)    
    // Will call OnMyMessage with the message. The parameters of the message will be unpacked for you.    
    IPC_MESSAGE_HANDLER(ViewHostMsg_MyMessage, OnMyMessage)      
    ...    
    IPC_MESSAGE_UNHANDLED_ERROR()  // This will throw an exception for unhandled messages.  
    IPC_END_MESSAGE_MAP()
}
    
    // This function will be called with the parameters extracted from the ViewHostMsg_MyMessage message.
MyClass::OnMyMessage(const GURL& url, int something) {
    ...
}
```



您也可以使用IPC_DEFINE_MESSAGE_MAP来实现您的函数定义。在这种情况下，不要指定消息变量名，它将在给定的类上声明onMessageReceived函数并实现其内部结构。

其他宏：

- IPC_MESSAGE_FORWARD： 这与IPC_MESSAGE_HANDLER相同，但您可以指定自己的类来发送消息，而不是将其发送到当前类。

  ```c++
  IPC_MESSAGE_FORWARD(ViewHostMsg_MyMessage, some_object_pointer, SomeObject::OnMyMessage)
  ```

- IPC_MESSAGE_HANDLER_GENERIC：这允许您编写自己的代码，但您必须自己从消息中解压缩参数：

  ```c++
  IPC_MESSAGE_HANDLER_GENERIC(ViewHostMsg_MyMessage, printf("Hello, world, I got the message."))
  ```



#### 安全考虑

IPC中的安全漏洞可能会产生[令人讨厌的后果](https://blog.chromium.org/2012/05/tale-of-two-pwnies-part-1.html)（文件被盗，沙箱逃逸，远程代码执行）。查看我们的[IPC文档安全性](http://www.chromium.org/Home/chromium-security/education/security-tips-for-ipc)，获取有关如何避免常见陷阱的提示。



## Channels （通道）

IPC :: Channel（在ipc / ipc_channel.h中定义）定义了跨管道通信的方法。ipc:：syncchannel提供了额外的功能，用于同步等待对某些消息的响应（渲染器进程按照下面“同步消息”部分中的描述使用此功能，但浏览器进程从不这样做）。

通道不是线程安全的。我们经常希望使用另一个线程上的通道发送消息。例如，当UI线程想要发送消息时，它必须通过I / O线程。为此，我们使用IPC :: ChannelProxy。它有一个与常规通道对象类似的API，但是它将消息代理到另一个线程以发送它们，并在接收消息时将其代理回原始线程。它允许您的对象（通常在UI线程上）在通道线程（通常是I / O线程）上安装IPC :: ChannelProxy :: Listener，以过滤掉代理的某些消息。我们将此用于资源请求和可以直接在I / O线程上处理的其他请求。 RenderProcessHost安装执行此过滤的RenderMessageFilter对象。



##  同步消息

从渲染器的角度来看，某些消息应该是同步的。这种情况主要发生在我们有一个WebKit调用它应该返回的东西时，但我们必须在浏览器中执行此操作。此类消息的示例包括拼写检查和获取JavaScript的cookie。浏览器到渲染器的IPC禁止同步，以防止在可能不稳定的渲染器上阻塞用户界面。

危险：不要处理UI线程中的任何同步消息！您必须只在I/O线程中处理它们。否则，应用程序可能会死锁，因为插件需要从UI线程进行同步绘制，当渲染器等待来自浏览器的同步消息时，这些将被阻止。

#### 声明同步消息

使用IPC_SYNC_MESSAGE_ *宏声明同步消息。这些宏具有输入和返回参数（非同步消息缺少返回参数的概念）。对于带有两个输入参数并返回一个参数的控制函数，您可以将2_1附加到宏名称以获取：

```c++
IPC_SYNC_MESSAGE_CONTROL2_1(SomeMessage,  // Message name                                                            GURL, //input_param1  
                           int, //input_param2  
                           std::string); //result
```

同样，您也可以将消息路由到视图，在这种情况下，您可以将“control”替换为“routed”，以获取IPC_SYNC_MESSAGE_ROUTED2_1。您还可以有0个输入或返回参数。当渲染器必须等待浏览器执行某些操作，但不需要任何结果时，将使用无返回参数。我们将它用于某些打印和剪贴板操作。

#### 发出同步消息

当WebKit线程发出同步IPC请求时，请求对象（从IPC::SyncMessage派生）通过IPC::SyncChannel对象（同一个对象也用于发送所有异步消息）发送到渲染器上的主线程。SyncChannel在收到同步消息时将阻塞调用线程，并且只在收到回复时才解除阻塞。

当WebKit线程正在等待同步回复时，主线程仍然从浏览器进程接收消息。这些消息将被添加到WebKit线程的队列中，以便在唤醒时进行处理。收到同步消息回复后，线程将被取消阻止。请注意，这意味着可以无序处理同步消息回复。

同步消息的发送方式与普通消息的发送方式相同，输出参数将提供给构造函数。例如：

```c++
const GURL input_param("http://www.google.com/");
std::string result;
content::RenderThread::Get()->Send(new MyMessage(input_param,                                                          result));
printf("The result is %s\n", result.c_str());
```



#### 处理同步消息

同步消息和异步消息使用相同的IPC_MESSAGE_HANDLER等宏来分派消息。消息的处理函数将具有与消息构造函数相同的签名，并且函数只将输出写入输出参数。对于上面的消息，您将添加

```c++
IPC_MESSAGE_HANDLER(MyMessage, OnMyMessage)
```

对于OnMessageReceived函数，写：

```c++
void RenderProcessHost::OnMyMessage(GURL input_param, std::string* result) {
*result = input_param.spec() + " is not available";
}
```



#### 将消息类型转换为消息名称

如果发生崩溃，并且您有消息类型，那么可以将其转换为消息名称。消息类型为32位值，高16位为类，低16位为id。该类基于ipc / ipc_message_start.h中的枚举，id基于定义消息的文件中的行号。这意味着您需要获取Chromium的确切修订版才能准确获取消息名称。

554011中的示例在Chromium修订版ad0950c1ac32ef02b0b0133ebac2a0fa4771cf20处为0x1c0098。这是类0x1c，它是与ChildProcessMsgStart匹配的第40行。

ChildProcessMsgStart消息位于content / common / child_process_messages.h中，IPC将位于0x98行或第152行，即ChildProcessHostMsg_ChildHistogramData。

如果您正在处理由content :: RenderProcessHostImpl :: OnBadMessageReceived引起的崩溃，此技术特别有用