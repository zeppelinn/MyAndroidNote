# OkHttp   
> OkHttp是Square公司开源的轻量级框架   
> **同步请求** 使用案例    
>> ```Java   
>> //一个同步请求案例
>> public String OkHttpGet() throws Exception {
>>     //实例化OkHttp对象，既可以直接new来使用默认属性配置，也可以通过OkHttpClient的Build模式来添加自定义属性
>>     OkHttpClient client = new OkHttpClient();
>> 
>>     Request request = new Request.Builder()
>>             .url(url)
>>             .build();
>> 
>>     Response response = null;
>>     try {
>>     //client.newCall(request)，该方法返回一个Call对象，用来执行execute方法
>>         response = client.newCall(request).execute();
>>     } catch (IOException e) {
>>         e.printStackTrace();
>>     }
>>     return response.body().string();
>> }
>> ```      
---  
>>```Java   
>> //Call内部的execute方法(RealCall.java)
>> //将所有的同步请求添加到同步请求队列当中
>> @Override public Response execute() throws IOException {
>>     synchronized (this) {
>>          //executed标志位标识当前Call请求已经被执行，如果再次调用就会抛出异常，所以每个call请求都只会执行一次
>>         if (executed) throw new IllegalStateException("Already Executed");
>>         executed = true;
>>     }
>>     captureCallStackTrace();
>>     eventListener.callStart(this);
>>     try {
>>          //获取调度器dispatcher，dispatcher在默认构造器中已经被初始化
>>          //
>>         client.dispatcher().executed(this);
>>          //不论是同步请求还是异步请求，最终都会调用下面getResponseWithInterceptorChain这个方法
>>          //getResponseWithInterceptorChain是一个拦截器链，它可以对网络请求进行监听、重写等操作
>>          //一个拦截器链由若干个拦截器组成，每当一个拦截器执行完毕，它都会去执行它所处链中的下一个拦截器
>>         Response result = getResponseWithInterceptorChain();
>>         if (result == null) throw new IOException("Canceled");
>>         return result;
>>     } catch (IOException e) {
>>         eventListener.callFailed(this, e);
>>         throw e;
>>     } finally {
>>          //一个同步请求请求完毕之后，需要将请求从dispatcher调度器中移除
>>         client.dispatcher().finished(this);
>>     }
>> }
>>```   
---  
>>```Java   
>>  //(Dispatcher.java)
>> synchronized void executed(RealCall call) {
>>      //可以看出，上面在执行了Dispatcher的executed方法之后，实际上是将同步请求对象添加到了一个双向队列中，这也是一个顺序执行的过程
>>      runningSyncCalls.add(call);
>> }
>>```   
---   
>> **异步请求** 使用案例   
>> ```Java
>> public void OkHttpSysGet() {
>>     String url = "http://wwww.baidu.com";
>>     OkHttpClient okHttpClient = new OkHttpClient();
>>     final Request request = new Request.Builder()
>>             .url(url)
>>             .get()//默认就是GET请求，可以不写
>>             .build();
>>     //初始化请求与同步类似
>>     Call call = okHttpClient.newCall(request);
>>     //进行异步网络请求
>>     call.enqueue(new Callback() {
>>         @Override
>>         public void onFailure(Call call, IOException e) {
>>             Log.d("okHttp", "onFailure: ");
>>         }
>>          @Override
>>         public void onResponse(Call call, Response response) throws IOException {
>>               Log.d("okHttp", "onResponse: " + response.body().string());
>>         }
>>     });
>> }
>> ```   
---   
>> ```Java   
>> //(RealCall.java)
>> @Override public void enqueue(Callback responseCallback) {
>>      //锁住当前RealCall对象
>>     synchronized (this) {
>>          //executed表示当前RealCall请求是否被执行过
>>         if (executed) throw new IllegalStateException("Already Executed");
>>         executed = true;
>>     }
>>     captureCallStackTrace();
>>     eventListener.callStart(this);
>>     //添加异步请求到异步请求队列中，AsyncCall实际上就是一个Runnable
>>     client.dispatcher().enqueue(new AsyncCall(responseCallback));
>> }
>> ```   
---   
>> ```Java  
>> //像异步请求队列中添加请求(Dispatcher.java)
>> synchronized void enqueue(AsyncCall call) {
>>      //首先判断当前异步请求队列时候超过最大请求数
>>     if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
>>          //如果没有超过最大请求数，则将请求添加到请求队列中并执行
>>         runningAsyncCalls.add(call);
>>          //添加到线程池中进行处理
>>         executorService().execute(call);
>>     } else {
>>          //如果已经超过最大请求个数，则将该异步请求添加到一个专门用来存放等待执行的队列中
>>         readyAsyncCalls.add(call);
>>     }
>> }
>> ```   
---
>> ```Java
>> //在线程池中处理异步请求(Dispatcher.java)
>> public synchronized ExecutorService executorService() {
>>     if (executorService == null) {
>>          /**
>>            * ThreadPoolExecutor线程池对象的创建   
>>            * @param 核心线程数量，表示保持在线程池当中的线程数量，如果为0则表示当前线程池不会保留任何空闲的线程，所有空闲的线程在等待一段时间之后就会停止   
>>            * @param 表示当前线程池中可以容纳的最大线程数量   
>>            * @param 表示空闲线程(线程数大于核心线程数量)活跃时间，单位为秒，这里为60，说明在该线程池中如果一个线程出现空闲，那么60秒之后它就会被回收   
>>            * @param 同步的线程等待队列，在一个请求还没有从该队列中出队时，其他请求是不能够入队的，可以处理高频的请求
>>            * @param 线程工厂，创建守护线程，用来处理Dispatcher
>>            */
>>         executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
>>             new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
>>     }
>>     return executorService;
>> }
>> ```   
---   
>> ```Java   
>> // 异步请求中线程池处理线程的方法(ThreadPoolExecutor.java)
>> public void execute(Runnable command) {
>>     if (command == null)
>>         throw new NullPointerException();
>>     int c = ctl.get();
>>     // 新过来一个任务，如果当前核心数为0，不创建核心线程，而是将任务添加到队列中等待执行
>>     if (workerCountOf(c) < corePoolSize) {
>>         if (addWorker(command, true))
>>             return;
>>         c = ctl.get();
>>     }
>>     //如果当前线程池中正好有线程空闲，就直接接收任务
>>     if (isRunning(c) && workQueue.offer(command)) {
>>         int recheck = ctl.get();
>>         //如果当前线程池中所有线程工作饱和，就会入队失败
>>         if (! isRunning(recheck) && remove(command))
>>             reject(command);
>>         else if (workerCountOf(recheck) == 0)
>>             //创建并启动新线程
>>             addWorker(null, false);
>>     }
>>     else if (!addWorker(command, false))
>>         reject(command);
>> }
>> ```      
---   
> 在OkHttp中，所有的请求并不是由线程池来直接管理的，线程池在OkHttp中只起到 **辅助缓存请求**的作用，真正负责管理请求的是 **Dispatcher**      
>> Dispatcher   
>> 1. 维护请求状态   
>> 2. 在维护请求状态的同时，也会维护线程池进行同步请求和异步请求的操作   
> 源码分析(Dispatcher.java)  
>> 在Dispatcher内部维护了三个队列，分别为：   
>> + 异步等待队列 Deque<AsyncCall> readyAsyncCalls   
>> + 异步执行队列 Deque<AsyncCall> runningAsyncCalls   
>> + 同步执行队列 Deque<RealCall> runningSyncCalls   
>> 在上面的AsyncCall(RealCall.java)中的execute方法中可以看到，不管Dispatcher的调度成功还是失败，最终都会调用Dispatcher的finish方法将当前请求从调度器中移除。   
>> ```Java
>> //(Dispatcher.java)finish最终调用finished方法来移除请求
>> private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
>>     int runningCallsCount;
>>     Runnable idleCallback;
>>     synchronized (this) {
>>         if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
>>         if (promoteCalls) promoteCalls();
>>         runningCallsCount = runningCallsCount();
>>         idleCallback = this.idleCallback;
>>     }
>>     if (runningCallsCount == 0 && idleCallback != null) {
>>         idleCallback.run();
>>     }
>> }
>> //根据当前执行队列中请求的数量对总的请求数量进行调整
>> private void promoteCalls() {
>>     //判断当前请求的数量是否已经超过最大请求数量，超过就返回
>>     if (runningAsyncCalls.size() >= maxRequests) return; // Already running max capacity.
>>     //等待队列中没有请求
>>     if (readyAsyncCalls.isEmpty()) return; // No ready calls to promote.
>>     //遍历等待队列
>>     for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
>>         AsyncCall call = i.next();
>>         //判断当前正在请求的数量是否小于主机所能接受的最大请求数
>>         if (runningCallsForHost(call) < maxRequestsPerHost) {
>>              //如果小于，表示主机还能接受请求，那么就将等待队列中的操作取出执行，并将其从队列中移除
>>              i.remove();
>>              runningAsyncCalls.add(call);
>>              executorService().execute(call);
>>         }
>>         if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
>>     }
>> }
>> ```   





