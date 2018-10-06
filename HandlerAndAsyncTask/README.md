# Handler   
> Android SDK提供给开发者方便进行异步消息处理的类   
> 诸如AsyncTask或者开源框架retrofit等，都是基于Handler的二次封装   


# Handler的基本组件  
> 1.Message:Handler接受和处理消息的对象   
> 2.Looper:每一个线程只能有一个Looper，Looper能够读取MessageQueue中的消息，每读到一个Message就会将它交给Handler的handleMessage来处理    
> 3.MessageQueue:消息队列，先进先出   
> 4.Handler:**发送消息**并 **处理消息**   

# Handler的整体机制   
> 在主线程中创建一个Looper，同时在Looper内部创建一个消息队列(MessageQueue)，在创建Handler的时候就会取出当前线程的Looper，通过Looper不断的去轮询MessageQueue中的Message，而通过Handler发送一个消息，其实就是在MessageQueue中添加一个Message   

# Handler源码分析   
> **构造器(Handler.java)**   
>> ```Java
>> public Handler(Callback callback, boolean async) {
>>    if (FIND_POTENTIAL_LEAKS) {
>>        final Class<? extends Handler> klass = getClass();
>>        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
>>                (klass.getModifiers() & Modifier.STATIC) == 0) {
>>            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
>>                klass.getCanonicalName());
>>        }
>>    }
>>    //主要看这里，Handler构造器对象对创建一个Looper对象
>>    mLooper = Looper.myLooper();
>>    if (mLooper == null) {
>>        //判断Looper对象是否创建成果，如果失败则会报出异常
>>        //Looper.prepare()必须在线程内部首先被调用，这是因为Handler需要往MessageQueue中添加消息，而MessageQueue是由Looper管理的，如果我们希望Handler能够正常工作，就必须确保在当前线程中有一个指定的Looper对象
>>        throw new RuntimeException(
>>            "Can't create handler inside thread that has not called Looper.prepare()");
>>    }
>>    //将Looper的MessageQueue绑定到Handler上
>>    mQueue = mLooper.mQueue;
>>    mCallback = callback;
>>    mAsynchronous = async;
>>}
>>```    
---
> **发送消息(Handler.java)**   
>> 不论是sendMessage还是sendEmptyMessage还是sendEmptyMessageDelayed，最终都会调用enqueueMessage  
>> ```Java
>> //将Handler发送的消息添加到MessageQueue中
>> private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
>>         //将当前的Handler绑定到消息上
>>         msg.target = this;
>>         if (mAsynchronous) {
>>             msg.setAsynchronous(true);
>>         }
>>         return queue.enqueueMessage(msg, uptimeMillis);
>>     }
>> ```   
---
> **Looper管理消息队列(Looper.java)**  
>> ```Java   
>> private static void prepare(boolean quitAllowed) {
>>     // sThreadLocal作为一个容器，存放Looper对象本身  
>>     // sThreadLocal可以确保每一个线程获取的Looper都是唯一的  
>>     // sThreadLocal也可以称为线程本地变量，它可以为每一个变量在每一个线程中创建一个副本，这样每个线程都可以通过访问自身内部副本的变量，避免了线程之间相互取变量的问题   
>>     if (sThreadLocal.get() != null) {
>>         //如果检测到当前线程中已经存在了Looper对象，如果存在抛出异常警告
>>         throw new RuntimeException("Only one Looper may be created per thread");
>>     }
>>     //将没有创建好线程的Looper对象放到线程本地变量中，这样下次就可以直接用了
>>     sThreadLocal.set(new Looper(quitAllowed));
>> }
>> ```   
---
> **构造器(Looper.java)**   
>> ```Java
>> //可以看到在构造器中Looper就已经创建了MessageQueue
>> private Looper(boolean quitAllowed) {
>>     mQueue = new MessageQueue(quitAllowed);
>>     mThread = Thread.currentThread();
>> }
>> ```   
> 需要注意的是，**MessageQueue的数据结构并不是队列，而是一个单链表**   
---
> **要点总结**
>> 1.Looper类主要是为每个线程开启的单独的消息循环。Android中所有新诞生的线程都是没有开启消息循环的，所以如果想要在线程中使用Handler，必须先使用Looper.prepare才可以(**主线程除外**)   
>> 2.Handler是Looper的一个接口，用来向指定Looper中的MessageQueue发送消息   
>> 3.在非主线程中 **不能**直接new Handler()。如果想要在子线程中使用Handler，那么必须先创建Looper对象，同1。   


