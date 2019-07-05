# Chromium多进程架构



本文档介绍了chromium的高级架构。



## Problem

构建永不崩溃或挂起的渲染引擎几乎是不可能的。构建一个非常安全的渲染引擎几乎也是不可能的。

在某些方面，2006年左右的Web浏览器状态类似于过去的单用户、协作式多任务操作系统。由于在这样的操作系统中行为不端的应用程序可能会占用整个系统，因此Web浏览器中的行为不当网页也可能会崩溃。所有的这些，只需要一个浏览器或者插件错误，就会关闭整个浏览器和所有的当前运行的选项卡。



## 架构概览

我们对浏览器选项卡使用单独的进程来保护整个应用程序免受渲染引擎中的错误和故障的影响。我们还限制了每个渲染引擎进程对其他进程以及系统其余部分的访问。在某些方面，这为Web浏览带来了内存保护和访问控制带给操作系统的好处。

我们将主进程作为“浏览器进程(browser process)” 或者 “浏览器(browser)”，运行UI，管理选项卡和插件进程。同样的，特定于选项卡的进程成为“渲染进程(render process)”或者“渲染器(renderers)”。渲染器使用Blink开源布局引擎来解释和布局HTML。

![arch](photo\arch.png)



### 管理渲染进程

每个渲染进程都有一个全局RenderProcess对象，该对象管理与父浏览器进程的通信并维护全局状态。浏览器为每个渲染进程维护一个相应的RenderProcessHost，它管理渲染器的浏览器状态和通信。浏览器和渲染器使用[Chromium的IPC系统](http://www.chromium.org/developers/design-documents/inter-process-communication)进行通信。



### 管理视图

每个渲染进程都有一个或多个RenderView对象，由RenderProcess管理，对应与内容选项卡。相应的RenderProcessHost维护与渲染器中每个视图对应的RenderViewHost。每个视图都有一个视图ID，用于区分同一渲染器中的多个视图。这些ID在一个渲染器中是唯一的，但不在浏览器中，因此识别视图需要renderProcessHost和视图ID。从浏览器到特定内容选项卡的通信时通过这些RenderViewHost对象完成的，这些对象知道如何通过RenderProcessHost将消息发送到RenderProcess和RenderView。



## 组件和接口

在渲染进程中：

- RenderProcess在browser进程中使用对应的RenderProcessHost处理IPC。每个渲染过程只有一个RenderProcess对象。这就是所有浏览器↔渲染器通信的发生方式。
- RenderView对象在Browser进程（通过RenderProcess）和我们的WebKit嵌入层与其使用对应的RenderViewHost进行通信。此对象表示选项卡或弹出窗口中的一个网页的内容。

在浏览进程中：

- Browser对象表示顶级浏览器窗口。
- RenderProcessHost对象表示单个浏览器↔渲染器IPC连接的浏览器端。每个渲染过程在浏览器进程中都有一个RenderProcessHost。
- RenderViewHost对象封装了与远程RenderView的通信，RenderWidgetHost在浏览器中处理RenderWidget的输入和绘制。

有关此嵌入如何工作的更多详细信息，请参阅[Chromium如何显示网页设计文档](http://www.chromium.org/developers/design-documents/displaying-a-web-page-in-chrome)。



## 共享渲染进程

通常，每个新窗口或选项卡都会在新进程中打开，浏览器将生成一个新进程并指示它创建一个RenderView。

有时需要或希望在标签或窗口之间共享渲染过程。Web应用程序打开一个它希望同步通信的新窗口，例如，在JavaScript中使用window.open。在这种情况下，当我们创建一个新窗口或选项卡时，我们需要重用打开窗口的过程。如果进程总数太大，或者用户已经打开导航到该域的进程，我们还有策略为现有进程分配新选项卡。这些策略介绍在[进程模型文档](http://www.chromium.org/developers/design-documents/process-models)。



## 检测崩溃或行为不端的渲染器

每个与浏览器进程的IPC连接都会监视进程句柄。如果发出这些句柄的信号，则渲染过程已崩溃，并且向标签通知崩溃。现在，我们显示一个悲伤标签到屏幕上，通知用户浏览器已崩溃。可以通过重新加载按钮或开始新导航来重新加载页面。发生这种情况时，我们注意到没有进程并创建一个进程。



## 沙箱渲染器

鉴于渲染器在单独的进程中运行，我们有机会通过[沙箱](https://chromium.googlesource.com/chromium/src/+/master/docs/design/sandbox.md)限制其对系统资源的访问。例如，我们可以确保渲染器仅通过其父浏览器进程访问网络。同样，我们可以使用主机操作系统的内置权限限制其对文件系统的访问。

除了限制渲染器对文件系统和网络的访问外，我们还可以限制其对用户显示和相关对象的访问。我们在单独的windows“桌面”上运行每个渲染过程，这对用户是不可见的。这可以防止受损的渲染器打开新窗口或捕获击键。



## 内存回收

鉴于渲染器在单独的进程中进行，则将隐藏选项卡视为较低优先级变得简单明了。通常，Windows上最小化进程会将其内存自动放入“可用内存”池中。在内存不足的情况下，Windows将在更换优先级较高的内存之前将此内存交换到磁盘，从而帮助保持用户可见程序的响应速度。我们可以将相同的原则应用于隐藏的标签。当渲染进程没有顶级选项卡时，我们可以释放该进程的“工作集”大小作为系统的提示，以便在必要时首先将该内存交换到磁盘。因为我们发现当用户在两个标签之间切换时减小工作集大小也会降低标签切换功能，我们会逐渐释放这个内存。这意味着如果用户切换回最近使用的选项卡，则该选项卡额内存比最近使用的选项卡更容易被分页。拥有足够内存来运行所有程序的用户根本不会注意到这个过程：Windows只会在需要时才会回收这些数据，因此当内存充足时，性能不会受到影响。

这有助于我们在低内存情况下获得更佳的内存占用。与很少使用的背景标签相关联的内存可以完全换出，而前景标签的数据可以完全加载到内存中。相比之下，单进程浏览器将所有选项卡的数据随机分布在其内存中，并且不可能如此干净地分离已使用和未使用的数据，从而浪费内存和性能。



## 插件和扩展

Firefox风格的NPAPI插件在自己的进程中运行，与渲染器分开。这在[Plugin Architecture](http://www.chromium.org/developers/design-documents/plugin-architecture)中有详细描述。

[Site Isolation](https://www.chromium.org/developers/design-documents/site-isolation)项目旨在提供渲染器之间的更多隔离，此项目的早期可交付成果包括在隔离的进程中运行Chrome的HTML / JavaScript内容扩展。



英文原文链接：http://www.chromium.org/developers/design-documents/multi-process-architecture
