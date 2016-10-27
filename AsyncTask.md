# AsyncTask

##execute()方法  

	public final AsyncTask<Params, Progress, Result> execute(Params... params) {  
        return executeOnExecutor(sDefaultExecutor, params);  
	} 

execute()方法调用了executeOnExecutor()方法：
 
	public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,  
            Params... params) {  
        if (mStatus != Status.PENDING) {  
            switch (mStatus) {  
                case RUNNING:  //Status.PENDING是Status的默认状态(等待执行)
                    throw new IllegalStateException("Cannot execute task:"  
                            + " the task is already running.");  
                case FINISHED:  
                    throw new IllegalStateException("Cannot execute task:"  
                            + " the task has already been executed "  
                            + "(a task can be executed only once)");  
            }  
        }  
  
        mStatus = Status.RUNNING;  //将任务的状态改成正在执行,从switch语句看出，每个任务在完成前只能执行一次，
  
        onPreExecute();  //主线程的准备工作
  
        mWorker.mParams = params;//将任务所需的参数传给mWorker.mParams  
        exec.execute(mFuture);  //exec(就是execute()方法传入的sDefaultExecutor)调用,这个方法中会调用mFuture的run()方法，而mFuture得run()方法中会调用mWorker中的call方法，call方法中：1、doInBackground()方法会执行，2、之后将doInBackgroung()返回结果发送
  
        return this;  
    }  


##mWorker
上面的代码中，mWorker是Asynctask的一个成员变量：

	private final WorkerRunnable<Params, Result> mWorker;
是一个实现了Callable接口的抽象类：

	private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
    }

在Asynctask的构造方法中对mWorker进行了初始化：

	public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);  //doInBackground()
                Binder.flushPendingCommands();
                return postResult(result);
            }
        };
	
	//...mFuture的初始化

	}

postResult():

	private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }

	private static Handler getHandler() {
        synchronized (AsyncTask.class) {
            if (sHandler == null) {
                sHandler = new InternalHandler();
            }
            return sHandler;
        }
    }

接收消息的Handler类:

	 private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);//在主线程中执行的更新操作，所以要求上面的sHandler必须在主线程中创建，而由于sHandler是AsyncTask的静态成员变量，所以AsyncTask必须在主线程中加载
                    break;
            }
        }
    }
finish()方法:

	private void finish(Result result) {  
        if (isCancelled()) {  
            onCancelled(result);  
        } else {  
            onPostExecute(result);  
        }  
        mStatus = Status.FINISHED;  
    }

##mFuture

	public AsyncTask() {

       //...mWorker的初始化

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());//get()表示获取mWorker的call的返回值
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }

postResultIfNotInvoked()方法：

	private void postResultIfNotInvoked(Result result) {
        final boolean wasTaskInvoked = mTaskInvoked.get();
        if (!wasTaskInvoked) {	//在mWorker初始化时就已经将mTaskInvoked为true，所以一般这个postResult执行不到。
            postResult(result);
        }
    }

##sDefaultExecutor

	private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;  
	public static final Executor SERIAL_EXECUTOR = new SerialExecutor();  
	private static class SerialExecutor implements Executor {  
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();  
        Runnable mActive;  
        public synchronized void execute(final Runnable r) {  
            mTasks.offer(new Runnable() {  
                public void run() {  
                    try {  
                        r.run();  
                    } finally {  
                        scheduleNext();  
                    }  
                }  
            });  
            if (mActive == null) {  
                scheduleNext();  
            }  
        }  
        protected synchronized void scheduleNext() {  
            if ((mActive = mTasks.poll()) != null) {  
                THREAD_POOL_EXECUTOR.execute(mActive);  
            }  
        }  
	}
sDefaultExecutor其实为SerialExecutor的一个实例，其内部维持一个任务队列；直接看其execute（Runnable runnable）方法，将runnable放入mTasks队尾；  
判断当前mActive是否为空，为空则调用scheduleNext方法  
scheduleNext，取出任务队列中的队首任务，如果不为null则传入THREAD_POOL_EXECUTOR进行执行。
下面看THREAD_POOL_EXECUTOR为何方神圣：

	public static final Executor THREAD_POOL_EXECUTOR  
          =new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,  
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);

虽然这是一个线程池，但如果有10个任务同时调用execute（）方法，第一个任务入队，然后在mActive = mTasks.poll()) != null被取出，并且赋值给mActivte，然后交给线程池去执行。然后第二个任务入队，但是此时mActive并不为null，并不会执行scheduleNext();所以如果第一个任务比较慢，10个任务都会进入队列等待；真正执行下一个任务的时机是，线程池执行完成第一个任务以后，调用Runnable中的finally代码块中的scheduleNext，所以虽然内部有一个线程池，其实调用的过程还是线性的。一个接着一个的执行，相当于单线程。


##总结一下:
>1. execute()方法调用了executeOnExecutor()方法，
2. executeOnExecutor()方法中，设置当前AsyncTask的状态为RUNNING，上面的switch也可以看出，每个异步任务在完成前只能执行一次。
执行onPreExecute()，当前依然在UI线程，所以我们可以在其中做一些准备工作。
将我们传入的参数赋值给了mWorker.mParams ,mWorker为一个Callable的子类，且在内部的call()方法中，调用了doInBackground(mParams)，然后得到的返回值作为postResult的参数进行执行；postResult中通过sHandler发送消息，最终sHandler的handleMessage中完成onPostExecute的调用。
exec.execute(mFuture)，mFuture为真正的执行任务的单元，将mWorker进行封装，然后由sDefaultExecutor交给线程池进行执行。
3. AsyncTask对象必须在主线程中创建，(在哪个线程中创建实例，onPostExcute方法就会在哪个线程中执行，如果要更新UI，则必须在主线程中创建实例)  
4. 默认情况下是串行执行的，  
5. 一个AsyncTask对象只能执行一次(FutureTask主要是在FutureTask的构造函数中传入callable。当通过ThreadPoolExecutor.execute(FutureTask)后，会调用FutureTask.run，在run中会调用Callable.call函数，执行完后会将callable清空，所以FutureTask只能被执行一次，第二次执行时会发现callable已经为空)