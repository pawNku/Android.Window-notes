# Android.Window-notes
Android之Window的一些理解和整理。结合了入门的‘’第一行代码‘’和进阶的‘’Android开发艺术探索‘’两本书和一个实例Demo~  

## 开发艺术探索之理解Window和WindowMananger
> Window是一个抽象类，具体实现是 PhoneWindow 。不管是 Activity 、 Dialog 、 Toast 它们的视图都是附加在Window上的，因此Window实际上是View的直接管理者。WindowManager 是外界访问Window的入口，通过WindowManager可以创建Window，而Window的具体实现位于 WindowManagerService 中，WindowManager和WindowManagerService的交互是一个IPC过程

* Window是View的直接管理者 WindowManager 是外界访问Window的入口

### 1. Window和WindowManager
> 下面代码演示了通过WindowManager添加Window的过程：
```
mWindowManager = (WindowManager) getSystemService(Context.WINDOW_SERVICE);
mFloatingButton = new Button(this);
mFloatingButton.setText("click me");
mLayoutParams = new WindowManager.LayoutParams(
LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT, 0, 0,
PixelFormat.TRANSPARENT);
mLayoutParams.flags = LayoutParams.FLAG_NOT_TOUCH_MODAL
| LayoutParams.FLAG_NOT_FOCUSABLE
| LayoutParams.FLAG_SHOW_WHEN_LOCKED;
mLayoutParams.type = LayoutParams.TYPE_SYSTEM_ERROR;
mLayoutParams.gravity = Gravity.LEFT | Gravity.TOP;
mLayoutParams.x = 100;
mLayoutParams.y = 300;
mFloatingButton.setOnTouchListener(this);
mWindowManager.addView(mFloatingButton, mLayoutParams);
```
> 上述代码将一个button添加到屏幕坐标为（100,300）的位置上。WindowManager的flags和type这两个属性比较重要

* Flags代表Window的属性，控制Window的显示特性
> FLAG_NOT_FOCUSABLE
  在此模式下，Window不需要获取焦点，也不需要接收各种输入事件，这个标记同时会启用FLAG_NOT_TOUCH_MODAL，最终事件会直接传递给下层具有    焦点的Window      
> FLAG_NOT_TOUCH_MODAL
在此模式下，系统将当前Window区域以外的点击事件传递给底层的Window，当前Window区域内的单击事件则自己处理。一般需要开启此标记    
> FLAG_SHOW_WHEN_LOCKED
开启此模式Window将显示在锁屏界面上     

* type参数表示Window的类型

应用Window：Activity
子Window： 如Dialog
系统Window ：如Toast和系统状态栏
Window是分层的，每个Window对应一个z-ordered，层级大的会覆盖在层级小的上面，和HTM的z-index概念一样。在三类Window中，应用Window的层级范围是199，子Window的层级范围是10001999，系统Window的层级范围是2000~2999，这些值对应WindowManager.LayoutParams的type参数。一般系统Window选用 TYPE_SYSTEM_OVERLAY 或者 TYPE_SYSTEM_ERROR （ 同时需要权限 <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" /> ） 。
WindowManager提供的功能很简单，常用的只有三个方法：

添加View
更新View
删除View
这个三个方法定义在 ViewManager 中，而WindowManager继承了ViewManager。

    public interface ViewManager
    {
        public void addView(View view, ViewGroup.LayoutParams params);
        public void updateViewLayout(View view, ViewGroup.LayoutParams params);
        public void removeView(View view);
    }
如何拖动window？
给view设置onTouchListener：mFloatingButton.setOnTouchListener(this)。在onTouch方法中更新view的位置，这个位置根据手指的位置设定。

8.2 Window的内部机制
Window是一个抽象的概念，每一个Window都对应着一个View和一个ViewRootImpl，Window和View通过ViewRootImpl来建立联系。因此Window并不是实际存在的，它是以View的形式存在的。所以WindowManager的三个方法都是针对View的，说明View才是Window存在的实体。在实际使用中无法直接访问Window，必须通过WindowManager来访问Window。

8.2.1 Window的添加过程
Window的添加过程需要通过WindowManager的addView()来实现, 而WindowManager是一个接口, 它的真正实现是WindowManagerImpl类。

WindowManagerImpl并没有直接实现Window的三大操作, 而是全部交给了WindowManagerGlobal来处理. WindowManagerGlobal以工厂的形式向外提供自己的实例. 而WindowManagerImpl这种工作模式就典型的桥接模式, 将所有的操作全部委托给WindowManagerGlobal来实现.

