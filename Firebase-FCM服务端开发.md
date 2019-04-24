

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
应用服务器或受信任的服务器环境向 FCM 后端发送消息请求，然后 FCM 后端再将消息发送到用户设备上运行的客户端应用。

受信任的服务器环境的要求
您的应用服务器环境必须符合以下条件：
* 能够向 FCM 后端发送格式正确的消息请求。
* 能够使用指数退避算法处理请求和重新发送请求。
* 能够安全地存储服务器授权凭据和客户端注册令牌。
* 对于 XMPP 协议（如果使用），服务器必须能够生成消息 ID 来唯一标识它发送的每条消息（FCM HTTP 后端会生成消息 ID 并在响应时返回这些 ID）。XMPP 消息 ID 对于每个发送者 ID 而言都应是唯一的。

建议使用 Firebase Admin SDK，因为它支持各种流行的编程语言，而且处理身份验证和授权的方法非常便捷。
Admin FCM API 可处理后端身份验证工作，同时便于发送消息和管理主题订阅。使用 Firebase Admin SDK，您可以执行以下操作：
* 向个别设备发送消息
* 向主题和与一个或多个主题匹配的条件语句发送消息。
* 为设备订阅和退订主题
* 针对不同目标平台构建量身定制的消息负载。

### FCM 服务器协议
目前 FCM 提供以下原始服务器协议：
* FCM HTTP v1 API
* 旧版 HTTP 协议
* 旧版 XMPP 协议

您的应用服务器可以分开使用或同时使用这些协议。因为在向多个平台发送消息方面 FCM HTTP v1 API 是最新、最灵活的协议，因此推荐尽可能使用此协议。如果您需要从设备到服务器的上行消息传递功能，则需实现 XMPP 协议。

XMPP 消息传递与 HTTP 消息传递具有以下差异：
* 上行/下行消息
    * HTTP：仅下行，即云到设备。
    * XMPP：上行和下行（即设备到云、云到设备）。
* 消息传递（同步或异步）
    * HTTP：同步。应用服务器以 HTTP POST 请求的形式发送消息，并等待响应。此机制是同步的，且发送者无法在收到响应前发送其他消息。
    * XMPP：异步。应用服务器通过持续型 XMPP 连接，以全线速向/从所有设备发送/接收消息。XMPP 连接服务器异步发送确认或失败通知（以 ACK 和 NACK JSON 编码的特殊 XMPP 消息形式）。
* JSON
    * HTTP：JSON 消息以 HTTP POST 的形式发送。
    * XMPP：JSON 消息封装在 XMPP 消息中。
* 纯文本
    * HTTP：纯文本消息以 HTTP POST 的形式发送。
    * XMPP：不支持。
* 向多个注册令牌发送多播下行消息。
    * HTTP：支持 JSON 格式的消息。
    * XMPP：不支持。


> 以上信息均摘自官网，更多信息请见官网。

### FCM 工作流

#### Overview
如下图所示，FCM充当消息发送者和客户端之间的中介。客户端应用程序是在设备上运行的启用FCM的应用程序。应用服务器（由您或您的公司提供）是客户端应用通过FCM与之通信的启用FCM的服务器。与GCM不同，FCM使您可以通过Firebase控制台通知GUI直接向客户端应用程序发送消息
![01-server-fcm-app-sml.png](https://github.com/tobeabme/blog/blob/master/images/messagePushFlowChart.jpg)

#### Registration with FCM
客户端应用必须首先向FCM注册，然后才能进行消息传递。客户端应用程序必须完成下图中显示的注册步骤：
![02-server-fcm-app-sml.png](https://github.com/tobeabme/blog/blob/master/images/messagePushFlowChart.jpg)

- 客户端应用程序联系FCM以获取注册令牌，将发件人ID，API密钥和App ID传递给FCM。 
- FCM将注册令牌返回给客户端应用程序。 
- 客户端应用程序（可选）将注册令牌转发到应用服务器。

应用程序服务器缓存注册令牌，以便与客户端应用程序进行后续通信。应用服务器可以向客户端应用程序发送确认，以指示已收到注册令牌。在进行此握手之后，客户端应用程序可以从应用服务器接收消息（或向其发送消息）。如果旧令牌被泄露，则客户端应用程序可能会收到新的注册令牌（有关应用程序如何接收注册令牌更新的示例，请参阅[使用FCM进行远程通知](https://docs.microsoft.com/zh-cn/xamarin/android/data-cloud/google-messaging/remote-notifications-with-fcm)）。

#### Downstream messaging
下图说明了Firebase Cloud Messaging如何存储和转发下游消息：
![03-downstream-sml.png](https://github.com/tobeabme/blog/blob/master/images/messagePushFlowChart.jpg)

当应用服务器向客户端应用程序发送下游消息时，它将使用上图中枚举的以下步骤：
- 应用服务器将消息发送到FCM。
- 如果客户端设备不可用，则FCM服务器将消息存储在队列中以供稍后传输。消息最多保留在FCM存储中4周（有关详细信息，请参阅[设置消息的生命周期](https://firebase.google.com/docs/cloud-messaging/concept-options#ttl)）。
- 当客户端设备可用时，FCM会将消息转发到该设备上的客户端应用程序。
- 客户端应用程序从FCM接收消息，对其进行处理，并将其显示给用户。例如，如果消息是远程通知，则将其呈现给通知区域中的用户。
- 在此消息传递方案中（应用服务器将消息发送到单个客户端应用程序），消息的长度最多可达4kB。

#### Topic messaging
主题消息使应用服务器可以向已选择加入特定主题的多个设备发送消息。您还可以通过Firebase控制台通知GUI撰写和发送主题消息。 FCM处理主题消息到订阅客户端的路由和传递。此功能可用于天气警报，股票报价和标题新闻等消息。
![04-topic-messaging-sml.png](https://github.com/tobeabme/blog/blob/master/images/messagePushFlowChart.jpg)

主题消息中使用以下步骤（在客户端应用程序获取注册令牌之后，如前所述）
- 客户端应用程序通过向FCM发送订阅消息来订阅主题。
- 应用服务器将主题消息发送到FCM以进行分发。
- FCM将主题消息转发给已订阅该主题的客户端。

## 设计与实现

### 问题分析


### 服务端开发实例



## 参考  

[Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging)  
[Device Group Management With Firebase Cloud Messaging](https://www.sentinelstand.com/article/device-group-management-with-firebase-cloud-messaging)  
[Firebase 云消息传送](https://docs.microsoft.com/zh-cn/xamarin/android/data-cloud/google-messaging/firebase-cloud-messaging)  

