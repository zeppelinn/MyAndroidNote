#Activity四种形态  
+ Active: Activity处于栈顶，用户可见，并能与用户进行交互  
+ Paused: 可见但不可交互，此时该Activity中所有的状态、成员变量等都仍然存在，只有在系统内存不足的情况下才会被回收  
+ Stopped: 不可见，和Paused状态一样，只有在系统内存不足的时候成员变量、状态等才会被系统回收
+ Killed: Activity被系统回收  

#Activity生命周期  
##异常情况下的Activity生命周期  
+ 当出现异常情况时，Android会调用Activity的 **onSaveInstanceState** 来保存当前Activity的状态  
+ 而当从异常状态恢复时，Android又会调用Activity的 **onRestoreInstanceState** 来恢复Activity在保存之前的状态，同时也会调用**onCreate**，两个回调方法的参数一致。需要注意的是，如果 **onRestoreInstanceState**被调用的话，那么它的参数(Bundle savedInstanceState)则 **不能为空**，如果为空，则该方法不会被调用。所以如果需要处理异常恢复的情况，建议在 **onRestoreInstanceState**中处理。  

#Activity之间的通信  
##1.Intent/Bundle   
> Intent既可以通过startActivity来开启一个活动，也可以通过startActivityForResult来在开启活动的同时传递结果给原来的Activity  
##2.类静态变量  
##3.全局变量    

#Activity 与 Fragment 通信  
##1.Activity将数据传递给Fragment  
> + Bundle: Activity通过 **Fragment.setArguments**将Bundle的实例化对象传送到Fragment当中，Fragment再通过 **getArguments**可以获取到Bundle当中保存的键值对。
----  
> + 直接在Activity中定义方法，Fragment获取到对应的Activity，然后再获取到对应的方法。在 **Fragment.onAttach(Activity activity)** 方法中可以将参数activity强转成为本Fragment依附的Activity，由此就可以直接调用Activity中的方法了。  

##2.Fragment传递数据给Activity  
> 广播 或者EventBus     
----  
> 接口回调: 在Fragment内部定义一个Listener，它所依附的Activity来实现这个Listener中定义的接口，在 **Fragment.onAttach(Activity activity)**将activity强转成(Listener)activity,并赋值给本地的Listener的实例对象listener，这样本地在调用listener中的方法并试图传递数据的时候，Activity就能通过实现的接口的参数获取到数据了。  

#Activity 与 Service 之间通信  
##1.绑定服务，利用ServiceConnection  
> Activity中通过bindService来绑定服务，ServiceConnection的实例化对象作为该方法的第二个参数传入到bindService中，同时要实现ServiceConnection的两个接口(onServiceConnected和onServiceDisconnected)  
##2.利用Intent传值，实现简单通信  
> Activity通过调用startService(Intent intent)，将数据包装到intent中来向Service传值。Service则会在onStartCommand中接收到这个intent对象。  
##3.定义Callback接口来监听服务中的进程变化  


#Activity 启动模式  
> 1.standard  
>> 系统默认启动模式，每启动一个Activity实例，系统就会将它置于栈的顶部，无论这个Activity时候已经被启动过。当以该模式启动一个Activity时，它的onCreate、onStart和onResume都会被一次调用。  

> 2.singleTop  
>> 栈顶复用。如果当前Activity已经存在在栈顶位置，那么再次启动该Activity的时候，系统就不会再创建该Activity的实例。这种方式比standard模式更节省资源。但是如果该Activity存在栈中但不是在栈顶的位置，那么在启动同一Activity的时候还是会创建一个新的Activity实例。同时在singleTop模式下开启一个Activity的时候，它的onNewIntent会被调用 **需要注意的是**，无论是standard还是singleTop，Activity在被启动的时候都不会另外再开辟一个新的任务栈，这与下面的两种模式有区别。  
>> 使用场景：IM对话框、新闻客户端推送  

> 3.singleTask  
>> 栈内复用模式。启动该模式的Activity之前，系统会在当前任务栈中检测时候存在该Activity实例，如果存在，就直接将该Activity实例置于栈顶，同时调用onNewIntent，而且本栈中位于该Activity上面的所有Activity都会被销毁，比较暴力。  
>> 在启动一个singleTask模式的Activity之前，系统会先根据taskAffinity(任务相关性)去寻找当前是否存在一个对应名字的任务栈。如果不存在就会创建一个新的task任务栈，并将新的Activity实例添加到该任务栈中。如果存在，就会将得到该任务栈，并查找该任务栈中是否存在该Activity实例。所以在开发过程中，我们可以将多个App中的Activity设置成拥有相同的taskAffinity，这样在启动这些Activity的时候就不会另外开辟新的task栈了。  
>> 应用场景：应用的主页面  

> 4.singleInstance  
>>该模式启动的Activity在整个系统中也只有一个实例，并且该实例独享一个任务栈。  







#Service  
##service 和 Thread 的区别和场景  
>Thread:程序执行的最小单元，是分配CPU的基本单位  
>>Thread的生命周期  
>>> 1.新建  
>>> 2.就绪：线程已经被启动，并正在等待CPU轮训到的时间片  
>>> 3.运行：已经获取到CPU分配的系统资源，开始执行相关操作  
>>> 4.死亡：线程执行完毕，或者被其他线程杀死，此时线程不可能再进入就绪或等待的状态  
>>> 5.阻塞：由于某种原因导致当前正在运行的线程让出自己的资源，并进入等待的状态  
>>缺点：**无法控制！** 一般的子线程是由Activity来管理的，如果Activity被挂起或者被销毁，而此时该子线程没有连带着一起销毁，那么它会一直运行下去，直到执行完毕。  
---  
>>Service  
>>>Service是Android的一种机制，是由主线程来托管的，所以Service中不可以进行耗时操作，否则系统会报ANR  
>>Service声明周期流程(两种方式)  
>>>1.调用startService()/stopService():  onCreate() ->    onStartCommand()    -> onDestroy()  
>>>2.调用bindService()/unBindService(): onCreate() -> onBind() -> onUnbind() -> onDestroy()  