# AsyncTask  
> AsyncTask实际上是对Handler和线程池的一种封装，AsyncTask实际上是一个抽象类，在使用它时需要用子类继承并重写四个方法。
> 在使用AsyncTask时，通过调用 **AsyncTask.execute**方法并传入参数，来异步执行某些操作。       
>> ```Java
>>    /**
>>     * 运行在UI线程中，在调用doInBackground()之前执行，
>>     * 在调用完AsyncTask.execute的方法之后，该接口会被立即回调，可以在这里做一些数据的初始化操作
>>     */
>>    @Override
>>    protected void onPreExecute() {
>>        Toast.makeText(context, "开始执行", Toast.LENGTH_SHORT).show();
>>    }
>>
>>    /**
>>    * 后台运行的方法，可以运行非UI线程，可以执行耗时的方法
>>    * 在onPreExecute方法执行完毕后立即执行
>>     */
>>    @Override
>>    protected Integer doInBackground(Void... voids) {
>>        int i = 0;
>>        while (i < 10) {
>>            i++;
>>            //调用publishProgress方法后，下面的onProgressUpdate会被调用
>>            publishProgress(i);
>>            try {
>>                Thread.sleep(1000);
>>           } catch (InterruptedException e) {
>>            }
>>        }
>>       return null;
>>    }
>>
>>
>>    /**
>>     * 运行在ui线程中，在doInBackground()执行完毕后执行
>>     */
>>    @Override
>>    protected void onPostExecute(Integer integer) {
>>        Toast.makeText(context, "执行完毕", Toast.LENGTH_SHORT).show();
>>    }
>>
>>    /**
>>     * 在publishProgress()被调用以后执行，publishProgress()用于更新进度，也是在主线程中执行
>>     */
>>    @Override
>>    protected void onProgressUpdate(Integer... values) {
>>          //tv.setText(""+values[0]);
>>   }
>> ```   
---
> **使用要点**  
>> 1.AsyncTask的实例必须在主线程中创建  
>> 2.AsyncTask的execute方法必须在主线程中调用  
>> 3.它的几个回调方法会由Android自动调用  
>> 4.一个AsyncTask实例，只能执行一次execute方法  
>> 5.AsyncTask默认情况下是串行执行任务的     

---  
> **FutureTask 和 Callable 补充**  
>> Callable是一个接口，内部有一个抽象方法call，它与runnable类似，但是跟runnable不同的是，callable的call方法有一个泛型的返回值，而runnable的run方法没有返回值   