检查所有参数是否合法, 如果是子Window那么还需要调整一些布局参数.
创建ViewRootImpl并将View添加到列表中.
通过ViewRootImpl来更新界面并完成Window的添加过程.
这个过程是通过ViewRootImpl的setView()来完成的. View的绘制过程是由ViewRootImpl来完成的, 在内部会调用requestLayout()来完成异步刷新请求. 而scheduleTraversals()实际上是View绘制的入口. 接着会通过WindowSession完成Window的添加过程(Window的添加过程是一次IPC调用). 最终会通过WindowManagerService来实现Window的添加.
WindowManagerService内部会为每一个应用保留一个单独的Session.

8.2.2 Window的删除过程
Window 的删除过程和添加过程一样, 都是先通过WindowManagerImpl后, 在进一步通过WindowManagerGlobal的removeView()来实现的.

方法内首先通过findViewLocked来查找待删除的View的索引, 这个过程就是建立数组遍历, 然后调用removeViewLocked来做进一步的删除.

这里通过ViewRootImpl的die()完成来完成删除操作. die()方法只是发送了请求删除的消息后就立刻返回了, 这个时候View并没有完成删除操作, 所以最后会将其添加到mDyingViews中, mDyingViews表示待删除的View的列表.

die方法中只是做了简单的判断, 如果是异步删除那么就发送一个MSG_DIE的消息, ViewRootImpl中的Handler会处理此消息并调用doDie(); 如果是同步删除, 那么就不发送消息直接调用doDie()方法.

在doDie()方法中会调用dispatchDetachedFromWindow()方法, 真正删除View的逻辑在这个方法内部实现. 其中主要做了四件事:

垃圾回收的相关工作, 比如清除数据和消息,移除回调.
通过Session的remove方法删除Window: mWindowSession.remove(mWindow), 这同样是一个IPC过程, 最终会调用WMS的removeWindow()方法.
调用View的dispatchDetachedFromWindow()方法, 内部会调用View的onDetachedFromWindow()以及onDetachedFromWindowInternal(). 而对于onDetachedFromWindow()就是在View从Window中移除时, 这个方法就会被调用, 可以在这个方法内部做一些资源回收的工作. 比如停止动画,停止线程
调用WindowManagerGlobal#doRemoveView方法刷新数据, 包括mRoots, mParams, mDyingViews, 需要将当前Window所关联的这三类对象从列表中删除.
8.2.3 Window的更新过程
WindowManagerGlobal#updateViewLayout()方法做的比较简单, 它需要更新View的LayoutParams并替换掉老的LayoutParams, 接着在更新ViewRootImpl中的LayoutParams. 这一步主要是通过setLayoutParams()方法实现.

在ViewRootImpl中会通过scheduleTraversals()来对View重新布局, 包括测量,布局,重绘. 除了View本身的重绘以外, ViewRootImpl还会通过WindowSession来更新Window的视图, 这个过程最后由WMS的relayoutWindow()实现同样是一个IPC过程.

8.3 Window的创建过程
由之前的分析可以知道，View是Android中视图的呈现方式，但是View不能单独存在，必须附着在Window这个抽象的概念上面，因此有视图的地方就有Window。这些视图包括：Activity、Dialog、Toast、PopUpWindow、菜单等等。

8.3.1 Activity的Window创建过程
Activity的大体启动流程: 最终会由ActivityThread中的PerformLaunchActivity()来完成整个启动过程, 这个方法内部会通过类加载器创建Activity的实例对象, 并调用其attach()方法为其关联运行过程中所依赖的一系列上下文环境变量。

在attach()方法里, 系统会创建Activity所属的Window对象并为其设置回调接口, Window对象的创建是通过PolicyManager#makeNewWindow()方法实现. 由于Activity实现了Window的CallBack接口, 因此当Window接收到外界的状态改变的时候就会回调Activity方法. 比如说我们熟悉的onAttachedToWindow(), onDetachedFromWindow(), dispatchTouchEvent()等等。

Activity将具体实现交给了Window处理, 而Window的具体实现就是PhoneWindow, 所以只需要看PhoneWindow的相关逻辑。分为以下几步

如果没有DecorView, 那么就创建它. 由installDecor()—>generateDecor()触发
将View添加到DecorView的mContentParent中
回调Activity的onContentChanged()通知activity视图已经发生改变
这个时候DecorView已经被创建并初始化完毕, Activity的布局文件也已经添加成功到DecorView的mContentParent中. 但是这个时候DecorView还没有被WindowManager正式添加到Window中. 虽然早在Activity的attach方法中window就已经被创建了, 但是这个时候由于DecorView并没有被WindowManager识别, 所以这个时候的Window无法提供具体功能, 因为他还无法接收外界的输入信息.

在ActivityThread#handleResumeActivity()方法中, 首先会调用Activity#onResume(), 接着会调用Activity#makeVisible(), 正是在makeVisible方法中, DecorView真正的完成了添加和显示这两个过程。

8.3.2 Dialog的Window创建过程
Dialog的Window的创建过程和Activity类似, 有如下几步

