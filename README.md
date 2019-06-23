![MacDown Screenshot](https://github.com/whsgzcy/chicken/blob/master/imgae/android_start.png)

## 1、什么是进程 什么是线程

这个问题我想工作五六年的人都不一定能答出来，但刚出来工作的人一定和百度上搜索的答得一致，是对的，但五年之后，当时的答案真的是对的吗？

因为我之前对这个的理解是错的，当我去理解Android源码的时候，逻辑总是对不上，所以开头以这张图片做标题，这个理解了，一些诸如内存泄露等问题的本质就可以迎刃而解了。

那被问到什么是进程什么是线程时，下述回答，至少不会失水准，后面更进一步的答案也需要我们不断的去思考

**线程共享进程的内存单元**

**解读**

在解读前，我们需要有以下准备

1、 Android是一个组件化的设计，这里的组件可不是我们的UI组件，或者说Android是一个弱化进程的设计，所以 四大组件 Activity Service BroadCast ContentProvider 他们既不是进程也不是线程，他们是组件，例如BroadCast，它是怎么通信的，读完源码其实是很简单的，他们是在Java Fragmentwork 中被设计用来服务于Application，管理他们的是Java Fragmentwork中各个单例，所以，内存泄露的本质就是当前线程在当前组件回收前没有被销毁。

2、基础建设永不变，这话是我朋友说的，网上什么xxx框架，被人封装了一下，感觉很高大上，导致自己写的基础代码都没那么自信，其实都是一样的，最高明的做法是，既然需要用到某一个框架，那只把框架里面核心的代码拿过来用就可以了，不必要整体导包。

3、后续阐述的应用 可以理解为apk 这样感觉会更好一点

4、站在巨人的肩膀而不是膀胱

**进程和线程**

当某个应用组件启动且该应用没有运行其他任何组件时，Android 系统会使用单个执行线程为应用启动新的 Linux 进程。默认情况下，同一应用的所有组件在相同的进程和线程（称为“主”线程）中运行。 

```
这里，在相同的进程中运行 相同的线程 指的就是主线程，那么问题来了，组件是在哪个或者什么样的线程中运行？？？
```

如果某个应用组件启动且该应用已存在进程（因为存在该应用的其他组件），则该组件会在此进程内启动并使用相同的执行线程。 但是，您可以安排应用中的其他组件在单独的进程中运行，并为任何进程创建额外的线程。

**进程**

默认情况下，同一应用的所有组件均在相同的进程中运行，且大多数应用都不会改变这一点。 但是，如果您发现需要控制某个组件所属的进程，则可在清单文件中执行此操作。

各类组件元素的清单文件条目—<activity>、<service>、<receiver> 和 <provider>—均支持 android:process 属性，此属性可以指定该组件应在哪个进程运行。您可以设置此属性，使每个组件均在各自的进程中运行，或者使一些组件共享一个进程，而其他组件则不共享。 此外，您还可以设置 android:process，使不同应用的组件在相同的进程中运行，但前提是这些应用共享相同的 Linux 用户 ID 并使用相同的证书进行签署。

此外，<application> 元素还支持 android:process 属性，以设置适用于所有组件的默认值。

```
这里就是我们常常看到的 比如导入第三方sdk，它会让你在你的清单上配置固定的process ，这样的做法是为了保活，那么问题来了，这样两个进程怎么通信？java中进程与进程之间是不能通信的，多个应用可以共享一块内存单元，所有他们的通信方式还是老样子 共享内存+反射
```

如果内存不足，而其他为用户提供更紧急服务的进程又需要内存时，Android 可能会决定在某一时刻关闭某一进程。在被终止进程中运行的应用组件也会随之销毁。 当这些组件需要再次运行时，系统将为它们重启进程。

决定终止哪个进程时，Android 系统将权衡它们对用户的相对重要程度。例如，相对于托管可见 Activity 的进程而言，它更有可能关闭托管屏幕上不再可见的 Activity 的进程。 因此，是否终止某个进程的决定取决于该进程中所运行组件的状态。 下面，我们介绍决定终止进程所用的规则。

```
这里，涉及到进程保活
```

**进程生命周期**

Android 系统将尽量长时间地保持应用进程，但为了新建进程或运行更重要的进程，最终需要移除旧进程来回收内存。 为了确定保留或终止哪些进程，系统会根据进程中正在运行的组件以及这些组件的状态，将每个进程放入“重要性层次结构”中。 必要时，系统会首先消除重要性最低的进程，然后是重要性略逊的进程，依此类推，以回收系统资源。

重要性层次结构一共有 5 级。以下列表按照重要程度列出了各类进程（第一个进程最重要，将是最后一个被终止的进程）：

```
重要性层次结构 这块的源码可以去读一下，看一下是怎么判断的
```

**前台进程**

用户当前操作所必需的进程。如果一个进程满足以下任一条件，即视为前台进程：

托管用户正在交互的 Activity（已调用 Activity 的 onResume() 方法）

托管某个 Service，后者绑定到用户正在交互的 Activity

托管正在“前台”运行的 Service（服务已调用 startForeground()）

托管正执行一个生命周期回调的 Service（onCreate()、onStart() 或 onDestroy()）

托管正执行其 onReceive() 方法的 BroadcastReceiver

通常，在任意给定时间前台进程都为数不多。只有在内存不足以支持它们同时继续运行这一万不得已的情况下，系统才会终止它们。 此时，设备往往已达到内存分页状态，因此需要终止一些前台进程来确保用户界面正常响应。

**可见进程**

没有任何前台组件、但仍会影响用户在屏幕上所见内容的进程。 如果一个进程满足以下任一条件，即视为可见进程：

托管不在前台、但仍对用户可见的 Activity（已调用其 onPause() 方法）。例如，如果前台 Activity 启动了一个对话框，允许在其后显示上一 Activity，则有可能会发生这种情况。
托管绑定到可见（或前台）Activity 的 Service。
可见进程被视为是极其重要的进程，除非为了维持所有前台进程同时运行而必须终止，否则系统不会终止这些进程。

**服务进程**

正在运行已使用 startService() 方法启动的服务且不属于上述两个更高类别进程的进程。尽管服务进程与用户所见内容没有直接关联，但是它们通常在执行一些用户关心的操作（例如，在后台播放音乐或从网络下载数据）。因此，除非内存不足以维持所有前台进程和可见进程同时运行，否则系统会让服务进程保持运行状态。

**后台进程**

包含目前对用户不可见的 Activity 的进程（已调用 Activity 的 onStop() 方法）。这些进程对用户体验没有直接影响，系统可能随时终止它们，以回收内存供前台进程、可见进程或服务进程使用。 通常会有很多后台进程在运行，因此它们会保存在 LRU （最近最少使用）列表中，以确保包含用户最近查看的 Activity 的进程最后一个被终止。如果某个 Activity 正确实现了生命周期方法，并保存了其当前状态，则终止其进程不会对用户体验产生明显影响，因为当用户导航回该 Activity 时，Activity 会恢复其所有可见状态。 有关保存和恢复状态的信息，请参阅 Activity文档。

**空进程**

不含任何活动应用组件的进程。保留这种进程的的唯一目的是用作缓存，以缩短下次在其中运行组件所需的启动时间。 为使总体系统资源在进程缓存和底层内核缓存之间保持平衡，系统往往会终止这些进程。

根据进程中当前活动组件的重要程度，Android 会将进程评定为它可能达到的最高级别。例如，如果某进程托管着服务和可见 Activity，则会将此进程评定为可见进程，而不是服务进程。

此外，一个进程的级别可能会因其他进程对它的依赖而有所提高，即服务于另一进程的进程其级别永远不会低于其所服务的进程。 例如，如果进程 A 中的内容提供程序为进程 B 中的客户端提供服务，或者如果进程 A 中的服务绑定到进程 B 中的组件，则进程 A 始终被视为至少与进程 B 同样重要。

由于运行服务的进程其级别高于托管后台 Activity 的进程，因此启动长时间运行操作的 Activity 最好为该操作启动服务，而不是简单地创建工作线程，当操作有可能比 Activity 更加持久时尤要如此。例如，正在将图片上传到网站的 Activity 应该启动服务来执行上传，这样一来，即使用户退出 Activity，仍可在后台继续执行上传操作。使用服务可以保证，无论 Activity 发生什么情况，该操作至少具备“服务进程”优先级。 同理，广播接收器也应使用服务，而不是简单地将耗时冗长的操作放入线程中。

```
这里就是进程保活，所以有一种方式就是一个像素点保活，一个像素点其实就是一个Activity，至于前台，这里的说的什么xx进程也就是在xml中配置的
```

**线程**

应用启动时，系统会为应用创建一个名为“主线程”的执行线程。 此线程非常重要，因为它负责将事件分派给相应的用户界面小部件，其中包括绘图事件。 此外，它也是应用与 Android UI 工具包组件（来自 android.widget 和 android.view 软件包的组件）进行交互的线程。因此，主线程有时也称为 UI 线程。

```
所以主线程就是ui线程，可以这么理解，我之前就是这么认为的，但真的是这样吗？

AOSP7.1

services/core/java/com/android/server/UiThread.java

core/java/android/annotation/MainThread.java

Ordinarily, an app's UI thread is also the main thread. However, under special circumstances, an app's UI thread might not be its main thread; for more information, see Thread annotations.

@MainThread
@UiThread
@WorkerThread
@BinderThread
@AnyThread

https://developer.android.com/studio/write/annotations.html#thread-annotations

注：构建工具会将 @MainThread 和 @UiThread 注解视为可互换，因此，您可以从 @MainThread 方法调用 @UiThread 方法，反之亦然。不过，如果系统应用在不同线程上带有多个视图，UI 线程可与主线程不同。因此，您应使用 @UiThread 标注与应用的视图层次结构关联的方法，使用 @MainThread 仅标注与应用生命周期关联的方法。

这里我师父曾经和我说过，Rom 开发，状态机可不用在Application使用，有xxxHandler机制可以使用

知道有这回事就行了
```

系统不会为每个组件实例创建单独的线程。运行于同一进程的所有组件均在 UI 线程中实例化，并且对每个组件的系统调用均由该线程进行分派。 因此，响应系统回调的方法（例如，报告用户操作的 onKeyDown() 或生命周期回调方法）始终在进程的 UI 线程中运行。

例如，当用户触摸屏幕上的按钮时，应用的 UI 线程会将触摸事件分派给小部件，而小部件反过来又设置其按下状态，并将失效请求发布到事件队列中。 UI 线程从队列中取消该请求并通知小部件应该重绘自身。

在应用执行繁重的任务以响应用户交互时，除非正确实现应用，否则这种单线程模式可能会导致性能低下。 具体地讲，如果 UI 线程需要处理所有任务，则执行耗时很长的操作（例如，网络访问或数据库查询）将会阻塞整个 UI。 一旦线程被阻塞，将无法分派任何事件，包括绘图事件。 从用户的角度来看，应用显示为挂起。 更糟糕的是，如果 UI 线程被阻塞超过几秒钟时间（目前大约是 5 秒钟），用户就会看到一个让人厌烦的“应用无响应”(ANR) 对话框。如果引起用户不满，他们可能就会决定退出并卸载此应用。

此外，Android UI 工具包并非线程安全工具包。因此，您不得通过工作线程操纵 UI，而只能通过 UI 线程操纵用户界面。 因此，Android 的单线程模式必须遵守两条规则：

不要阻塞 UI 线程

不要在 UI 线程之外访问 Android UI 工具包

```
这些都是常识
```

**总结**

通过现象看本质，通过本质去处理现象，基础工程才是王道！

















## 参考

https://developer.android.com/guide/components/processes-and-threads.html