# Retrofit   
> Retrofit是RESTFul风格的HTTP网络请求框架的封装   
> 它的网络请求的工作本质上是由OkHttp完成的，而Retrofit仅仅负责 **网络请求接口的封装以及对OkHttp请求返回结果的解析**   
> 使用方法：   
>> 1. 创建描述网络请求的接口   
>> 2. 创建Retrofit实例   
>> 3. 创建网络请求接口实例对象并配置网络请求参数   
>> 4. 发送网络请求  
>> 5. 处理服务器返回的数据   
---   
> 创建描述网络请求接口:   
>> ```Java
>> //将Http请求抽象成Java接口
>> public interface retrofit_interface {
>>     //通过注解描述网络请求参数
>>     //GET中的参数表示路由（Retrofit对象在实例化时会指定baseUrl，比如我们通过get的方式访问http://localhost:3000/userInfo，那么baseURL就是http://localhost:3000/，而路由就是userInfo）
>>     @GET("部分URL")
>>     Call<Person> getCall();
>> }
>> ```   
---   
>> ```Java
>> //Retrofit对象的实例化
>> Retrofit retrofit = new Retrofit.Builder()
>>             .baseUrl("http://www.sina.com/") // 设置网络请求的Url地址
>>             .addConverterFactory(GsonConverterFactory.create()) // 设置数据解析器
>>             .addCallAdapterFactory(RxJavaCallAdapterFactory.create()) // 支持RxJava平台
>>             .build();
---   
>> ```Java
>> // 创建 网络请求接口 的实例
>> retrofit_interface request = retrofit.create(retrofit_interface.class);
>> ```
---   
>> ```Java
>> //对 发送请求 进行封装，获取Call对象的实例
>> Call<Person> call = request.getCall();
>> ```   
---   
>> ```Java
>> //调用call对象来进行网络请求
>> public void callRequest() {
>>     //发送网络请求(异步)
>>     call.enqueue(new Callback<Person>() {
>>         //请求成功时回调
>>         @Override
>>         public void onResponse(Call<Person> call, Response<Person> response) {
>>             // 对返回数据进行处理
>>             response.body();
>>         }
>> 
>>         //请求失败时候的回调
>>         @Override
>>         public void onFailure(Call<Person> call, Throwable throwable) {
>>             System.out.println("连接失败");
>>         }
>>     });
>> }
>> ```   

---   
> Retrofit使用注意事项   
>> + 解析网络请求接口的注解配置网络请求参数   
>> + **动态代理** 生成 **网络请求对象**   
>> + **网络请求适配器** 将 **网络请求对象** 进行平台分配（Android、RxJava、Java8等）   
>> + **网络请求执行器** 发送网络请求   
>> + **数据转换器** 解析服务器返回的数据   
>> + **回调执行器** 切换线程   
>> + 主线程处理返回结果   
---   
> 源码分析（Retrofit.java）
>> **构造器**
>> ```Java   
>>  //用来存储网络请求配置的对象(请求方法、数据转换器、网络请求工厂、网络请求适配器、URL等等)
>>  private final Map<Method, ServiceMethod> serviceMethodCache = new LinkedHashMap<>();
>>  // 生产网络请求器Call的对象
>>  private final okhttp3.Call.Factory callFactory;
>>  private final HttpUrl baseUrl;
>>  //放置数据转化器工厂(GSON)
>>  private final List<Converter.Factory> converterFactories;
>>  //放置网络请求适配器(生产CallAdapter)
>>  private final List<CallAdapter.Factory> adapterFactories;
>>  //回调方法执行器
>>  private final Executor callbackExecutor;
>>  //标志位 判断是否提前对接口中的注解进行转换
>>  private final boolean validateEagerly;
>>
>>  Retrofit(okhttp3.Call.Factory callFactory, HttpUrl baseUrl,
>>      List<Converter.Factory> converterFactories, List<CallAdapter.Factory> adapterFactories,
>>      Executor callbackExecutor, boolean validateEagerly) {
>>    this.callFactory = callFactory;
>>    this.baseUrl = baseUrl;
>>    this.converterFactories = unmodifiableList(converterFactories); // Defensive copy at call site.
>>    this.adapterFactories = unmodifiableList(adapterFactories); // Defensive copy at call site.
>>    this.callbackExecutor = callbackExecutor;
>>    this.validateEagerly = validateEagerly;
>>  }
>> ```   
---   
>> ```Java   
>> // 调用build方法完成整个Retrofit对象的创建
>> public Retrofit build() {
>>     if (baseUrl == null) {
>>      throw new IllegalStateException("Base URL required.");
>>     }
>>     okhttp3.Call.Factory callFactory = this.callFactory;
>>     if (callFactory == null) {
>>          //如果在实例化Retrofit对象的之后没有指定网络请求适配器，就会默认使用OkHttp提供的网络请求适配器
>>          callFactory = new OkHttpClient();
>>     }
>>     Executor callbackExecutor = this.callbackExecutor;
>>     if (callbackExecutor == null) {
>>         callbackExecutor = platform.defaultCallbackExecutor();
>>     }
>> 
>>     // Make a defensive copy of the adapters and add the default Call adapter.
>>     List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
>>     adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));
>> 
>>     // Make a defensive copy of the converters.
>>     List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);
>>     return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
>>         callbackExecutor, validateEagerly);
>> }
>> ```    
---   
>> 网络请求接口实例的创建(Retrofit.java)   
>> ```Java
>>  public <T> T create(final Class<T> service) {
>>    Utils.validateServiceInterface(service);
>>    // 上面提到的是否需要解析注解的标志位
>>    // eagerlyValidateMethods会遍历网络请求接口中的所有注解，并将其转换为网络请求配置信息（ServiceMethod对象），然后存在一个LinkHashMap中
>>    if (validateEagerly) {
>>      eagerlyValidateMethods(service);
>>    }
>>    //动态代理模式，动态的生成网络请求接口
>>    // 当我们在调用网络请求接口的getCall方法来获取Call对象的时候，Retrofit就会将所有的网络请求配置发送到这里来进行解析
>>    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
>>        new InvocationHandler() {
>>          private final Platform platform = Platform.get();
>>          @Override 
>>          public Object invoke(Object proxy, Method method, Object... args)
>>              throws Throwable {
>>            // If the method is a method from Object then defer to normal invocation.
>>            if (method.getDeclaringClass() == Object.class) {
>>              return method.invoke(this, args);
>>            }
>>            if (platform.isDefaultMethod(method)) {
>>              return platform.invokeDefaultMethod(method, service, proxy, args);
>>            }
>>            // 读取网络请求接口中的方法
>>            ServiceMethod serviceMethod = loadServiceMethod(method);
>>            // 创建OkHttpCall网络请求对象，将网络请求配置和参数传入
>>            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
>>            return serviceMethod.callAdapter.adapt(okHttpCall);
>>          }
>>        });
>>  }
>> ```    
---
>> ```Java   
>>  //ServiceMethod中的一些成员变量(网络请求配置信息)
>>  final okhttp3.Call.Factory callFactory;  //Call请求的工厂，生产OkHttp中的call对象
>>  final CallAdapter<?> callAdapter;       // 网络请求适配器(用来适配Android、RxJava、Java8等不同的平台)
>>  private final HttpUrl baseUrl;
>>  private final Converter<ResponseBody, T> responseConverter;  //负责将服务器返回的数据转换为泛型T
>>  private final String httpMethod;
>>  private final String relativeUrl;
>>  private final Headers headers;
>>  private final MediaType contentType;    //http报文body的类型
>>  private final boolean hasBody;
>>  private final boolean isFormEncoded;
>>  private final boolean isMultipart;
>>  private final ParameterHandler<?>[] parameterHandlers;  //方法参数的处理器，负责解析接口中定义的方法和参数，并根据此定义请求的配置
>> ```   
---   
>> ```Java   
>> //ServiceMethod的创建方法(ServiceMethod.java)
>> public Builder(Retrofit retrofit, Method method) {
>>   this.retrofit = retrofit;
>>   this.method = method;
>>   //获取网络请求接口中的注解
>>   this.methodAnnotations = method.getAnnotations();
>>   //获取网络请求接口中的参数类型
>>   this.parameterTypes = method.getGenericParameterTypes();
>>   //获取网络请求接口中的注解的内容
>>   this.parameterAnnotationsArray = method.getParameterAnnotations();
>> }
>> ```   
---   
> **Retrofit发送请求的过程**   
>> 1. 对网络请求接口的方法中的每个参数利用对应的ParameterHandler进行解析   
>> 2. 使用OkHttp的Request发送网络请求    
>> 3. 对返回的数据使用之前设置的数据转换器(GsonConverterFactory)解析返回的数据   
>> 4. 进行线程切换从而在主线程中处理返回的数据结果   
---   
> **Retrofit总结**   
>> 1. Retrofit将Http请求抽象成Java接口   
>> 2. 在接口中使用注解描述和配置网络请求参数   
>> 3. 使用动态代理的方式，动态地将网路请求接口的注解解析成HTTP请求    
>> 4. Retrofit最终还是通过OkHttp来进行网络请求的，它只是一个RESTFUL风格的对OkHttp进行封装的框架   





# RxJava   
使用RxJava的基本步骤   
1. 创建被观察者(Observable)和生产事件   
2. 创建观察者(Observer)并定义响应事件的行为   
3. 通过订阅(Subscribe)来连接观察者和被观察者   

> **RxJava应用案例**   
>> ```Java
>> public class RxjavaDemo {
>>     // 1. 创建被观察者 Observable 对象
>>     Observable<Integer> observable = Observable.create(new ObservableOnSubscribe<Integer>() {
>> 
>>         // 2. 在复写的subscribe（）里定义需要发送的事件
>>         @Override
>>         public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
>>             emitter.onNext(1);
>>             emitter.onNext(2);
>>             emitter.onNext(3);
>>             emitter.onComplete();
>>         }
>>     });
>> 
>>     // 1. 创建观察者 （Observer ）对象
>>     Observer<Integer> observer = new Observer<Integer>() {
>>         // 2. 创建对象时通过对应复写对应事件方法 从而 响应对应事件
>> 
>>         // 观察者接收事件前，默认最先调用复写 onSubscribe（）
>>         @Override
>>         public void onSubscribe(Disposable d) {
>>             Log.d("RxJava", "开始采用subscribe连接");
>>         }
>> 
>>         // 当被观察者生产Next事件 & 观察者接收到时，会调用该复写方法 进行响应
>>         @Override
>>         public void onNext(Integer value) {
>>             Log.d("RxJava", "对Next事件作出响应" + value);
>>         }
>> 
>>         // 当被观察者生产Error事件& 观察者接收到时，会调用该复写方法 进行响应
>>         @Override
>>         public void onError(Throwable e) {
>>             Log.d("RxJava", "对Error事件作出响应");
>>         }
>> 
>>         // 当被观察者生产Complete事件& 观察者接收到时，会调用该复写方法 进行响应
>>         @Override
>>         public void onComplete() {
>>             Log.d("RxJava", "对Complete事件作出响应");
>>         }
>>     };
>> 
>>     // 1. 创建观察者 （Observer ）对象
>>     Subscriber<String> subscriber = new Subscriber<String>() {
>> 
>>         // 2. 创建对象时通过对应复写对应事件方法 从而 响应对应事件
>> 
>>         // 当被观察者生产Next事件 & 观察者接收到时，会调用该复写方法 进行响应
>>         @Override
>>         public void onNext(String value) {
>>             Log.d("1", "对Next事件作出响应" + value);
>>         }
>> 
>>         // 当被观察者生产Error事件& 观察者接收到时，会调用该复写方法 进行响应
>>         @Override
>>         public void onError(Throwable e) {
>>             Log.d("1", "对Error事件作出响应");
>>         }
>> 
>>         // 当被观察者生产Complete事件& 观察者接收到时，会调用该复写方法 进行响应
>>         @Override
>>         public void onCompleted() {
>>             Log.d("1", "对Complete事件作出响应");
>>         }
>> 
>>         @Override
>>         public void onStart() {
>>             super.onStart();
>>         }
>>     };
>> 
>>     public void doSubsribe() {
>>         observable.subscribe(observer);
>>     }
>> }
>> ```


# RxJava+Retrofit实现轮询和handler的实现方式   
>定义Retrofit网络请求接口(LoopRequest_interface.java)   
>> ```Java
>> public interface LoopRequest_interface {
>>     @GET("ajax.php?a=fy&f=auto&t=auto&w=hi%20world")
>>     Observable<Person> getCall();
>> }
>> ```
---   
>使用RxJava+Retrofit实现轮询请求(LoopRequestDemo.java)   
>> ```Java   
>> public class LoopRequestDemo {
>>     public void deLoopRequest(){
>>         /*
>>          * 步骤1：采用interval（）延迟发送
>>          * 注：此处主要展示无限次轮询，若要实现有限次轮询，仅需将interval（）改成intervalRange（）即可
>>          **/
>>         Observable.interval(2,1, TimeUnit.SECONDS)
>>                  /*
>>                   * 步骤2：每次发送数字前发送1次网络请求（doOnNext（）在执行Next事件前调用）
>>                   * 即每隔1秒产生1个数字前，就发送1次网络请求，从而实现轮询需求
>>                   **/
>>                 .doOnNext(new Consumer<Long>() {
>>                     @Override
>>                     public void accept(Long integer) throws Exception {
>> 
>>                  /*
>>                   * 步骤3：通过Retrofit发送网络请求
>>                   **/
>>                         // a. 创建Retrofit对象
>>                         Retrofit retrofit = new Retrofit.Builder()
>>                                 .baseUrl("http://fy.iciba.com/") // 设置 网络请求 Url
>>                                 .addConverterFactory(GsonConverterFactory.create()) //设置使用Gson解析(记得加入依赖)
>>                                 .addCallAdapterFactory(RxJavaCallAdapterFactory.create()) // 支持RxJava
>>                                 .build();
>> 
>>                         // b. 创建 网络请求接口 的实例
>>                         LoopRequest_interface request = retrofit.create(LoopRequest_interface.class);
>> 
>>                         // c. 采用Observable<...>形式 对 网络请求 进行封装
>>                         Observable<Person> observable = request.getCall();
>>                         // d. 通过线程切换发送网络请求
>>                         observable.subscribeOn(Schedulers.io())               // 切换到IO线程进行网络请求
>>                                 .observeOn(Schedulers.newThread())  // 切换回到主线程 处理请求结果
>>                                 .subscribe(new Observer<Person>() {
>>                                     @Override
>>                                     public void onSubscribe(Disposable d) {
>>                                     }
>> 
>>                                     @Override
>>                                     public void onNext(Person result) {
>>                                         // e.接收服务器返回的数据
>>                                     }
>> 
>>                                     @Override
>>                                     public void onError(Throwable e) {
>>                                     }
>> 
>>                                     @Override
>>                                     public void onComplete() {
>> 
>>                                     }
>>                                 }
>>                          );
>> 
>>                     }
>>                 }).subscribe(new Observer<Long>() {
>>             @Override
>>             public void onSubscribe(Disposable d) {
>> 
>>             }
>>             @Override
>>             public void onNext(Long value) {
>> 
>>             }
>> 
>>             @Override
>>             public void onError(Throwable e) {
>>             }
>> 
>>             @Override
>>             public void onComplete() {
>>             }
>>         });
>>     }
>> }
>> ```   



# RxJava缓存策略   
> LRU(Least Recently Used)最近最少使用
>> LRU算法是系统在内存不足的情况下进行内存回收的一种策略，系统会将最近并且是最少使用的内存进行回收。一般在Java中我们可以使用 **LinkedHashMap**来管理数据集合，它内部就采用了LRU算法。   
>> Android在3.1之后提出了一种内存缓存类LruCache，它实现的主要原理就是将最近使用的对象用强引用的方式存储在一个LinkedHashMap当中。   
>> 在LruCache中维护了一个缓存对象列表(LinkedHashMap)，而列表中元素的排列方式则是按照访问的顺序实现的。最少访问的对象会被放置在列表的 **队尾**。而在内存不足的情况出现的时候，就会从队尾开始删除对象。     
---   
> 使用RxJava建立高效的数据获取方式（依次从磁盘、内存和网络请求中获取数据）   
>> ```Java
>> public class CacheRxjavaDemo {
>>     String memoryCache = null;
>>     String diskCache = "从磁盘缓存中获取数据";
>>     /*
>>      * 设置第1个Observable：检查内存缓存是否有该数据的缓存
>>      **/
>>     Observable<String> memory = Observable.create(new ObservableOnSubscribe<String>() {
>>         @Override
>>         public void subscribe(ObservableEmitter<String> emitter) throws Exception {
>>             // 先判断内存缓存有无数据
>>             if (memoryCache != null) {
>>                 // 若有该数据，则发送
>>                 emitter.onNext(memoryCache);
>>             } else {
>>                 // 若无该数据，则直接发送结束事件
>>                 emitter.onComplete();
>>             }
>>         }
>>     });
>> 
>>     /*
>>      * 设置第2个Observable：检查磁盘缓存是否有该数据的缓存
>>      **/
>>     Observable<String> disk = Observable.create(new ObservableOnSubscribe<String>() {
>>         @Override
>>         public void subscribe(ObservableEmitter<String> emitter) throws Exception {
>>             // 先判断磁盘缓存有无数据
>>             if (diskCache != null) {
>>                 // 若有该数据，则发送
>>                 emitter.onNext(diskCache);
>>             } else {
>>                 // 若无该数据，则直接发送结束事件
>>                 emitter.onComplete();
>>             }
>>         }
>>     });
>> 
>>     /*
>>      * 设置第3个Observable：通过网络获取数据
>>      **/
>>     Observable<String> network = Observable.just("从网络中获取数据");
>>     public void doCache() {
>>         // 将三个被观察者对象按照优先级排列加入
>>         Observable.concat(memory, disk, network)
>>                 // 2. 通过firstElement()，从串联队列中取出并发送第1个有效事件（Next事件），即依次判断检查memory、disk、network
>>                 .firstElement()
>>                 // 3. 观察者订阅
>>                 .subscribe(new Consumer<String>() {
>>                     @Override
>>                     public void accept(String s) throws Exception {
>>                         Log.d("cache", "最终获取的数据来源 =  " + s);
>>                     }
>>                 });
>>     }
>> }
>> ```














# 图片加载框架Glide   
> 基本使用方法   
>> ```Java
>> //简单使用
>> Glide.with(this)
>>      .load("url")
>>      .into(imageview);
>> //更多配置参数
>> Glide.with(this)   //Context类型，Application、Activity或者Fragment的this都可以，Glide会根据我们传入with当中的上下文所属的Activity或者Fragment的生命周期来自动执行加载、销毁等动作。
>>      .load("url")  //图片加载地址(网络资源，本地资源，甚至二进制流)
>>      .placeholder(R.drawable.ic_launcher_background)
>>      .error(R.drawable.ic_launcher_background)
>>      .override(50, 50)  //Glide不会直接将整个图片的完整尺寸加载到内存中，我们可以在这里定义引用图片的宽高，Glide会帮助我们对要显示的图片进行内存优化
>>      .fitCenter()  //缩放图像，使图像的宽高都小于imageView所定义的最大边界范围
>>      .centerCrop()  //缩放图像，会让图像填满imageView，但是会裁剪掉超出imageView边界范围的部分
>>      .skipMemoryCache(true)  //跳过或者不跳过内存缓存，参数如果为true，则表示不会将图片添加到内存缓存当中。Glide会默认将所有的图片都放到内存缓存当中。需要注意的是，如果两次加载了同一张图片，第一次使用了内存缓存，而第二次即使调用了skipMemoryCache(true)，Glide仍然会将该图片存储到内存中。所以在加载图片的时候，我们需要保证图片的
>>      .diskCacheStrategy(DiskCacheStrategy.NONE)  //硬盘缓存策略，如果我们关闭了内存缓存策略，Glide则会将图片加入到磁盘缓存当中。
>>      .priority(Priority.HIGH)  //优先级，如果需要在同一事件内加载多个图像，而每种图像的等级不一样，这里就可以通过优先级的次序来对图像加载的先后顺序进行排序。
>>      .into(imageview);
>> ```   
---   
>> ```Java
>>  //磁盘缓存策略中的枚举类型(DiskCacheStrategy.java)
>>  /** Caches with both {@link #SOURCE} and {@link #RESULT}. */
>>  // 缓存所有版本的图片
>>  ALL(true, true),
>>  /** Saves no data to cache. */
>>  //啥都不缓存,默认
>>  NONE(false, false),
>>  /** Saves just the original data to cache. */
>>  //仅仅缓存图片的原始备份，其他缩放或者处理过的图片不会进行缓存
>>  SOURCE(true, false),
>>  /** Saves the media item after all transformations to cache. */
>>  //与上面相反，仅仅返回最终的被处理过的图像
>>  RESULT(false, true);
>> ```   







# 注入框架ButterKnife   
> 1. 简介：Butterknife是一个依托Java的注解机制来实现辅助代码生成的框架。需要注意的是Butterknife中用到的注解并不是在运行时反射的，而是在编译的时候就已经生成的类   
> 2. 基本使用   
>> ```Java
>>    //绑定一个View。(View不能为private或者static)
>>    @BindView(R.id.textview)
>>    TextView mText;
>>
>>    //绑定多个View。
>>    @BindViews({ R2.id.button1, R2.id.button2,  R2.id.button3})
>>    public List<Button> buttonList;
>>    @Override
>>    protected void onCreate(Bundle savedInstanceState) {  
>>        super.onCreate(savedInstanceState);  
>>        setContentView(R.layout.activity_main);  
>>        ButterKnife.bind(this);  
>>        buttonList.get(0).setText( "hello 1 ");  
>>        buttonList.get(1).setText( "hello 2 ");  
>>        buttonList.get(2).setText( "hello 3 ");  
>>    }
>>
>>    //给单个view添加点击事件
>>    @OnClick(R.id.textview)
>>    public ovid onTextClick(){
>>        Toast.makeText(this, 'onTextClicked', Toast.LENGTH_SHORT).show();
>>    }
>>    
>>    //给多个view添加点击事件
>>    @OnClick({R.id.textView1, R.id.textView2})
>>    public void onMultiTextClick(View view){
>>        switch(view.getId()){
>>            //...
>>        }
>>    }
>> ```   
>     
> 3. Butterknife原理   
>> + 在编译的时候，Butterknife会扫描Java代码中所有的Butterknife注解   
>> + 在Butterknife中，它会调用ButterKnifeProcessor来生成Java类`<className>$$ViewBinder`   
>> + 调用bind方法加载生成的ViewBinder类，为控件动态的注入注解中的属性和绑定事件，这也是为什么声明控件的时候不能够将其设置为private或者static，因为这样设置之后就只能通过反射来获取控件及其注解的属性了。      
>> + 注：首先Butterknife **并没有**使用反射的方式实现读取Activity或者Fragment中的所有注解及其属性，因为这样会占用Activity大量的性能；同时反射也会产生许多的临时变量，进而引起垃圾回收，而大量频繁的垃圾回收是会造成UI卡顿的。Butterknife使用的是Java的Annotation Processing(注解处理技术)，它的原理是在编译Java代码的时候读入注解，并根据注解生成新的Java代码，最后再将其编译成为Java字节码。   