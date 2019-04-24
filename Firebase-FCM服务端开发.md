

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
![01-server-fcm-app-sml.png](https://github.com/tobeabme/blog/blob/master/images/01-server-fcm-app-sml.png)

#### Registration with FCM
客户端应用必须首先向FCM注册，然后才能进行消息传递。客户端应用程序必须完成下图中显示的注册步骤：
![02-app-registration-sml.png](https://github.com/tobeabme/blog/blob/master/images/02-app-registration-sml.png)

- 客户端应用程序联系FCM以获取注册令牌，将发件人ID，API密钥和App ID传递给FCM。 
- FCM将注册令牌返回给客户端应用程序。 
- 客户端应用程序（可选）将注册令牌转发到应用服务器。

应用程序服务器缓存注册令牌，以便与客户端应用程序进行后续通信。应用服务器可以向客户端应用程序发送确认，以指示已收到注册令牌。在进行此握手之后，客户端应用程序可以从应用服务器接收消息（或向其发送消息）。如果旧令牌被泄露，则客户端应用程序可能会收到新的注册令牌（有关应用程序如何接收注册令牌更新的示例，请参阅[使用FCM进行远程通知](https://docs.microsoft.com/zh-cn/xamarin/android/data-cloud/google-messaging/remote-notifications-with-fcm)）。

#### Downstream messaging
下图说明了Firebase Cloud Messaging如何存储和转发下游消息：
![03-downstream-sml.png](https://github.com/tobeabme/blog/blob/master/images/03-downstream-sml.png)

当应用服务器向客户端应用程序发送下游消息时，它将使用上图中枚举的以下步骤：
- 应用服务器将消息发送到FCM。
- 如果客户端设备不可用，则FCM服务器将消息存储在队列中以供稍后传输。消息最多保留在FCM存储中4周（有关详细信息，请参阅[设置消息的生命周期](https://firebase.google.com/docs/cloud-messaging/concept-options#ttl)）。
- 当客户端设备可用时，FCM会将消息转发到该设备上的客户端应用程序。
- 客户端应用程序从FCM接收消息，对其进行处理，并将其显示给用户。例如，如果消息是远程通知，则将其呈现给通知区域中的用户。
- 在此消息传递方案中（应用服务器将消息发送到单个客户端应用程序），消息的长度最多可达4kB。

#### Topic messaging
主题消息使应用服务器可以向已选择加入特定主题的多个设备发送消息。您还可以通过Firebase控制台通知GUI撰写和发送主题消息。 FCM处理主题消息到订阅客户端的路由和传递。此功能可用于天气警报，股票报价和标题新闻等消息。
![04-topic-messaging-sml.png](https://github.com/tobeabme/blog/blob/master/images/04-topic-messaging-sml.png)

主题消息中使用以下步骤（在客户端应用程序获取注册令牌之后，如前所述）
- 客户端应用程序通过向FCM发送订阅消息来订阅主题。
- 应用服务器将主题消息发送到FCM以进行分发。
- FCM将主题消息转发给已订阅该主题的客户端。

## 设计与实现

### 问题分析

问题

同一个设备，同一个应用，多个账号的情况
假设有A，B，C 3个账号，如果A账号登录后发表了一篇文章，然后A从APP内退出了。在发表文章之后—退出之前，如果服务端给A账号发送了通知，那么A应该能近实时的收到Firebase的设备通知。如果A退出了，服务端无法且也不会往Firebase发送通知给A设备Token，因为服务端会在A退出时删除对应的Token。

如果A从APP内退出后，又登录A或B、C账号，那再次登录后的注册Token和之前的Token是不一致的。所以如果发往上一个Token的通知，期望在这一次的重新登录后接收到是不可能的，因为每次登录后Token都不一样，只能接受指定Token的通知。

如果不是从APP内退出，如是APP的系统后台进程退出了，如果这个期间有设备通知，肯定是收不到的。在APP重新打开之后Token应该是不变的，但是也没发收到错过的设备通知，此时只能收到最后一条消息通知。


同一个账号，多个设备的情况
期望是每个设备都收到同样的通知。当然如果有一个设备上的账号退出了，或者APP本身彻底退出了，那么该设备无法收到通知，但是不影响该账号在其它设备上接收消息通知。


流程

1. 账号在登录时向服务端注册Token（成功登录后，获取注册Token然后请求服务端Token注册接口）。
2. 服务端保存用户ID、设备、Token（用户ID从 http header 的 LoginToken 里认证并解析出来，设备信息也从http header中获取）。「如果采用Topic方式这一步不是必须的」
3. 服务端创建Topic：以用户ID作为Topic。
4. 服务端订阅Topic：以uid对应的Tokens作为订阅者，订阅Topic（注意一个uid可能有多个注册token噢）。
5. 服务端发送消息到Topic：Topic名称实际为uid。
6. Firebase 会把通知消息Push到Topic的订阅者。
7. 客户端收到通知或消息：如果同一个账号，多个设备的情况，那么每个设备都会收到通知。

1. 账号在退出时向服务端注销Token（退出前，获取注册Token然后向服务端发送注销Token请求）。
2. 服务端删除已经注册的Token（某账号下的）。
3. 服务端取消订阅Topic。


Firebase设备通知推送原则

当应用在前台或系统后台运行时，会接收到Firebase推送给该应用的所有通知。否则当应用重新启动时，会且只能收到Firebase推送的最后一条通知。为什么呢？在我看来这是合理的，设备通知不同于站内消息，不可能把应用彻底离线的这段时间所产生的设备通知，在应用重新打开时一次全部推送给设备。另外即使没有设备通知还有站内消息呢，错过的通知在站内消息里都能找到。

设备 - Token - 用户账号 三者关系

账号 在 多个设备 登录，账号：设备=1:N
一个设备一个Token，设备：Token=1:1

那是否同一个token可以在多个设备上接受消息吗？

用户登录.生成Token.注册APPServer

安全保障：
Token注册接口需要校验用户是否登录及登录Token是否正常。

### 数据结构

```
notify_firebase

device
token
uid
CreateAt
```

### Go实例
```
package main

import (
    "context"
    "fmt"
    "log"

    firebase "firebase.google.com/go"
    "firebase.google.com/go/messaging"
    "google.golang.org/api/option"
)

func main() {
    var serviceAccountKey = []byte(`{
        "type": "service_account",
        "project_id": "lvxiaorun-22c97",
        "private_key_id": "a212612afee74a54e9e30cb2bfbc5b0c118ca3cc",
        "private_key": "-----BEGIN PRIVATE KEY-----\nMIIEvwIBADANBgkqhkiG9w0BAQEFAASCBKkwggSlAgEAAoIBAQCpieVk85eBv9Eh\nGktoEf7rv+lz7PQh6CTa4F2QtljaFM3k3hv4v3F+JYnVhWKYL1yz7D4vNgByv3lD\nsXGRMwe1uK5pP5DLke7X1q0RQJn928QrUKDK71MKqDPTeYMjZEoVLdFRbVzbvpf9\nwks6H+vh9CFGwJ4D/H/7tX//bRR/f6ape5J+VwEjWH0itLXd35klh3oqWCA4/Dwg\nxNg7LzejVdM+GOhyuvLCZfEOcno8wF+YI8Uj7Ut27Wj9fLcO2cW2ybaK8Y87CfKO\nKmQgSt1lqgbrksi+fe725BxbXZp/eF9MrS8CPLOWgv/DRW3aP0ocgxx6TX0z8K1a\ncFdvXwcxAgMBAAECggEABgfe9liY6NdlLceE66qKNiIhQIurAoLCvttw0Js/7WAE\nk/HXrmFG/P0CWmtQfsfehRLwAldqLCrF+kO3XboiOdNcNue5M5iZFanwBZdV8vsM\njxri4V0ih9REZa8inFFutjKnSb15amKs/uyYpvRoPGUmAuGKrWsftVk3OKONcVyP\nA2L8N/keTc8IWKe4GlIdfoU2f/hAxbVYiu0u2HXgdGlFmY1vpOrFgHbJZcXyUlhg\nQvtgCnt/bwKr3GkuWXJff9pm4wf7A2e73jv68zuucCV5P9iExVIAZIkLvhlxgqn1\ncKF7Ib5K/RVzdXgcVNpL0V6ewv+IC1fjHKLJYbXiDQKBgQDSbOXnxzb639NOIlvJ\nSb0wpTJVmWLWsIvJjO74K2ZNOQ38f7LDuvZ1/M2W0QsvVOw43Gpqijza0HV0aCRH\nqrPldgTRBJAqY/VexWt/kuOvrCA1IkXRl8jayLoMrGxCv12fiIp8ryDbikCuVDCE\n9VuenZGbe1++O5aBjMG/3c6sxQKBgQDOQgVAiAiTmaiUv9NPIX5sfpuV0U7QB6TT\n83wS45ahheanWf/G2CvidazK9KR5oEquljVGTPsmK01x17/1pnM+o6p84ZETNQ12\n4SsJpT7NxId/Vkw2Hmn/qzhHZa1wckdEPrHc70Upre/YJ6snp+brCBQNiuhALZde\nc6yrHZKvfQKBgQCNvlE3ydfdMjxyW26crpFEXWMEiigsGgxvngGzJfjpd89WEObo\nNd6jJ8GNIA96uKfOvZrpXWkUtGsKGMSnifNYVCF2cq5x/5dfWXjKHLZGtZmUcRu6\nzZW82o2Iz/S1GZcFScKPrqBhgkWDqK5uQaCPvfBBXd/mktkVNy2kAtOfSQKBgQC+\nx6xp+ynLtOaM6D4BRJ7Wpektk5QNsfRRJDdQlXi/4MXvZ7zBZTR6XJQ+ijkUUyKh\nCEkwxIXN0WHp+kERbCvO9b39kvsIxBq3KiEP4+wKkk0uiFkn+cvb87izuaXKi7nF\nsyP7ksnrenqN+mtC2/goz6kUubaHnmQTtnUxNcJ3VQKBgQDDbVOQx1LCex9HHAV4\n2Rula9E27w0ujYdhf3YvQ66mOG6Ig5+1YhiXrKrElfCCAVc7H8KmAt7iPHcJXKgC\nMJSJK7XWasj+Ld2SdbPT2LgwGglx907N+wyS47+vOh9Ppu3d5Tv9/BIKkvNKXBu+\ndQBw8Xw1P+U5QtIPf7LOKMtNEw==\n-----END PRIVATE KEY-----\n",
        "client_email": "firebase-adminsdk-k3958@lvxiaorun-22c97.iam.gserviceaccount.com",
        "client_id": "103134511880014813376",
        "auth_uri": "https://accounts.google.com/o/oauth2/auth",
        "token_uri": "https://oauth2.googleapis.com/token",
        "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
        "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/firebase-adminsdk-k3958%40lvxiaorun-22c97.iam.gserviceaccount.com"
     }`)

    // opt := option.WithCredentialsFile("path/to/serviceAccountKey.json")
    opt := option.WithCredentialsJSON(serviceAccountKey)
    app, err := firebase.NewApp(context.Background(), nil, opt)
    if err != nil {
        log.Fatalf("error initializing app: %v\n", err)
    }

    // Obtain a messaging.Client from the App.
    ctx := context.Background()
    client, err := app.Messaging(ctx)
    if err != nil {
        log.Fatalf("error getting Messaging client: %v\n", err)
    }

    uid := "25696773511053390"
    // This registration token comes from the client FCM SDKs.
    registrationToken := "fxV6ToLJh3A:APA91bGfcaOl4mmnj_mPY7MTscjzT0aZLvyK5xaLLboWavxFoeqc3hZu_npEtaINebzHAfOrARg4kn9RmWC9ZYKvhqJrPhnNI43qtUruQsvd7Or7w_ZnDG4agOMM_7xB0J4ci9UHPT5S"

    // These registration tokens come from the client FCM SDKs.
    registrationTokens := []string{
        registrationToken,
        // ...
        "cBYSJNhfG_Q:APA91bFzLxiSVynUc2thc6aGfF1ba_6WoJvOctw2_1cIlUEr2r7Pf-n_Qk6uisLpc9Whcf-UU4WwcjnRwLTm_Zok1pH2RGw2_WvLmaT_AdZp84caH29haB4gQFIdrc0wQSr-vVgR0F3o",
    }

    subscribe(ctx, client, uid, registrationTokens)
    // sendMsgToToken(ctx, client, registrationToken)
    sendMsgToTopic(ctx, client, uid)
    // createCustomToken(ctx, app)
}

func sendMsgToTopic(ctx context.Context, client *messaging.Client, topic string) {
    notification := &messaging.Notification{
        Title: "0007 2019-04-23",
        Body: "fffffffffffffffffffffffffff.",
    }

    // See documentation on defining a message payload.
    message := &messaging.Message{
        Data: map[string]string{
            "score": "88888",
            "time": "2:45",
        },
        Notification: notification,
        Topic: topic,
    }

    // Send a message to the devices subscribed to the provided topic.
    response, err := client.Send(ctx, message)
    if err != nil {
        log.Fatalln(err)
    }
    // Response is a message ID string.
    fmt.Println("Successfully sent message:", response)
}

func sendMsgToToken(ctx context.Context, client *messaging.Client, registrationToken string) {
    // See documentation on defining a message payload.

    notification := &messaging.Notification{
        Title: "$GOOG up 1.43% on the day",
        Body: "$GOOG gained 11.80 points to close at 835.67, up 1.43% on the day.",
    }

    // timestampMillis := int64(12345)

    message := &messaging.Message{
        // Data: map[string]string{
        //  "score": "850",
        //  "time": "2:45",
        // },
        Notification: notification,
        Webpush: &messaging.WebpushConfig{
            Notification: &messaging.WebpushNotification{
                Title: "title",
                Body: "body",
                //      Icon: "icon",
            },
            FcmOptions: &messaging.WebpushFcmOptions{
                Link: "https://fcm.googleapis.com/",
            },
        },
        Token: registrationToken,
    }

    // Send a message to the device corresponding to the provided
    // registration token.
    response, err := client.Send(ctx, message)
    if err != nil {
        log.Fatalln(err)
    }
    // Response is a message ID string.
    fmt.Println("Successfully sent message:", response)
}

func subscribe(ctx context.Context, client *messaging.Client, topic string, registrationTokens []string) {
    // Subscribe the devices corresponding to the registration tokens to the
    // topic.
    response, err := client.SubscribeToTopic(ctx, registrationTokens, topic)
    if err != nil {
        log.Fatalln(err)
    }
    // See the TopicManagementResponse reference documentation
    // for the contents of response.
    fmt.Println(response.SuccessCount, "tokens were subscribed successfully")
}

func createCustomToken(ctx context.Context, app *firebase.App) {
    authClient, err := app.Auth(context.Background())
    if err != nil {
        log.Fatalf("error getting Auth client: %v\n", err)
    }

    token, err := authClient.CustomToken(ctx, "25696773511053390")
    if err != nil {
        log.Fatalf("error minting custom token: %v\n", err)
    }

    log.Printf("Got custom token: %v\n", token)
}
```


## 参考  

[Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging)  
[Device Group Management With Firebase Cloud Messaging](https://www.sentinelstand.com/article/device-group-management-with-firebase-cloud-messaging)  
[Firebase 云消息传送](https://docs.microsoft.com/zh-cn/xamarin/android/data-cloud/google-messaging/firebase-cloud-messaging)  

