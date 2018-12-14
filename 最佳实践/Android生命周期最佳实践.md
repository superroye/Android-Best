# Android生命周期最佳实践

> Android的Activity/Fragment都有自己的生命周期，我们调用第三方组件，UI操作，或者后台任务，都要注意Android组件的生命周期的判断和正确利用，避免不必要的业务错误或崩溃的发生。

下面有几种生命周期的利用场景。



- ## 生命周期解耦

  一般利用生命周期，在Activity/Fragment 的对应生命周期函数调用相关接口：

  ```java
  class MainActivity extends Activity {
      
      void onStart(){
          VideoApi.start();
      }
      
      void onResume(){
          VideoApi.autoPlay();
      }
      
      void onPause(){
          VideoApi.pause();
      }
      
      void onStop(){
          VideoApi.stop();
      }
      
      void onDestroy(){
          VideoApi.release();
      }
  }
  ```

  咋一看，也没毛病，假设还有一些分享组件，上报组件需要依赖生命周期，那所有第三方组件的调用逻辑都塞到Activity里面，Activity就膨胀的不可维护了。



我们尝试把组件逻辑抽出来，并且实现一个LifecycleCallback给Activity调用，专门负责收集生命周期相关回调。这样的话，似乎组件就解放了，不再依赖Activity了。



  事实上，[Android-arch](https://developer.android.google.cn/topic/libraries/architecture/)  架构，已经包含了我们的所需，可以在任何地方实现LifecycleObserver，就可以监控Acvitity生命周期，使用这种方式，代码变得更优雅，扩展也更加轻松。

  ```java
  class MainActivity extends Activity {
      
      void onCreate(){
          getLifecycle().addObserver(lifecycleObserver);
      }
  ｝
      
  class lifecycleObserver implements LifecycleObserver{
      
      @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
      void onDestroy() {
          if (mCompositeDisposable != null) {
              mCompositeDisposable.dispose();
          }
      }
  }
  ```

- ## 非激活状态停止调用

  > Activity组件，包含oncreate、onstart、onresume、onpause、onstop、ondestroy状态，特别是异步调用场景，Activity在destroy状态时调用UI操作，就会报token非法不可用。我们要避免这种情况，我们只希望可以有一种统一处理方案。因为无休止的判断，真的很糟糕！

  下面有3种情况，值得注意。

  ### 1、子线程调用UI操作

  在子线程，很容易有这种操作，比如:

  ```java
  activity.runUiThread({
  
      if(!activity.isFinishing() || activity.isDestroyed()){
  		retrun；
      }
  	new Dialog(activity).show();
  });
  
  or new Handler(Looper.mainLooper()).post(Runnable);
  ```

  runUiThread也是用Handler实现，只是在activity类里面，更方便调用罢了。

  我们也可以利用handler封装一个通用的runOnUi，并且利用代理模式做到一样的体验，还能自动判断生命周期。

  ```java
  class Task {
      static Handler mainHandler = new Handler(Looper.mainLooper())；
          
      static runOnUi(Activity activity, Runnable call){
          mainHandler.post(new Runnable(){
              if(!activity.isFinishing() || activity.isDestroyed()){
  				retrun；
      		}
              call.run();
          });
      }
  }
  ```

  似乎没毛病，也很简洁！

  然而有点尴尬，它并没有覆盖更多场景。

  假设有些特殊场景，需要在Resume状态的时候才执行，或者至少onCreate状态的时候执行，或者初始化就执行。

  那也可以这么干！

  ```java
  /**
   ** 里面用了Rxjava2实现，并且简单判断了生命周期状态，这种实现更灵活更优雅
  **/
  RxTask.runOnMain()
  .scopeProvider(TaskLifecycleScopeProvider.from(context, Lifecycle.State.CREATED))
  .subscribe(context ->{
      //dosomething
      //dosomething
  });
  ```



    ### 2、执行异步操作，并通知UI线程

  这是个经典场景，调用本地内存，远程网络，或解析大JSON，都需要在后台线程执行完，再通知UI刷新。

  这里，我们用RxJava+autoDisposable（uber开源）来实现，autoDisposable能自动释放事件。

  ```java
  Observable.just(Param)
      .map({param->
          //get network data
      })
      .subscribeOn(AndroidSchedulers.mainThread())
  .as(autoDisposable(AndroidLifecycleScopeProvider.from(lifecycleOwner,Lifecycle.Event.ON_STOP)))
  .subscribe(new DefaultObserver<Long>() {
                      @Override
                      public void onNext(Long aLong) {
                          test();
                      }
  
                      @Override
                      public void onError(Throwable e) {
  
                      }
  
                      @Override
                      public void onComplete() {
                      }
  });
  ```

  这种场景有点不同，需要先执行后台线程，再通知UI，所以可以用上面的as操作做拦截UI处理。



  上述，看起来好像复杂了，异步任务，通常直接定义线程池也能实现

  ```java
  //do background task
  void doBg(Runnable task) {
      Executors.getIOExecutor().run({
          task.run();
  	});
  }
  
  //do UI update
  void doUI(Runnable task) {
      Executors.getUIExecutor().run({
          task.run();
  	});
  }
  
  doBg(new Runnable(){
      run(){
          //bg task
          doUI(new Runnable(){
              if(!activity.isDestroy()){
              	return;
          	}
              //do UI update
          });
      }
  });
  ```

  上面一个后台任务执行，并通知UI，还比较简单。要是多个后台任务呢？多任务执行完再通知UI，恐怕陷入嵌套地狱了。

  所以这才是Rxjava的优势所在，它能应对各种任务连接，任务组合，而并没增加复杂度。例如：

  ```java
  //一个异步事件
  Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> e) throws Exception {
                //bg task
                e.onNext("快餐（牛肉面）");//发送数据
                e.onComplete();//关闭发送数据，以后不能再使用e.onNext
            }
        }).subscribe( 
      new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {
                tv.setText(tv.getText()+"RxJava-开始送快餐"+"\n");
            }
  
            @Override
            public void onNext(String s) {
                tv.setText(tv.getText()+"RxJava-快餐送达:"+s+"\n");
            }
  
            @Override
            public void onError(Throwable e) {
                tv.setText(tv.getText()+"RxJava-送快餐出错"+"\n");
            }
  
            @Override
            public void onComplete() {
                tv.setText(tv.getText()+"RxJava-本次快餐送达完毕"+"\n");
            }
        }
  );
  
  //多个异步事件map(task1, task2) & reduce(result), 根据字符串异步返回MD5值，并合并结果（对比是否一致）
  Observable.just(resopnse1, resopnse2)
  .flatMap(new Function<Object, ObservableSource<String>>() {
  	public ObservableSource<String> apply(final Object resopnse) {
  		return Observable.create(new ObservableOnSubscribe<String>() {
  			public void subscribe(ObservableEmitter<String> e) {
  			    String md5 = getMd5();
  				e.onNext(String.valueOf(md5));
  				e.onComplete();
  		 }}).subscribeOn(Schedulers.io()).observeOn(Schedulers.computation());
  	}
  })
  .reduce(new BiFunction<String, String, String>() {
  	  public String apply(String o, String o2) {
  		  return o.equals(o2) ? "y" : "n";
  	  }
  }).subscribeOn(Schedulers.io()).observeOn(Schedulers.computation())
  .subscribe(observer);
  
  ```

  > Rxjava 执行异步事件，并通知执行结果。
  >
  > 主要步骤是：
  >
  > 1、创建事件
  >
  > 有Observable.create, just, from, using, zip等创建操作符
  >
  > 2、事件转换操作（可选）
  >
  > map，flatmap，as，compose等等
  >
  > 3、设置事件线程池和订阅线程池
  >
  > 4、订阅事件
  >
  > subscribe(observer)函数

  ### 3、在UI线程内拦截

  > 还有一种场景，在既定的线程环境里，调用Activity，只能判断Activity的生命周期有效性了

  ```java
  if(!activity.isDestroy()){
      return;
  }
  //do UI update
  ```

  ### 4、定时器/延时任务

  > 定时器任务，一般可以用handler循环调用实现，可以用timer实现，也可以用CountDownTimer实现，但是，你必须要跟随Activity/fragment释放，否则发生内存泄漏或崩溃。擦，这么干，也太麻烦了！

  这时候，又想起RxJava+autoDisposable，Observable.timer, Observable.interval操作能做延时事件和定时器，只需要添加autoDisposable绑定lifecycleOwner生命周期，就能自动释放事件。

  ```java
  Observable.interval(1, TimeUnit.SECONDS).subscribeOn(AndroidSchedulers.mainThread())
                  .as(autoDisposable(AndroidLifecycleScopeProvider.from(this, Lifecycle.Event.ON_DESTROY)))
                  .subscribe(new DefaultObserver<Long>() {
                      @Override
                      public void onNext(Long aLong) {
                          android.util.Log.d("", "interval========" + Thread.currentThread().getName()+" "+System.currentTimeMillis());
                      }
  
                      @Override
                      public void onError(Throwable e) {
                          
                      }
  
                      @Override
                      public void onComplete() {
                      }
                  });
  ```



  ### 5、任务约束和恢复任务

  > 有时候，网路不好或电量低的时候，需要暂停任务执行，等设备恢复，我们再恢复任务执行。
  >
  > 此时，其他异步架构都不好使。

  JobScheduler可以解决问题，但是据闻android 21，22版本还有bug。



  其实我想介绍**WorkManager**，(后面摘自Android-dev官网)

  WorkManager API可以轻松指定可延迟的异步任务以及何时运行它们。这些API允许您创建任务并将其交给WorkManager立即运行或在适当的时间运行。



  WorkManager根据设备API级别和应用程序状态等因素选择适当的方式来运行任务。如果WorkManager在应用程序运行时执行您的任务之一，WorkManager可以在您应用程序进程的新线程中运行您的任务。如果您的应用程序未运行，WorkManager会选择一种合适的方式来安排后台任务 - 具体取决于设备API级别和包含的依赖项，WorkManager可能会使用 [`JobScheduler`](https://developer.android.google.cn/reference/android/app/job/JobScheduler.html)，[Firebase JobDispatcher](https://github.com/firebase/firebase-jobdispatcher-android#user-content-firebase-jobdispatcher-)或[`AlarmManager`](https://developer.android.google.cn/reference/android/app/AlarmManager.html)。您无需编写设备逻辑来确定设备具有哪些功能并选择适当的API; 相反，您可以将您的任务交给WorkManager，让它选择最佳选项。

  # WorkManager基础知识

  使用WorkManager，您可以轻松设置任务并将其交给系统，以便在您指定的条件下运行。

  本概述介绍了最基本的WorkManager功能。在此页面中，您将学习如何设置任务，指定应运行的条件，并将其交给系统。您还将学习如何设置重复作业。

  有关更高级的WorkManager功能（如作业链和传递和返回值）的信息，请参阅 [WorkManager高级功能](https://developer.android.google.cn/topic/libraries/architecture/workmanager/advanced.html)。还有更多功能; 有关完整的详细信息，请参阅[WorkManager参考文档](https://developer.android.google.cn/reference/androidx/work/package-summary)。

  **注意：**要将WorkManager库导入Android项目，请参阅[向](https://developer.android.google.cn/topic/libraries/architecture/adding-components.html#workmanager)项目[添加组件](https://developer.android.google.cn/topic/libraries/architecture/adding-components.html#workmanager)。

  ## 类和概念

  WorkManager API使用几个不同的类。在某些情况下，您需要子类化其中一个API类。

  这些是最重要的WorkManager类：

  - [`Worker`](https://developer.android.google.cn/reference/androidx/work/Worker.html)：指定您需要执行的任务。WorkManager API包含一个抽象[`Worker`](https://developer.android.google.cn/reference/androidx/work/Worker.html)类。您可以扩展此类并在此处执行工作。
  - [`WorkRequest`](https://developer.android.google.cn/reference/androidx/work/WorkRequest.html)：代表一项单独的任务。[`WorkRequest`](https://developer.android.google.cn/reference/androidx/work/WorkRequest.html)对象至少 指定[`Worker`](https://developer.android.google.cn/reference/androidx/work/Worker.html)应该执行任务的类。但是，您还可以向[`WorkRequest`](https://developer.android.google.cn/reference/androidx/work/WorkRequest.html)对象添加详细信息 ，指定诸如运行任务的环境之类的内容。每个[`WorkRequest`](https://developer.android.google.cn/reference/androidx/work/WorkRequest.html)都有一个自动生成的唯一ID; 您可以使用ID执行取消排队任务或获取任务状态等操作。[`WorkRequest`](https://developer.android.google.cn/reference/androidx/work/WorkRequest.html)是一个抽象的类; 在您的代码中，您将使用其中一个直接子类， [`OneTimeWorkRequest`](https://developer.android.google.cn/reference/androidx/work/OneTimeWorkRequest.html)或 [`PeriodicWorkRequest`](https://developer.android.google.cn/reference/androidx/work/PeriodicWorkRequest.html)。
    - [`WorkRequest.Builder`](https://developer.android.google.cn/reference/androidx/work/WorkRequest.Builder.html)：用于创建[`WorkRequest`](https://developer.android.google.cn/reference/androidx/work/WorkRequest.html)对象的辅助类 。同样，你将使用其中一个子类，[`OneTimeWorkRequest.Builder`](https://developer.android.google.cn/reference/androidx/work/OneTimeWorkRequest.Builder.html)或 [`PeriodicWorkRequest.Builder`](https://developer.android.google.cn/reference/androidx/work/PeriodicWorkRequest.Builder.html)。
    - [`Constraints`](https://developer.android.google.cn/reference/androidx/work/Constraints.html)：指定对任务运行时间的限制（例如，“仅在连接到网络时”）。使用创建[`Constraints`](https://developer.android.google.cn/reference/androidx/work/Constraints.html)对象[`Constraints.Builder`](https://developer.android.google.cn/reference/androidx/work/Constraints.Builder.html)，并在创建之前 将其传递 [`Constraints`](https://developer.android.google.cn/reference/androidx/work/Constraints.html)给 。[`WorkRequest.Builder`](https://developer.android.google.cn/reference/androidx/work/WorkRequest.Builder.html)[`WorkRequest`](https://developer.android.google.cn/reference/androidx/work/WorkRequest.html)
  - [`WorkManager`](https://developer.android.google.cn/reference/androidx/work/WorkManager.html)：排队和管理工作请求。您将[`WorkRequest`](https://developer.android.google.cn/reference/androidx/work/WorkRequest.html) 对象传递[`WorkManager`](https://developer.android.google.cn/reference/androidx/work/WorkManager.html)给队列任务。 [`WorkManager`](https://developer.android.google.cn/reference/androidx/work/WorkManager.html)调度任务的方式是分散系统资源的负载，同时遵守您指定的约束。
  - [`WorkInfo`](https://developer.android.google.cn/reference/androidx/work/WorkInfo.html)：包含有关特定任务的信息。在 [`WorkManager`](https://developer.android.google.cn/reference/androidx/work/WorkManager.html)提供了[`LiveData`](https://developer.android.google.cn/reference/android/arch/lifecycle/LiveData.html)对每个 [`WorkRequest`](https://developer.android.google.cn/reference/androidx/work/WorkRequest.html)对象。所述[`LiveData`](https://developer.android.google.cn/reference/android/arch/lifecycle/LiveData.html)保持 [`WorkInfo`](https://developer.android.google.cn/reference/androidx/work/WorkInfo.html)对象; 通过观察[`LiveData`](https://developer.android.google.cn/reference/android/arch/lifecycle/LiveData.html)，您可以确定任务的当前状态，并在任务完成后获取任何返回的值。

  ## 典型工作流程

  假设您正在编写照片库应用程序，该应用程序需要定期压缩其存储的图像。您希望使用WorkManager API来计划图像压缩。在这种情况下，您不必特别注意压缩发生的时间; 你想设置任务而忘记它。

  首先，您将定义您的[`Worker`](https://developer.android.google.cn/reference/androidx/work/Worker.html)类，并覆盖其 [`doWork()`](https://developer.android.google.cn/reference/androidx/work/Worker.html#doWork())方法。您的worker类指定了如何执行操作，但没有任何关于何时应该运行任务的信息。



  ```java
  public class CompressWorker extends Worker {
  
      public CompressWorker(
          @NonNull Context context,
          @NonNull WorkerParameters params) {
          super(context, params);
      }
  
      @Override
      public Result doWork() {
  
          // Do the work here--in this case, compress the stored images.
          // In this example no parameters are passed; the task is
          // assumed to be "compress the whole library."
          myCompress();
  
          // Indicate success or failure with your return value:
          return Result.success();
  
          // (Returning Result.retry() tells WorkManager to try this task again
          // later; Result.failure() says not to try again.)
      }
  }
  ```



  ```java
  OneTimeWorkRequest compressionWork =
          new OneTimeWorkRequest.Builder(CompressWorker.class)
      .build();
  WorkManager.getInstance().enqueue(compressionWork);
  ```

  [`WorkManager`](https://developer.android.google.cn/reference/androidx/work/WorkManager.html)选择适当的时间来运行任务，平衡诸如系统负载，设备是否插入等考虑因素。在大多数情况下，如果您未指定任何约束，请立即 [`WorkManager`](https://developer.android.google.cn/reference/androidx/work/WorkManager.html)运行您的任务。如果需要检查任务状态，可以[`WorkInfo`](https://developer.android.google.cn/reference/androidx/work/WorkInfo.html)通过获取相应的句柄来获取对象。例如，如果要检查任务是否已完成，可以使用以下代码：`LiveData<WorkInfo>`

  ## 任务约束

  如果您愿意，可以指定任务运行时间的约束。例如，您可能希望指定该任务仅在设备空闲并连接到电源时运行。在这种情况下，您需要创建一个 [`OneTimeWorkRequest.Builder`](https://developer.android.google.cn/reference/androidx/work/OneTimeWorkRequest.Builder.html)对象，并使用该构建器来创建实际的[`OneTimeWorkRequest`](https://developer.android.google.cn/reference/androidx/work/OneTimeWorkRequest.html)：

  ```java
  // Create a Constraints object that defines when the task should run
  Constraints myConstraints = new Constraints.Builder()
      .setRequiresDeviceIdle(true)
      .setRequiresCharging(true)
      // Many other constraints are available, see the
      // Constraints.Builder reference
       .build();
  
  // ...then create a OneTimeWorkRequest that uses those constraints
  OneTimeWorkRequest compressionWork =
                  new OneTimeWorkRequest.Builder(CompressWorker.class)
       .setConstraints(myConstraints)
       .build();
  ```

  然后像以前一样将新[`OneTimeWorkRequest`](https://developer.android.google.cn/reference/androidx/work/OneTimeWorkRequest.html)对象传递给 在找到运行任务的时间时考虑您的约束。[`WorkManager.enqueue()`](https://developer.android.google.cn/reference/androidx/work/WorkManager#enqueue(java.util.List%3C?%20extends%20androidx.work.WorkRequest%3E))[`WorkManager`](https://developer.android.google.cn/reference/androidx/work/WorkManager.html)

  ## 取消任务

  您可以在排队后取消任务。要取消该任务，您需要其工作ID，您可以从该[`WorkRequest`](https://developer.android.google.cn/reference/androidx/work/WorkRequest.html)对象获取该工作ID 。例如，以下代码取消`compressionWork`了上一节中的请求：

  ```java
  UUID compressionWorkId = compressionWork.getId();
  WorkManager.getInstance().cancelWorkById(compressionWorkId);
  ```

  [`WorkManager`](https://developer.android.google.cn/reference/androidx/work/WorkManager.html)尽最大努力取消任务，但这本质上是不确定的 - 当您尝试取消任务时，任务可能已经运行或已完成。[`WorkManager`](https://developer.android.google.cn/reference/androidx/work/WorkManager.html)还提供了在尽力而为的基础上取消 [唯一工作序列中的](https://developer.android.google.cn/topic/libraries/architecture/workmanager/advanced.html#unique)所有任务或具有指定[标记的](https://developer.android.google.cn/topic/libraries/architecture/workmanager/basics#tag)所有任务的方法。

  ### 







