# 深入理解 iOS Rendering Process

![](ios-rendering-process/rendering.jpg)

## 前言

iOS 最早名为 iPhone OS，是 [Apple](https://www.apple.com/) 公司专门为其硬件设备开发的操作系统，最初于 2007 年随第一代 iPhone 推出，后扩展为支持 Apple 公司旗下的其他硬件设备，如 iPod、iPad 等。

作为一名 iOS Developer，相信大多数人都有写出过造成 iOS 设备卡顿的代码经历，相应的也有过想方设法优化卡顿代码的经验。

本文将从 OpenGL 的角度结合 Apple 官方给出的部分资料，介绍 iOS Rendering Process 的概念及其整个底层渲染管道的各个流程。

相信在理解了 iOS Rendering Process 的底层各个阶段之后，我们可以在平日的开发工作之中写出性能更高的代码，在解决帧率不足的显示卡顿问题时也可以多一些思路~

## 索引

- iOS Rendering Process 概念
- iOS Rendering 技术框架
- OpenGL 主要渲染步骤
- OpenGL Render Pipeline
- Core Animation Pipeline
- Commit Transaction
- Animation
- 全文总结
- 扩展阅读

## iOS Rendering Process 概念

iOS Rendering Process 译为 iOS 渲染流程，本文特指 iOS 设备从设置将要显示的图元数据到最终在设备屏幕成像的整个过程。

在开始剖析 iOS Rendering Process 之前，我们需要对 iOS 的渲染概念有一个基本的认知：

### 基于平铺的渲染

iOS 设备的屏幕分为 N * N 像素的图块，每个图块都适合于 [SoC](https://en.wikipedia.org/wiki/System_on_a_chip) 缓存，几何体在图块内被大量拆分，只有在所有几何体全部提交之后才可以进行光栅化（Rasterization）。

![](ios-rendering-process/tile_based_rendering.jpg)

> Note: 这里的光栅化指将屏幕上面被大量拆分出来的几何体渲染为像素点的过程。

![](ios-rendering-process/rasterization.jpg)

## iOS Rendering 技术框架

事实上 iOS 渲染相关的层级划分大概如下：

![](ios-rendering-process/ios_rendering_framework.png)

### UIKit

嘛~ 作为一名 iOS Developer 来说，应该对 UIKit 都不陌生，我们日常开发中使用的用户交互组件都来自于 UIKit Framework，我们通过设置 UIKit 组件的 Layout 以及 BackgroundColor 等属性来完成日常的界面绘画工作。

其实 UIKit Framework 自身并不具备在屏幕成像的能力，它主要负责对用户操作事件的响应，事件响应的传递大体是经过逐层的**视图树**遍历实现的。

> 那么我们日常写的 UIKit 组件为什么可以呈现在 iOS 设备的屏幕上呢？

### Core Animation

Core Animation 其实是一个令人误解的命名。你可能认为它只是用来做动画的，但实际上它是从一个叫做 **Layer Kit** 这么一个不怎么和动画有关的名字演变而来的，所以做动画仅仅是 Core Animation 特性的冰山一角。

Core Animation 本质上可以理解为是一个复合引擎，旨在尽可能快的组合屏幕上不同的显示内容。这些显示内容被分解成独立的图层，即 CALayer，CALayer 才是你所能在屏幕上看见的一切的基础。

其实很多同学都应该知道 CALayer，UIKit 中需要在屏幕呈现的组件内部都有一个对应的 CALayer，也就是所谓的 Backing Layer。正是因为一一对应，所以 CALayer 也是树形结构的，我们称之为**图层树**。

视图的职责就是**创建并管理**这个图层，以确保当子视图在层级关系中**添加或者被移除**的时候，**他们关联的图层**也**同样对应在层级关系树当中有相同的操作**。

> 但是为什么 iOS 要基于 UIView 和 CALayer 提供两个平行的层级关系呢？为什么不用一个简单的层级关系来处理所有事情呢？

原因在于要做**职责分离**，这样也能**避免很多重复代码**。在 iOS 和 Mac OS X 两个平台上，事件和用户交互有很多地方的不同，基于多点触控的用户界面和基于鼠标键盘的交互有着本质的区别，这就是为什么 iOS 有 UIKit 和 UIView，而 Mac OS X 有 AppKit 和 NSView 的原因。他们功能上很相似，但是在实现上有着显著的区别。

> Note: 实际上，这里并不是两个层级关系，而是**四个**，每一个都扮演不同的角色，除了**视图树**和**图层树**之外，还存在**呈现树**和**渲染树**。

### OpenGL ES & Core Graphics

#### OpenGL ES

[OpenGL ES](https://en.wikipedia.org/wiki/OpenGL_ES) 简称 GLES，即 OpenGL for Embedded Systems，是 OpenGL 的子集，通常面向**图形硬件加速处理单元（GPU）**渲染 2D 和 3D 计算机图形，例如视频游戏使用的计算机图形。

OpenGL ES 专为智能手机，平板电脑，视频游戏机和 PDA 等嵌入式系统而设计 。OpenGL ES 是“历史上应用最广泛的 3D 图形 API”。

#### Core Graphics

[Core Graphics](https://developer.apple.com/documentation/coregraphics?language=objc) Framework 基于 Quartz 高级绘图引擎。它提供了具有无与伦比的输出保真度的低级别轻量级 2D 渲染。您可以使用此框架来处理基于路径的绘图，转换，颜色管理，离屏渲染，图案，渐变和阴影，图像数据管理，图像创建和图像遮罩以及 PDF 文档创建，显示和分析。

> Note: 在 Mac OS X 中，Core Graphics 还包括用于处理显示硬件，低级用户输入事件和窗口系统的服务。

### Graphics Hardware

[Graphics Hardware](https://en.wikipedia.org/wiki/Graphics_hardware) 译为图形硬件，iOS 设备中也有自己的图形硬件设备，也就是我们经常提及的 GPU。

[图形处理单元（GPU）]()是一种专用电子电路，旨在快速操作和改变存储器，以加速在用于输出到显示设备的帧缓冲器中创建图像。GPU 被用于嵌入式系统，手机，个人电脑，工作站和游戏控制台。现代 GPU 在处理计算机图形和图像方面非常高效，并且 GPU 的高度并行结构使其在**大块数据并行处理的算法**中比通用 CPU 更有效。

## OpenGL 主要渲染步骤

[OpenGL](https://en.wikipedia.org/wiki/OpenGL) 全称 Open Graphics Library，译为开放图形库，是用于渲染 2D 和 3D 矢量图形的**跨语言，跨平台**的**应用程序编程接口（API）**。OpenGL 可以直接访问 GPU，以实现硬件加速渲染。

一个用来渲染图像的 OpenGL 程序主要可以大致分为以下几个步骤：

- 设置图元数据
- 着色器-shader 计算图元数据（位置·颜色·其他）
- 光栅化-rasterization 渲染为像素
- fragment shader，决定最终成像
- 其他操作（显示·隐藏·融合）

> Note: 其实还有一些非必要的步骤，与本文主题不相关，这里点到为止。

我们日常开发时使用 UIKit 布局视图控件，设置透明度等等都属于**设置图元数据**这步，这也是我们日常开发中可以影响 OpenGL 渲染的主要步骤。

## OpenGL Render Pipeline

如果有同学看过 WWDC 的一些演讲稿或者接触过一些 OpenGL 知识，应该对 Render Pipeline 这个专业术语并不陌生。

不过 Render Pipeline 实在是一个初次见面不太容易理解的词，它译为**渲染管道**，也有译为渲染管线的...

其实 Render Pipeline 指的是**从应用程序数据转换到最终渲染的图像之间的一系列数据处理过程**。

好比我们上文中提到的 OpenGL 主要渲染步骤一样，我们开发应用程序时在**设置图元数据**这步为视图控件的设定布局，背景颜色，透明度以及阴影等等数据。

下面以 OpenGL 4.5 的 Render Pipeline 为例介绍一下：

![](ios-rendering-process/opengl_rendering_pipeline.png)

这些图元数据流入 OpenGL 中，传入**顶点着色器（vetex shader）**，然后顶点着色器对其进行着色器内部的处理后流出。之后可能进入**细分着色阶段（tessellation shading stage）**，其中又有可能分为细分控制着色器和细分赋值着色器两部分处理，还可能会进入**几何着色阶段（geometry shading stage）**，数据从中传递。最后都会走**片元着色阶段（fragment shading stage）**。

> Note: 图元数据是以 copy 的形式流入 shader 的，shader 一般会以特殊的**类似全局变量的形式**接收数据。

OpenGL 在最终成像之前还会经历一个阶段名为**计算着色阶段（compute shaing stage）**，这个阶段 OpenGL 会计算最重要在屏幕中成像的像素位置以及颜色，如果在之前提交代码时用到了 CALayer 会引起 **blending** 的显示效果（例如 Shadow）或者视图颜色或内容图片的 alpha 通道开启，都将会加大这个阶段 OpenGL 的工作量。

## Core Animation Pipeline

上文说到了 iOS 设备之所以可以成像不是因为 UIKit 而是因为 LayerKit，即 Core Animation。

Core Animation 图层，即 CALayer 中包含一个属性 contents，我们可以通过给这个属性赋值来**控制 CALayer 成像的内容**。这个属性的类型定义为 id，在程序编译时不论我们给 contents 赋予任何类型的值，都是可以编译通过的。但实践中，**如果 contents 赋值类型不是 CGImage，那么你将会得到一个空白图层**。

> Note: 造成 contents 属性的奇怪表现的原因是 Mac OS X 的历史包袱，它之所以被定义为 id 类型是因为在 Mac OS X 中这个属性对 CGImage 和 NSImage 类型的值都起作用。但是在 iOS 中，如果你赋予一个 UIImage 属性的值，仅仅会得到一个空白图层。

说完 Core Animation 的 contents 属性，下面介绍一下 iOS 中 Core Animation Pipeline：

- 在 Application 中布局 UIKit 视图控件间接的关联 Core Animation 图层
- Core Animation 图层相关的数据提交到 iOS Render Server，即 OpenGL ES & Core Graphics
- Render Server 将与 GPU 通信把数据经过处理之后传递给 GPU
- GPU 调用 iOS 当前设备渲染相关的图形设备 Display

![](ios-rendering-process/core_animation_pipeline.png)

> Note: 由于 iOS 设备目前的显示屏最大支持 **60 FPS** 的刷新率，所以每个处理间隔为 16.67 ms。

可以看到从 Commit Transaction 之后我们的图元数据就将会在下一次 RunLoop 时被 Application 发送给底层的 Render Server，底层 Render Server 直接面向 GPU 经过一些列的数据处理将处理完毕的数据传递给 GPU，然后 GPU 负责渲染工作，根据当前 iOS 设备的屏幕计算图像**像素位置以及像素 alpha 通道混色计算**等等最终在当前 iOS 设备的显示屏中呈现图像。

> 嘛~ 由于 Core Animation Pipeline 中 Render Server 包含 OpenGL ES & Core Graphics，其中 OpenGL ES 的渲染可以参考上文 OpenGL Render Pipeline 理解。

## Commit Transaction

Core Animation Pipeline 的整个管线中 iOS 常规开发一般可以影响到的范围也就仅仅是在 Application 中布局 UIKit 视图控件间接的关联 Core Animation 图层这一级，即 **Commit Transaction 之前的一些操作**。

那么在 Commit Transaction 之前我们一般要做的事情有哪些？

- Layout，构建视图
- Display，绘制视图
- Prepare，额外的 Core Animation 工作
- Commit，打包图层并将它们发送到 Render Server

### Layout

在 Layout 阶段我们能做的是把 constraint 写的尽量高效，iOS 的 Layout Constraint 类似于 Android 的 Relative Layout。

> Note: Emmmmm... 据观察 iOS 的 Layout Constraint 在书写时应该尽量少的依赖于视图树中同层级的兄弟视图节点，它会拖慢整个视图树的 Layout 计算过程。

**这个阶段的 Layout 计算工作是在 CPU 完成的**，包括 `layoutSubviews` 方法的重载，`addSubview:` 方法填充子视图等

### Display

其实这里的 Display 仅仅是我们设置 iOS 设备要最终成像的图元数据而已，重载视图 `drawRect:` 方法可以自定义 UIView 的显示，其原理是在 `drawRect:` 方法内部绘制 bitmap。

> Note: 重载 `drawRect:` 方法绘制 bitmap 过程**使用 CPU 和 内存**。

所以重载 `drawRect:` 使用不当会造成 CPU 负载过重，App 内存飙升等问题。

### Prepare

这个步骤属于附加步骤，一般处理图像的解码 & 转换等操作。

### Commit

Commit 步骤指打包图层并将它们发送到 Render Server。

> Note: Commit 操作会**递归执行**，由于图层和视图一样是以树形结构存在的，当图层树过于复杂时 Commit 操作的开销也会非常大。

#### CATransaction

CATransaction 是 Core Animation 中用于将多个图层树操作分配到渲染树的**原子更新**中的机制，对图层树的每个修改都必须是事务的一部分。

CATransaction 类没有属性或者实例方法，并且也不能用 `+alloc` 和 `-init` 方法创建它，我们只能用类方法 `+begin` 和 `+commit` 分别来入栈或者出栈。

事实上任何可动画化的图层属性都会被添加到栈顶的事务，你可以通过 `+setAnimationDuration:` 方法设置当前事务的动画时间，或者通过 `+animationDuration` 方法来获取时长值（默认 0.25 秒）。

Core Animation 在每个 RunLoop 周期中自动开始一次新的事务，即使你不显式地使用 `[CATransaction begin]` 开始一次事务，在一个特定 RunLoop 循环中的任何属性的变化都会被收集起来，然后做一次 0.25 秒的动画（CALayer 隐式动画）。

> Note: CATransaction 支持**嵌套**。

## Animation

对于 App 用户交互体验提升最明显的工作莫过于使用动画了，那么 iOS 是如何处理动画的渲染过程的呢？

日常开发中如果不是特别复杂的动画我们一般会使用 UIView Animation 实现，iOS 将 UIView Animation 的处理过程分为以下三个阶段：

- 调用 `animateWithDuration:animations:` 方法
- 在 Animation Block 中进行 Layout，Display，Prepare，Commit
- Render Server 根据 Animation 逐帧渲染

![](ios-rendering-process/animation.png)

> Note: 原理是 `animateWithDuration:animations:` 内部使用了 CATransaction 来将整个 Animation Block 中的代码作为原子操作 commit 给了 RunLoop。

### 基于 CATransaction 实现链式动画

事实上大多数的动画交互都是有动画执行顺序的，尽管 UIView Animation 很强大，但是在写一些顺序动画时使用 UIView Animation 只能在 `+ (void)animateWithDuration:delay:options:animations:completion:` 方法的 completion block 中层级嵌套，写成一坨一坨 block 堆砌而成的代码，实在是难以阅读更别提后期维护了。

在得知 UIView Animation 使用了 CATransaction 时，我们不禁会想到这个 completion block 是不是也是基于 CATransaction 实现的呢？

Bingo！CATransaction 中有 `+completionBlock` 以及 `+setCompletionBlock:` 方法可以对应于 UIView Animation 的 completion block 的书写。

> Note: 我的一个开源库 [**LSAnimator - 可多链式动画库**](https://github.com/Lision/LSAnimator) 在**动画顺序链接时也用到了 CATransaction**。

## 全文总结

结合上下文不难梳理出一个 iOS **最基本的完整渲染经过（Rendering pass）**。

![](ios-rendering-process/rendering_pass.png)

### 性能检测思路

基于整篇文章的内容归纳一下我们在日常的开发工作中遇到性能问题时检测问题代码的思路：

| 问题 | 建议 | 检测工具 |
| :---: | :---: | :---: |
| 目标帧率 | 60 FPS | Core Animation instrument |
| CPU or GPU | 降低使用率节约能耗 | Time Profiler instrument |
| 不必要的 CPU 渲染 | GPU 渲染更理想，但要清楚 CPU 渲染在何时有意义 | Time Profiler instrument |
| 过多的 offscreen passes | 越少越好 | Core Animation instrument |
| 过多的 blending | 越少越好 | Core Animation instrument |
| 奇怪的图片格式或大小 | 避免实时转换或调整大小 | Core Animation instrument |
| 开销昂贵的视图或特效 | 理解当前方案的开销成本 | Xcode View Debugger |
| 想象不到的层次结构 | 了解实际的视图层次结构 | Xcode View Debugger |

文章写得比较用心（是我个人的原创文章，转载请注明 https://lision.me/），如果发现错误会优先在我的 个人博客 中更新。如果有任何问题欢迎在我的微博 @Lision 联系我~

希望我的文章可以为你带来价值~

## 扩展阅读

- [WWDC2014-Advanced Graphics and Animations for iOS Apps](https://developer.apple.com/videos/play/wwdc2014/419/)
- [iOS 保持界面流畅的技巧](https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)
- [iOS-Core-Animation-Advanced-Techniques](https://github.com/AttackOnDobby/iOS-Core-Animation-Advanced-Techniques)