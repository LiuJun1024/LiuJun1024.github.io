#异步消息机制
##Handler
###1. constructor of Handler


	/**
 	 * Default constructor associates this handler with the {@link Looper} for the
     * current thread.
     *
     * If this thread does not have a looper, this handler won't be able to receive messages
     * so an exception is thrown.
     */
    public Handler() {
        this(null, false);
    }


这个构造函数得到的Handler默认属于当前线程，而且如果当前线程如果没有Looper则通过这个默认构造实例化Handler时会抛出异常，至于是啥异常还有为啥咱们继续往下分析，this(null, false)的实现如下：

	public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
        mLooper = Looper.myLooper();
        if (mLooper == null) {//此处解释了为什么当前线程没有looper会抛异常
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }



##Lopper
###1. myLopper()方法
	 /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static Looper myLooper() {
        return sThreadLocal.get();
    }

从sThreadLocal对象中get了一个Looper对象返回。跟踪了一下sThreadLocal对象，发现他定义在Looper中，是一个static final类型的ThreadLocal<Looper>对象（在Java中，一般情况下，通过ThreadLocal.set() 到线程中的对象是该线程自己使用的对象，其他线程是不需要访问的，也访问不到的，各个线程中访问的是不同的对象。）。所以可以看出，如果sThreadLocal中有Looper存在就返回Looper，没有Looper存在自然就返回null了。
###2. prepare()

	 public static void prepare() {
        prepare(true);
    }

	private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

prepare()仅仅是对prepare(boolean quitAllowed) 的封装而已，默认传入了true，也就是将MessageQueue对象中的quitAllowed标记标记为true而已

另外，UI线程（Activity等）启动的时候系统已经帮我们自动调用了Looper.prepare()方法，所以我们在UI线程中使用Handler时不需要调用prepare()了

以前一直都说Activity的入口是onCreate方法，其实android上一个应用的入口应该是ActivityThread类的main方法就行了。

所以为了解开UI Thread为何不需要创建Looper对象的原因，我们看下ActivityThread的main方法，如下：

	public static void main(String[] args) {
        SamplingProfilerIntegration.start();

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Environment.initForCurrentUser();

        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());

        Security.addProvider(new AndroidKeyStoreProvider());

        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();//注意这里！！！

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }

我们看看这个 Looper.prepareMainLooper();

	public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }

可以看到，UI线程中会始终存在一个Looper对象（sMainLooper 保存在Looper类中，UI线程通过getMainLooper方法获取UI线程的Looper对象），从而不需要再手动去调用Looper.prepare()方法了。如下Looper类提供的get方法：

	 public static Looper getMainLooper() {
        synchronized (Looper.class) {
            return sMainLooper;
        }
    }

###3.constructor of Lopper
	private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }

Lopper的构造方法中会创建一个MessageQueue，用于将所有收到的消息以队列的形式进行排列，并提供入队和出队的方法。）得到当前Thread实例引用而已。通过这里可以发现，一个Looper只能对应了一个MessageQueue。


>整理一下思路：线程中使用Handler之前必须调用Looper.prepare()方法，prepare()方法中会向Looper类中的sThreadLocal中添加一个Lopper对象，之后使用Handler时，Handler的构造方法中会通过Looper.myLooper()方法从sThreadLocal得到Looper对象，并且得到该Looper对象的消息队列mQueue


##Handler发送消息时都发生了什么？

Handler中提供的很多个发送消息方法中除了sendMessageAtFrontOfQueue()方法之外，其它的发送消息方法最终都调用了sendMessageAtTime()方法。所以，咱们先来看下这个sendMessageAtTime方法，如下：

	public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

"除外"的sendMessageAtFrontOfQueue()方法：

	public final boolean sendMessageAtFrontOfQueue(Message msg) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, 0);
    }

和sendMessageAtTime()方法基本一致。

两个基本的方法最终都调用enqueueMessage()方法：

	 private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }

而handler中的enqueueMessage()方法最终调用消息队列的enqueueMessage()方法：

	boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w("MessageQueue", e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }


可以看出，Handler发送消息的时候其实是将msg加入到了Looper的消息队列中


##Handler接受消息的时候发生了什么？

Looper.prepare()后，使用完Handler后要 Looper.loop()；
看看Looper类的loop()方法：

	/**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            msg.target.dispatchMessage(msg);

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }

可以看出，loop()方法中会得到当前Looper对象从而得到Looper的消息队列MessageQueue，然后往下走，在一个死循环中调用**MessageQueue的next()方法：**

	 Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (false) Log.v("MessageQueue", "Returning message: " + msg);
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf("MessageQueue", "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }

可以看到，如果当前MessageQueue中存在待处理的消息mMessages就将这个消息出队，然后让下一条消息成为mMessages，否则就进入一个阻塞状态（在上面Looper类的loop方法上面也有英文注释，明确说到了阻塞特性），一直等到有新的消息入队。

看loop()方法中（msg.target.dispatchMessage(msg);），每当有一个消息出队就将它传递到msg.target的dispatchMessage()方法中。其中这个msg.target其实就是上面分析Handler发送消息代码部分Handler的enqueueMessage方法中的msg.target = this;语句，也就是当前Handler对象。所以接下来的重点自然就是回到Handler类看看我们熟悉的dispatchMessage()方法，如下：

	/**
     * Handle system messages here.
     */
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }

可以看见dispatchMessage方法中的逻辑比较简单，具体就是如果mCallback不为空，则调用mCallback的handleMessage()方法，否则直接调用Handler的handleMessage()方法，并将消息对象作为参数传递过去。

##Handler停止接受消息

