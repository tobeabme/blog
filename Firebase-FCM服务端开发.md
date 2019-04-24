

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

### 何时使用针对具体平台的字段

如果需要执行下列操作，请使用针对具体平台的字段：
* 仅向特定平台发送字段
* 发送针对具体平台的字段以及通用字段

每当您想仅向特定平台发送值时，请不要使用通用字段，而是使用针对具体平台的字段。例如，要仅向 iOS 和网页发送通知而不向 Android 发送通知，您必须针对 iOS 和网页各使用一组字段。

示例：包含针对具体平台的递送选项的通知消息

以下 v1 发送请求会向所有平台发送通用的通知标题和内容，但也会发送一些针对具体平台的覆盖内容。具体而言，该请求会：
* 为 Android 和网页平台设置较长的存留时间，同时将 APNs (iOS) 消息设置为低优先级
* 设置适当的键来定义用户在 Android 和 iOS 上点按相应通知的结果（分别是 click_action 和 category）。
```
{
  "message":{
     "token":"bk3RNwTe3H0:CI2k_HHwgIpoDKCIZvvDMExUdFQ3P1...",
     "notification":{
       "title":"Match update",
       "body":"Arsenal goal in added time, score is now 3-0"
     },
     "android":{
       "ttl":"86400s",
       "notification"{
         "click_action":"OPEN_ACTIVITY_1"
       }
     },
     "apns": {
       "headers": {
         "apns-priority": "5",
       },
       "payload": {
         "aps": {
           "category": "NEW_MESSAGE_CATEGORY"
         }
       }
     },
     "webpush":{
       "headers":{
         "TTL":"86400"
       }
     }
   }
 }
```

### 限制和扩缩
我们的目标是始终提供通过 FCM 发送的每条消息。但是，传递每条消息有时会导致整体用户体验不佳。在其他情况下，我们需要提供边界以确保 FCM 为所有发送者提供可扩缩的服务。

**向单一设备发送消息的最大速率** 
您可以向单一设备发送最多 240 条消息/分钟和 5000 条消息/小时。这个高阈值意味着允许短期的流量突发，例如当用户通过聊天快速互动时。此限制可防止发送逻辑时发生的错误无意中耗尽设备上的电池电量。

警告：不要定期发送接近此最大速率的消息。这可能会浪费最终用户的资源，并且您的应用可能会被标记为滥用。

**上行消息限额** 
我们将每个项目的上行消息限制为 15000 条/分钟，以避免上行目标服务器过载。

我们将每台设备的上行消息限制为 1000 条/分钟，以防止因不良应用行为导致电池电量耗尽。

**主题消息限额** 
主题订阅添加/移除率限制为每个项目 3000 QPS。

我们将每个项目正在进行的主题扇出数量限制为 1000。之后，我们可能会拒绝其他扇出请求，直到某些扇出完成为止。

实际可实现的主题扇出率受同时请求扇出的项目数量的影响。单个项目的扇出率为 10000 QPS 并不罕见，但这个数字不是一项保证，而是系统总负载的结果。值得注意的是，可用的扇出容量在项目之间划分，而不是在扇出请求之间划分。因此，如果您的项目有两个正在进行的扇出，那么每个扇出只能确保可用扇出率的一半。最大化扇出速度的推荐方法是一次只有一个进行中的活动扇出。

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

