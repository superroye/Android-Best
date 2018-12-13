# 处理Android生命周期最佳实践



- ## 依附生命周期

  一般做法是，在Activity/Fragment 的对应生命周期函数调用相关接口：

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



  事实上，[Android-arch](https://developer.android.google.cn/topic/libraries/architecture/)架构，已经包含了我们的所需，可以在任何地方实现LifecycleObserver，就可以监控Acvitity生命周期，使用这种方式，代码变得更优雅，扩展也更加轻松。

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

- ## 避免非激活时调用

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
  
  or new Handler(Looper.mainLooper()).post();
  ```

  runUiThread也是用Handler实现，只是在activity类里面，更方便调用罢了。

  我们也可以利用handler封装一个通用的runOnUi，并且利用代理模式做到一样的体验，还能自动判断生命周期。

  ```java
  class Task {
      static runOnUi(Activity activity, Runnable call){
          new Handler(Looper.mainLooper()).post(new Runnable(){
              if(!activity.isFinishing() || activity.isDestroyed()){
  				retrun；
      		}
              call.run();
          });
      }
  }
  ```

  似乎没毛病，也很简洁！

  然而有点尴尬，假设有个特殊场景，需要在onstop的时候中断执行，或者onpause的时候中断。

  还不够强大！下面尝试用uber开源的autodispose来解决。

  ```java
  
  ```


  ###     2、执行异步操作，并通知UI线程



- 胜多负少的









