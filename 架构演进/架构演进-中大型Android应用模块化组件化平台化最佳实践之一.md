​	在整个Android开发领域，大大小小的互联网公司，各大技术网站，不少关于Android应用架构的技术文章，其中有代表性的，当数美团外卖Android架构和微信的架构。

​	也许会问，既然业界有成熟的软件架构，为何还要弄一套自己的架构？

## 常用的架构

- app--base--组件库/ui库
- 模块化
- 组件化
- 插件化

## 业界的现状

在开发领域，公司大小项目都有，那么架构的选择上，会有很多考虑。

一次性应用，基本无新功能迭代，可能简单的分个app和base；

中型应用，选择有一定的模块化，组件化的架构设计；

大型应用，完全的组件化或插件化设计；



最常见的状况，应用先选择一个简单架构，随着应用增加功能迭代，再不断的升级架构方案。

通常企业的架构方案大相径庭，而细节上又很大差异，因为架构在不断的演进过程中，所以适合企业的架构才是好架构。

## 美团和微信的现状

美团外卖跟美团应用，有业务的重合性，并且业务庞大，功能点很多。业务型的应用注重强运营，不断推出红包活动，优惠券活动功能，所以有强烈业务复用需求和频繁的功能迭代需求。他们组件化方案中，增加了平台层管理业务的差异化，以适应多应用的多样性需求。美团架构演进：https://tech.meituan.com/2018/03/16/meituan-food-delivery-android-architecture-evolution.html

微信又有点不同，微信业务模块很多，但是微信是个10亿级超级应用平台，他们对稳定性和热修复更感兴趣。

微信架构演进：https://blog.csdn.net/seu_calvin/article/details/77149750

但是不管怎么说，即使大型企业也不是一步到位，都是逐渐演进的过程。

## 我们的架构选择

我们选择架构的标准，不是照搬业界的标准，也不是照搬美团和微信。应该是设计适合自己公司业务发展需要，适合自己的技术栈的架构。而且不论大小应用，都用一套架构，不做特殊衡量，做到标准化生产，模式化开发。

下面介绍最近定义的一套架构，https://github.com/superroye/zzc_mobile_arch

![架构依赖关系图](https://raw.githubusercontent.com/superroye/zzc_mobile_arch/master/doc/%E6%9E%B6%E6%9E%84%E5%9B%BE20190201.png)

图片画的很渣，不影响优秀的你对它的理解。

- ### app

  划分了多模块，个人中心/首页/客服等等，模块之间不能直接调用，代码隔离，模块间需要用ARoute通信。

- ### base

  base是app的一些模块通用的封装，bean，Application初始化，组件（统计，友盟，热云监测，网络库）初始化，或者其他需要下沉到base供其他模块调用的类。

- ### platformSDK

  platformSDK负责集成一些通用的push，log，动态权限，或其他功能组件的集成（包括对原始组件或依赖的二次封装，更快捷的调用），通常以aar的形式供所有的应用依赖调用。

- ### platformCore

  platformCore，把platformSDK的基础功能下沉到platformCore，譬如Application对象的维护，工具类。所有的组件或其他module都依赖platformCore。

- ### 组件库

  #### 网络库

  网络库是Rxjava2+retrofit2+okhttp3的二次封装。包含功能有：

  - 快速的网络库初始化和调用。
  - 多种缓存策略。只读缓存，只读网络，优先缓存，优先缓存并网络，只读缓存1小时。
  - 统一公参。支持统一设置header，get参数，post参数。
  - 可定义的通用响应数据类型。json自动转成bean。
  - 可定义的ResponseObserver。支持请求时loading效果，支持自定义响应数据的业务错误码。

  ```java
  //初始化
  ApiManager.builder(CoolAPi.class)
                  .baseUrl("https://suggest.taobao.com/")
                  .requestAdapter(new CommonParamsAdapter() {
                      @Override
                      public void addHeader(Request.Builder builder) {
  
                      }
  
                      @Override
                      public void addQueryParams(Request originalRequest,      HttpUrl.Builder httpUrlBuilder) {
  
                      }
  
                      @Override
                      public void addPostParams(Request originalRequest, Request.Builder requestBuilder) {
  
                      }
                  }).build();
  
  //调用请求
  ResponseDataObserver observer = new ResponseDataObserver<TaobaoTest>(this){
  
              @Override
              public void onResponse(TaobaoTest result) {
                  T.Log.d("====="+result.toString());
                  list().postValue(result);
              }
          };
          observer.setLoading(getUIBehavior());
  
         ApiManager.api(CoolAPi.class).testSearchNetwork("市").subscribe(observer);
  ```



  - uilibs

    包含：
    基础Activity/Fragment
    dialog,BottomTab, statusbar
    recyclerview快速开发

    viewtools



    原BaseActivity是直接在里头实现沉浸式、loading、toolbar功能，通过继承来实现多样性，
    现把BaseActivity的沉浸式、loading、toolbar等通用UI行为，抽象成行为，并支持外部扩展。用组合代替继承，并遵循开闭原则。（详细见[github](https://github.com/superroye/zzc_mobile_arch)）

  - designStyle

    包含一组尺寸，颜色规范，可推动设计师们按规范设计。
    主要目的是撸UI尽量不用思考，提高开发效率和统一的尺寸和颜色体验。

    通过重新修改尺寸，颜色值，主题样式即可快速使用。

    **内容包含：**
    文字size，大中小几种规范；
    组件外距/内距(margin/padding) 统一只有几种规格；
    分界线长度，宽度，颜色；
    文字颜色，主颜色，次要颜色，最次要颜色；
    activity主题，状态栏，标题栏，背景颜色，光标颜色，问题颜色等等；
    dialog主题，圆角，边距，底色；

  - autodispose

    RxJava+autoDisposable实现的针对处理Android生命周期问题的一个库，很实用。使用场景见（[Android生命周期处理，多场景下的优雅实践](https://www.jianshu.com/p/147299a9a16d)）



  未完成：

  模块通信的ARoute的接入

  Application的组件化并行初始化

  platformSDK对组件的快速接入封装

  UI组件的完善

  UI不同Android SDK版本的兼容性处理