##Service 和 IntentService  
>**IntentService**  
>>IntentService内部有一个工作线程HandlerThread来处理耗时操作  
>>IntentService在执行完毕操作之后会自己主动结束，不需要其他角色来stop  
>>IntentService可以启动多个，并且所有的操作会以队列的形式被执行。IntentService一次只会执行一个操作  

##启动Service和绑定Service  
>1.先绑定服务后启动服务：即使绑定服务的Activity被销毁，也不会对Service有影响  
>2.先启动服务后绑定服务：在Activity调用了onStop之后，Service也会结束  

##序列化Parcelable和Serializable  
> 由于存在内存中的对象都不是常驻的，为了对对象实现持久化，就需要将对象写入磁盘当中，这就是序列化。而反序列化就是将数据从磁盘等其他介质中移动到内存中。  
> 一个对象想要实现序列化操作，那么它的类就必须实现Parcelable（Android特有）或者Serializable（Java原生支持）接口。  
> Parcelable 和 Serializable 的区别  
>> 一个类只要实现Serializable就被序列化和反序列化，在实现过程中会生成一个serialVersionUID，当一个实现了Serializable接口的对象需要被反序列化时，系统会检测该文件中的UID，并将其与当前类的UID对比。只有当两个UID相同时，实现了Serializable接口的实例对象才能够被序列化。这种操作 **内存开销比较大**  
>> Parcelable相较于Serializable对内存的占用比较小，但是代码实现比较麻烦。  
>> 如果只是在内存间进行数据的传输(Activity之间或者aidl之间)，考虑性能问题，可以使用Parcelable。而Serializable对数据持久化的支持更好，所以如果要对数据进行持久化操作，使用Serializable比较好。同时考虑到android版本之间的差异，对Parcelable的实现可能会有区别，所以尽量选择Serializable。  

##binder  
###AIDL  
>aidl(android接口定义语言)是最常见的binder应用实例，是android提供的进程间通信机制(IPC)  
>使用aidl  
>> 1.创建AIDL: 实体对象(需要对对象进行序列化，可以让aidl对象实现Parcelable接口)、新建AIDL文件、make工程  
>> 2.服务端: 新建Service、创建Binder对象、定义方法  
>> 3.客户端: 实现ServiceConnection接口，调用bindService，通过onServiceConnected时传过来的IBinder对象获取aidl实例  



#Broadcast  
##静态注册和动态注册  
> 1.静态注册: 在AndroidManifest.xml文件中定义receiver标签，并将实现的广播接收器注册到里面来。同时填写intent-filter标签，指明该广播对应的action。当App启动的时候，系统就会自动的实例化并注册该广播。   
> 2.动态注册: 调用Context的registerReceiver方法来动态注册广播。动态调用的好处是可以随时调用unregisterReceiver来取消订阅该广播。为了防止内存泄露，我们一般在onResume中动态注册广播，在onPause中动态反注册广播。因为onPause在整个App被销毁之前(无论何种情况)都一定会被执行，所以放在这里比较安全。  
> 3.需要注意的是，**广播接收器默认是运行在UI线程中的**，所以不能在onReceive里面做一些耗时操作。   


#WebView   
##WebView常见的一些坑   
> 1.Android API Level 16以及之前的版本存在远程代码执行安全漏洞，该漏洞源于程序没有正确限制使用WebView.addJavascriptInterface方法，远程攻击者可以通过使用Java Reflection API利用该漏洞执行任意Java对象的方法。
> 2.WebView在布局文件中的使用：webView写在其他容器中时，在整个Activity被销毁的时候，除了要让WebView的父布局调用removeView移除WebView之外，还要在这之后调用WebView的 **removeAllViews和destroy**才能真正的移除WebView，不导致内存泄露的情况发生。
> 3.jsbridge: 实现远端JS与本地Native代码的相互调用
> 4.在判断网页是否加载完毕的时候，尽量 **不要使用**webviewClient.onPageFinished，因为如果网页在加载完成之前产生了跳转，那么这个方法会被回调无数次。可以使用WebChromeClient.onProgressChanged来判断。 
> 5.后台耗电: 当我们的App开启WebView去加载网页的时候，WebView会自己开启线程，如果我们没有正确处理WebView的内存回收，那么它残余的线程就会一直在后台运行，抢占了很多的资源。**粗暴的建议**是在Activity的onDestroy中直接调用System.exit(0)来关闭JVM。  
> 6.WebView硬件加速导致页面渲染的问题: 暂时关闭硬件加速   

##WebView的内存泄露问题   
> 原因：WebView中所有的操作都是在一个独立的线程中执行的，调用它的Activity没有办法控制这个线程，所以就会导致WebView的这个线程一直持有Activity资源，导致Activity不能被回收，造成内存泄露(原理上跟匿名内部类持有外部类的引用导致外部类无法被回收的情况类型)   
###解决方案   
> 1.独立进程，简单暴力，不过可能涉及到进程间通信   
> 2.动态添加WebView，对传入WebView中使用的Context使用弱引用，动态添加的意思是在布局创建一个ViewGroup来防止WebView，Activity创建时add进来，在Activity停止时remove掉。  