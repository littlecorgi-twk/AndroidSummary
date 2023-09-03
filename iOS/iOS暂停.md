# iOS暂停

## 事件逻辑

所有的事件定义在 `QADEventDelegate.h`



## 暂停贴请求逻辑

`QADPauseModel` 是暂停贴请求类，他继承自 `QADRequestModel`。

`QADPauseModel` 有很多方法：

-   `sendRequest` 发送请求
-   `clearDelegatesAndCancel` 取消请求
-   `isLoading` 是否是加载中



### 取消请求

暂停广告中，调用`clearDelegatesAndCancel` 方法的地方只有 `QADPauseViewController.cancelPauseModel()` ，这个方法内部会先判断 `QADPauseModel.isLoading` 来看请求是否还在，如果在的话执行取消。



`cancelPauseModel` 调用的地方：

-   `QADPauseViewContrller.dealloc` 当这个对象被 OC 回收时，会调用这个方法
-   `didReceiveAdCancelAd` 收到通知需要取消广告
-   `didReceiveAdPlayerStatusChange` 收到播放器状态发生变化，变成续播时关闭广告
-   `didReceiveAdScreenModeWillChange` 屏幕方向发生变化
-   `didReceiveAdClickPauseCloseButton` 点击暂停广告关闭按钮