创建Window
Dialog的创建后的实际就是PhoneWindow, 这个过程和Activity的Window创建过程一致
初始化DecorView并将Dialog的视图添加到DecorView中
这个过程也类似, 都是通过Window去添加指定的布局文件.
将DecorView添加到Window中并显示
在Dialog的show方法中, 会通过WindowManager将DecorView添加Window中.
普通的Dialog有一个特殊之处, 那就是必须采用Activity的Content, 如果采用Application的Content, 那么就会报错. 报的错是没有应用token所导致的, 而应用token一般只有Activity才拥有.

还有一种方法. 系统Window比较特殊, 他可以不需要token, 因此只需要指定对话框的Window为系统类型就可以正常弹出对话框.

//JAVA 给Dialog的Window改变为系统级的Window
dialog.getWindow().setType(WindowManager.LayoutParams.TYPE_SYSTEM_ERROR);
//XML 声明权限
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>
8.3.3 Toast的Window创建过程
Toast和Dialog不同, 它的工作过程就稍显复杂. 首先Toast也是基于Window来实现的. 但是由于Toast具有定时取消的功能, 所以系统采用了Handler. 在Toast的内部有两类IPC过程, 第一类是Toast访问NotificationManagerService()后面简称NMS. 第二类是NotificationManagerService回调Toast里的TN接口.

Toast属于系统Window, 它内部的视图有两种方式指定, 一种是系统默认的样式, 另一种是通过setView方法来指定一个自定义View. 不管如何, 他们都对应Toast的一个View类型的内部成员mNextView. Toast内部提供了cancel和show两个方法. 分别用于显示和隐藏Toast. 他们内部是一个IPC过程.

显示和隐藏Toast都是需要通过NMS来实现的. 由于NMS运行在系统的进程中, 所以只能通过远程调用的方式来显示和隐藏Toast. 而TN这个类是一个Binder类. 在Toast和NMS进行IPC的过程中, 当NMS处理Toast的显示或隐藏请求时会跨进程回调TN的方法. 这个时候由于TN运行在Binder线程池中, 所以需要通过Handler将其切换到当前主线程. 所以由其可知, Toast无法在没有Looper的线程中弹出, 因为Handler需要使用Looper才能完成切换线程的功能.

对于非系统应用来说, 最多能同时存在对Toast封装的ToastRecord上限为50个. 这样做是为了防止DOS(Denial of Service). 如果不这样, 当通过大量循环去连续的弹出Toast, 这将会导致其他应用没有机会弹出Toast, 那么对于其他应用的Toast请求, 系统的行为就是拒绝服务, 这就是拒绝服务攻击的含义.

在ToastRecord被添加到mToastQueue()中后, NMS就会通过showNextToastLocked()方法来显示当前的Toast.

Toast的显示是由ToastRecord的callback来完成的. 这个callback实际上就是Toast中的TN对象的远程Binder. 通过callback来访问TN中的方法是需要跨进程的. 最终被调用的TN中的方法会运行在发起Toast请求的应用的Binder线程池.

Toast的隐藏也会通过ToastRecord的callback完成的.同样是一次IPC过程. 方式和Toast显示类似.

以上基本说明Toast的显示和影响过程实际上是通过Toast中的TN这个类来实现的. 他有两个方法show(), hide(). 分别对应着Toast的显示和隐藏. 由于这两个方法是被NMS以跨进程的方式调用的, 因此他们运行在Binder线程池中. 为了将执行环境切换到Toast请求所在线程中, 在他们内部使用了handler。

TN的handleShow中会将Toast的视图添加到Window中.
TN的handleHide中会将Toast的视图从Window中移除.

以上三节的总结

在创建视图并显示出来时，首先是通过创建一个Window对象，然后通过WindowManager对象的 addView(View view, ViewGroup.LayoutParams params); 方法将 contentView 添加到Window中，完成添加和显示视图这两个过程。
在关闭视图时，通过WindowManager来移除DecorView， mWindowManager.removeViewImmediate( view); 。
Toast比较特殊，具有定时取消功能，所以系统采用了Handler，内部有两类IPC过程：
Toast访问 NotificationManagerService
NotificationManagerService 回调Toast里的 TN 接口

显示和隐藏Toast都通过NotificationManagerService（ NMS） 来实现，而NMS运行在系统进程中，所以只能通过IPC来进行显示/隐藏Toast。而TN是一个Binder类，在Toast和NMS进行IPC的过程中，当NMS处理Toast的显示/隐藏请求时会跨进程回调TN中的方法，这时由于TN运行在Binder线程池中，所以需要通过Handler将其切换到当前线程（ 即发起Toast请求所在的线程） ，然后通过WindowManager的 addView/removewView 方法真正完成显示和隐藏Toast。