我们已经知道Looper.loop()方法中会以一个死循环不停遍历消息队列，如何停止呢？这就用到Looper中的quit()方法:

	public void quit() {
        mQueue.quit(false);
    }

Looper.quit()方法调用了消息队列的quit(boolean safe)方法：

	 void quit(boolean safe) {
        if (!mQuitAllowed) {
            throw new IllegalStateException("Main thread not allowed to quit.");
        }

        synchronized (this) {
            if (mQuitting) {
                return;
            }
            mQuitting = true;

            if (safe) {
                removeAllFutureMessagesLocked();
            } else {
                removeAllMessagesLocked();
            }

            // We can assume mPtr != 0 because mQuitting was previously false.
            nativeWake(mPtr);
        }
    }

通过判断标记mQuitAllowed来决定该消息队列是否可以退出，然而当mQuitAllowed为fasle时抛出的异常竟然是”Main thread not allowed to quit.”，Main Thread，所以可以说明Main Thread关联的Looper一一对应的MessageQueue消息队列是不能通过该方法退出的。**这个mQuitAllowed值是在Looper.prepqre()中赋值的**追根到底说明mQuitAllowed就是Looper.prpeare的参数，我们默认调运的Looper.prpeare();其中对mQuitAllowed设置为了true，所以可以通过quit方法退出，而主线程ActivityThread的main中使用的是Looper.prepareMainLooper();，这个方法里对mQuitAllowed设置为false，所以才会有上面说的”Main thread not allowed to quit.”。回到quit方法继续看，可以发现实质就是对mQuitting标记置位，这个mQuitting标记在MessageQueue的阻塞等待next方法中用做了判断条件，所以可以通过quit方法退出整个当前线程的loop循环。


##关于Handler导致内存泄露的分析与解决方法

当使用内部类（包括匿名类）来创建Handler的时候，Handler对象会隐式地持有一个外部类对象（通常是一个Activity）的引用。而Handler通常会伴随着一个耗时的后台线程一起出现，这个后台线程在任务执行完毕之后，通过消息机制通知Handler，然后Handler把消息发送到UI线程。然而，如果用户在耗时线程执行过程中关闭了Activity（正常情况下Activity不再被使用，它就有可能在GC检查时被回收掉），由于这时线程尚未执行完，而该线程持有Handler的引用，这个Handler又持有Activity的引用，就导致该Activity暂时无法被回收（即内存泄露）。

###解决方法：

####一 通过程序逻辑：
1、在关闭Activity的时候停掉你的后台线程。线程停掉了，就相当于切断了Handler和外部连接的线，Activity自然会在合适的时候被回收。

2、如果你的Handler是被delay的Message持有了引用，那么使用相应的Handler的removeCallbacks()方法，把消息对象从消息队列移除就行了（如上面的例子部分的onStop中代码）。

####二 将Handler声明为静态类
静态类不持有外部类的引用，所以你的Activity可以随意被回收。代码如下：

	static class TestHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        mImageView.setImageBitmap(mBitmap);
    }
	}

这时你会发现，由于Handler不再持有外部类对象的引用，导致程序不允许你在Handler中操作Activity中的对象了。所以你需要在Handler中增加一个对Activity的弱引用（WeakReference），如下：

	static class TestHandler extends Handler {
    WeakReference<Activity > mActivityReference;

    TestHandler(Activity activity) {
        mActivityReference= new WeakReference<Activity>(activity);
    }

    @Override
    public void handleMessage(Message msg) {
        final Activity activity = mActivityReference.get();
        if (activity != null) {
            mImageView.setImageBitmap(mBitmap);
        }
    }
	}

##HandlerThread源码分析

	public class HandlerThread extends Thread {
    //线程的优先级
    int mPriority;
    //线程的id
    int mTid = -1;
    //一个与Handler关联的Looper对象
    Looper mLooper;

    public HandlerThread(String name) {
        super(name);
        //设置优先级为默认线程
        mPriority = android.os.Process.THREAD_PRIORITY_DEFAULT;
    }

    public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;
    }
    //可重写方法，Looper.loop之前在线程中需要处理的其他逻辑在这里实现
    protected void onLooperPrepared() {
    }
    //HandlerThread线程的run方法
    @Override
    public void run() {
        //获取当前线程的id
        mTid = Process.myTid();
        //创建Looper对象
        //这就是为什么我们要在调用线程的start()方法后才能得到Looper(Looper.myLooper不为Null)
        Looper.prepare();
        //同步代码块，当获得mLooper对象后，唤醒所有线程
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        //设置线程优先级
        Process.setThreadPriority(mPriority);
        //Looper.loop之前在线程中需要处理的其他逻辑
        onLooperPrepared();
        //建立了消息循环
        Looper.loop();
        //一般执行不到这句，除非quit消息队列
        mTid = -1;
    }

    public Looper getLooper() {
        if (!isAlive()) {
            //线程死了
            return null;
        }

        //同步代码块，正好和上面run方法中同步块对应
        //只要线程活着并且mLooper为null，则一直等待
        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }

    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            //退出消息循环
            looper.quit();
            return true;
        }
        return false;
    }

    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            //退出消息循环
            looper.quitSafely();
            return true;
        }
        return false;
    }

    public int getThreadId() {
        //返回线程id
        return mTid;
    }
	}
其实HandlerThread就是Thread、Looper和Handler的组合实现，Android系统这么封装体现了Android系统组件的思想，同时也方便了开发者开发。HandlerThread主要是对Looper进行初始化，并提供一个Looper对象给新创建的Handler对象，使得Handler处理消息事件在子线程中处理。这样就发挥了Handler的优势，同时又可以很好的和线程结合到一起。

