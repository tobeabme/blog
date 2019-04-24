

FCM 云消息传递（Firebase Cloud Messaging）是一种免费的跨平台消息传递解决方案，可让您向客户端应用程序发送推送消息。使用FCM，您可以将推送通知发送到单个设备，设备组或订阅特定“主题”的设备。


## FCM特征

### 消息类型
使用 FCM，您可以向客户端发送两种类型的消息：
* 通知消息，有时被视为“显示消息”。此类消息由 FCM SDK 自动处理。
* 数据消息，由客户端应用处理。

应用在后台运行时，通知消息将被传递至通知面板。应用在前台运行时，消息由回调函数处理。

接收同时包含通知和数据有效负载的消息时的应用行为取决于应用是在后台还是前台运行 - 特别是在接收时应用是否处于活动状态。
* 在后台运行时，应用会在通知面板中接收通知有效负载，且仅在用户点按通知时处理数据有效负载。
* 在前台运行时，您的应用将会接收一个消息对象，且两种有效负载都可用。

以下是包含 notification 键和 data 键的 JSON 格式的消息：
```
{
  "message":{
    "token":"bk3RNwTe3H0:CI2k_HHwgIpoDKCIZvvDMExUdFQ3P1...",
    "notification":{
      "title":"Portugal vs. Denmark",
      "body":"great match!"
    },
    "data" : {
      "Nick" : "Mario",
      "Room" : "PortugalVSDenmark"
    }
  }
}
```

### 限制和扩缩


## FCM 服务端环境

### 要求


### FCM 服务器协议

### FCM 工作流

## 设计与实现

### 问题分析


### 服务端开发实例



## 参考  

[Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging)  
[Device Group Management With Firebase Cloud Messaging](https://www.sentinelstand.com/article/device-group-management-with-firebase-cloud-messaging)  
[Firebase 云消息传送](https://docs.microsoft.com/zh-cn/xamarin/android/data-cloud/google-messaging/firebase-cloud-messaging)  