> **Async源码分析**  
>> **构造器(AsyncTask.java)**   
>>> 会传入一个Looper对象作为参数，内部有一个WorkerRunnable实例对象，这个WorkerRunnable就实现了上面提到的 **Callable**接口，在这里会实现它的 **call**方法，最终调用一个名为 **postResult**的方法。   
---
>> **postResult**   
>>> ```Java   
>>> private Result postResult(Result result) {
>>>    @SuppressWarnings("unchecked")
>>>    //通过getHandler获取到主线程的Handler，并根据这个Handler来进行判断
>>>    //将AsyncTaskResult发送给Handler来处理
>>>    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
>>>            new AsyncTaskResult<Result>(this, result));
>>>    message.sendToTarget();
>>>    return result;
>>> }
>>> ```   
---
>>> ```Java
>>> private static class InternalHandler extends Handler {
>>>     public InternalHandler(Looper looper) {
>>>         super(looper);
>>>     }
>>>     @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
>>>     @Override
>>>     public void handleMessage(Message msg) {
>>>         AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
>>>         switch (msg.what) {
>>>             case MESSAGE_POST_RESULT:
>>>                 // There is only one result
>>>                 //mTask就是当前绑定的AsyncTask，result.mData[0]代表的就是AsyncTask中调用doInBackground返回的结果
>>>                 //在finish中会判断当前的AsyncTask是否被取消，如果没有就执行我们重写的AsyncTask的onPostExecute方法
>>>                 result.mTask.finish(result.mData[0]);
>>>                 break;
>>>             case MESSAGE_POST_PROGRESS:
>>>                 result.mTask.onProgressUpdate(result.mData);
>>>                break;
>>>         }
>>>     }
>>> }
>>> ```   
---
> **实例化AsyncTask**
>> ```Java  
>> public AsyncTask(@Nullable Looper callbackLooper) {
>>     //这里的mHandler就是主线程的Handler
>>    mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
>>        ? getMainHandler()
>>        : new Handler(callbackLooper);
>>      //Callable类型的对象
>>    mWorker = new WorkerRunnable<Params, Result>() {
>>        //call方法是在线程池中的某个线程中执行的
>>        public Result call() throws Exception {
>>          //设置为true之后就表示当前任务已经开始执行
>>            mTaskInvoked.set(true);
>>            Result result = null;
>>            try {
>>                // 将call所跑的线程设置为后台线程
>>                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
>>                //noinspection unchecked
>>                //在工作线程中做doInBackground的一些耗时操作
>>                result = doInBackground(mParams);
>>                Binder.flushPendingCommands();
>>            } catch (Throwable tr) {
>>               mCancelled.set(true);
>>                throw tr;
>>            } finally {
>>            // 将上面doInBackground的返回值交给postResult
>>                postResult(result);
>>            }
>>            return result;
>>        }
>>    };
>>      //
>>    mFuture = new FutureTask<Result>(mWorker) {
>>        @Override
>>        protected void done() {
>>            try {
>>                postResultIfNotInvoked(get());
>>            } catch (InterruptedException e) {
>>                android.util.Log.w(LOG_TAG, e);
>>            } catch (ExecutionException e) {
>>                throw new RuntimeException("An error occurred while executing doInBackground()",
>>                        e.getCause());
>>            } catch (CancellationException e) {
>>                postResultIfNotInvoked(null);
>>            }
>>        }
>>    };
>> }
>> ```
---
>> **AsyncTask中线程池的execute和executeOnExecutor方法**
>>> ```Java
>>> //AsyncTask对象在被实例化之后，我们一般都是通过执行它的execute方法来执行异步操作
>>> //从下面的注解可以看到，execute方法是跑在主线程中的
>>> @MainThread
>>> public final AsyncTask<Params, Progress, Result> execute(Params... params) {
>>>     //sDefaultExecutor就是我们默认执行任务的线程池
>>>     return executeOnExecutor(sDefaultExecutor, params);
>>> }
>>> ```
---
>>> ```java
>>> @MainThread
>>> public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
>>>         Params... params) {
>>>     if (mStatus != Status.PENDING) {
>>>         switch (mStatus) {
>>>             case RUNNING:
>>>                 throw new IllegalStateException("Cannot execute task:"
>>>                         + " the task is already running.");
>>>             case FINISHED:
>>>                 throw new IllegalStateException("Cannot execute task:"
>>>                         + " the task has already been executed "
>>>                         + "(a task can be executed only once)");
>>>         }
>>>     }
>>> 
>>>     mStatus = Status.RUNNING;
>>>     //实现判断当前AsyncTask是否已经在执行或者已经结束执行，因为一个AsyncTask实例只能执行一次
>>>     //所以如果是正在执行或者已经结束，这边是走不到的，在上面就已经抛出了异常
>>>     onPreExecute();
>>>     mWorker.mParams = params;
>>>     //mFuture就是FutureTask实例，实现了Callable和Runnable的接口
>>>     exec.execute(mFuture);
>>> 
>>>     return this;
>>> }
>>>```