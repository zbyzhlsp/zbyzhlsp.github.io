---
title: 反作弊SDK说明
---
# 反作弊SDK说明


### 反作弊的应用场景和意义
　　我们在业务上经常会有一些促销活动，如“五折大促”等，这些活动通常只允许某个用户或某个设备参加一次。但是经常有非法用户，使用模拟器或者少数设备，通过篡改imei或者其他参数的方式欺骗后台来进行刷单获利，这种行为会对公司利益造成巨大损失。该反作弊服务是在这样的场景下，基于Qunar自己的反作弊实践而产生的。

### 反作弊的原理
在业务的关键流程中，客户端收集详细的用户手机设备软硬件信息，并上传到服务端；服务端根据反作弊策略进行风险评定。

### 工作流程说明(虚线为初始化流程)

![工作流程|center|450*300](http://7xt995.com2.z0.glb.clouddn.com/%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B.png)
<br>
> ① 请求“反作弊策略”，获取gid<br>
> ② 获取gid，保存策略<br>
> ③ 根据“策略”收集“设备特征信息”，并发送到反作弊服务（初始化完成）<br>
> ④ 在需要“埋点”的关键业务处（比如下单接口之前），收集“设备特征信息”，并连同“业务参数”（比如用户id等）一起发送到反作弊服务
> ⑤ 反作弊服务根据“初始化”和“埋点”时收到的“设备特征信息”，判断该设备的风险等级，并根据业务参数，将判定结果推送给业务后台，业务后台做响应的处理（异步埋点方式）。或者反作弊服务立刻将判定结果返回给app(同步埋点方式)，app做出响应的业务处理。

### 客户端使用文档
#### 一、申请appkey
- 调用者提供**包名**和**证书SHA1**，由Qunar生成唯一的appkey

#### 二、配置
1. 将appkey以meta-data的方式配置到清单文件中，name固定为**QAntiCheat.Key**：
```xml
<meta-data
   android:name="QAntiCheat.Key"
   android:value="9biuyqZ6OLiyZvBTTmB42d4m" />
```
2. 将**aar文件**或者**Jar+so**引入项目中
#### 三、接口说明
1. 初始化接口：
- 在app的Application的onCreate中调用init()方法的两种重载之一：

```
//初始化反作弊sdk，不设置默认回调接口地址，在埋点时设置。
QAntiCheat.init(Context context);
//初始化反作弊sdk,设置默认的回调接口地址，埋点时不再统一设置
QAntiCheat.init(Context context,String defaultCallback);
```

2. 埋点
所谓**埋点**，即在关键业务（如下单）处，发送设备特征信息到服务端，用于判断是否存在作弊风险
- **异步埋点**：
	服务器不会立刻返回校验结果，而是将校验结果推送到设置的callback地址上。

	```
	  QAntiCheat.build(context)
		.setExtParam("{orderId:32166788,uid:738205}")//设置业务参数，json
		.setAsyncCallback("http://www.qunar.com/QAnti/callback")//设置异步校验的回调接口地址
		.setLocation(423.12323,765.32131,BAIDU_MAP)//设置定位结果
		.setBType(100)//设置业务类型，与后端商定
		.doAsyncCheck();//调用异步埋点函数
	```

- **同步埋点**
服务器会立刻返回作弊校验的结果，并通过接口返回校验结果，开发者根据回调结果进行后续业务的处理。用户需要自己实现一下`QAntiHandler.SyncCheckListener`接口，并作为参数传递给`doSyncCheck()`方法

```
  QAntiCheat.build(context)
	.setExtParam("{orderId:32166788,uid:738205}")//设置业务参数，json
	.setLocation(423.12323,765.32131,BAIDU_MAP)//设置定位结果
	.setBType(100)//设置业务类型，与后端商定
	.doSyncCheck(new QAntiHandler.SyncCheckListener() {
                          @Override
                          public void onCheckBegin() {
                          }
                          @Override
                          public void onCheckSuccess() {
                          }
                          @Override
                          public void onCheckError() {
                          }
                          @Override
                          public void onCheckEnd() {
                          }
                      });
```
3. 释放与取消任务

```
QAntiCheat.cancelAllTasks();
```


取消网络任务池中未执行或者尚未返回的任务，并释放指向同步埋点回调接口的变量，防止内存泄露。**如果使用了同步埋点的方式，务必在必要的生命周期（如onPause()、onStop()或onDestroy()）中调用该方法**
