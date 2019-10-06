## AlertManager
---
### Architecture
![image](https://github.com/prometheus/alertmanager/raw/master/doc/arch.svg?sanitize=true)

---
- Alertmanager作为一个独立的组件，负责接收并处理来自Prometheus Server(也可以是其它的客户端程序)的告警信息。Alertmanager可以对这些告警信息进行进一步的处理，比如当接收到大量重复告警时能够消除重复的告警信息，同时对告警信息进行分组并且路由到正确的通知方，Prometheus内置了对邮件，Slack等多种通知方式的支持，同时还支持与Webhook的集成，以支持更多定制化的场景。同时AlertManager还提供了静默和告警抑制机制来对告警通知行为进行优化。

    - 客户端通过`POST`请求向AlertManager推送告警信息。
    - 每条告警信息中的`labels`可用于唯一识别告警信息并用于去重。

- AlertManager主要分为两个部分，路由(`router`)和接收器(`receiver`)。告警消息先被经过路由树，然后被分配到对应的接收器中。路由树是由预先设定的路由规则生成的。其高可用架构如上图所示，具体流程如下：
    - Prometheus会通过调用AlertManager提供的告警接口将原始的告警消息发送到AlertManager。
    - AlertManager的API除了接收告警，还接收静默请求，将其分别保存到各自的`provider`里。
    - `provider`提供了一个订阅（`subscribe`）接口，这样`Dispatcher`组件便可以获取告警数据，并对数据进行分组，通过用户预先设置的规则进入告警抑制阶段或静默阶段。
    - 如果通过了上面的告警静默阶段，则进入路由分发阶段，最终发送通知。

#### 上报数据格式
```
[
  {
    "labels": {
      "alertname": "<requiredAlertName>",
      "<labelname>": "<labelvalue>",
      ...
    },
    "annotations": {
      "<labelname>": "<labelvalue>",
    },
    "startsAt": "<rfc3339>",
    "endsAt": "<rfc3339>",
    "generatorURL": "<generator_url>"
  },
  ...
]
```
- 其中：
    - 告警信息的`Fingerprint`是通过`labels`来进行计算的。
    - `startAt`和`endsAt`在`provider`的告警合并会用到。

---
### Alert Provider
- Prometheus或者告警发送系统可以通过API的方式发送给Alertmanager，收到告警后将告警分别存储在`Alert Provider`中（当前实现是存储在`内存`中，可以通过接口的方式自行实现其他存储方式比如MySQL或者ES）。
- 已解决的`alerts`会定期地从`provider`中清除，用户可以定义相应的回调函数，在`alerts`被删除时进行相应的回调处理。
- `Alert Provider`收到告警信息后对相同的指标数据进行合并处理。


```
// Alerts gives access to a set of alerts. All methods are goroutine-safe.
type Alerts interface {
    // Subscribe returns an iterator over active alerts that have not been
    // resolved and successfully notified about.
    // They are not guaranteed to be in chronological order.
    Subscribe() AlertIterator
    // GetPending returns an iterator over all alerts that have
    // pending notifications.
    GetPending() AlertIterator
    // Get returns the alert for a given fingerprint.
    Get(model.Fingerprint) (*types.Alert, error)
    // Put adds the given alert to the set.
    Put(...*types.Alert) error
}
```

---
### Silence Provider

---
### Dispatcher
- `Dispatcher` sorts incoming alerts into aggregation groups and assigns the correct notifiers to each.
- AlertManager内部的`Dispatcher`通过订阅的方式获得告警信息更新（获得`Alerts的迭代器`，通过for循环不断的获得发送到信道中的`Alerts`, 通过route的match函数获得匹配的route对象（比如基于标签的正则表达，传递到不同的邮件或者slack信道路由）,并且每隔一段时间将执行一次清理操作（当`aggregate group`中的告警数量为空的时候），删除之前的记录。收到的Alert通过标签匹配的方式被送到不同的聚合组中等待`Pipeline`流程进行处理。

#### Aggregate group
- aggrGroup aggregates alert fingerprints into groups to which a common set of routing options applies.
It emits notifications in the specified intervals.
- 聚合组`aggregate group`用来管理具有相同属性信息的告警，通过将相同类型的告警进行分组可以统一的管理，因为有时候告警处理是大量同时出现的（比如一个数据中心的失效将导致成百上千的告警产生，通过分组可以聚合相同标签到一个邮件或者接收者中）。分组创建将依赖于处理`route`路由和告警的`labels`标签，不同的告警`labels`将产生不同的聚合组，所有接收到的告警将首先计算一个聚合组的`Fingerprint`如果找到则直接插入到该组，否则创建一个新的聚合组，每次新创建的聚合组都会启动一个goroutine来执行实际的`pipeline work`.

---
### Inhibitor
- An `Inhibitor` determines whether a given `label set` is muted based on the currently active alerts and a set of `inhibition rules`. It implements the `Muter` interface.
- `Inhibitor`用于管理相同的告警配置， ~~比如下面的配置定义了当告警名称alertname一致的时候，如果严重告警存在的时候，途同级别告警将被过滤掉。~~
- 查询流程上将获得的alert的label进行检查，匹配检查的内容满足target匹配但是source不匹配的标记为Inhibited.

---
### Silencer

- `Silencer`用来取消告警，比如直接配置告警在某一段时间内不触发任何消息，可以基于正则表达式的匹配，该配置可以通过alertmanager的WebUI或者API接口配置。
- 当流程传递到`Silence`步骤时候，`Silence`模块将循环检查每一个告警是否满足匹配，~~比如设置某一个告警标签出现后取消告警。当查询结束后返回一个sils（Silence的结构体，用来指定某一类告警的Silence在一段时间内的处理对象。）一个告警可能会被多个Silence同时管理。~~
- 同时要实现集群管理，彼此之间的Silence状态也要共享（告警发送给多个AM），因此系统设计的时候加入了SilenceProvider来进行集群之间的Silence管理,彼此之间通过protoBuf来进行数据状态的同步。同时集群在接收到告警后也要进行通知，告知其他的节点关于告警的处理状态，防止多个通知同时被发送。

---
### Notify Provider


---
### Router

---
### Receiver Stage

#### Wait
- 等待间隔用来设置发送告警的等待时间，对于集群操作中，需要根据不同的peer设置不同的超时时间，如果仅仅一个Server本身，等待间隔设置为0；

#### Dedup
- `Dedup Stage`用于管理告警的去重，传递的参数中包含了一个`NotificationLog`, 用来保存告警的发送记录。当有多个机器组成集群的时候，`NotificationLog`会通过协议去进行通信，传递彼此的记录信息，加入集群中的A如果发送了告警，该记录会传递给B机器，并进行`merge`操作，这样B机器在发送告警的时候如果查询已经发送，则不再进行告警发送。

#### Retry
- `Retry Stage`利用`backoff`策略来管理告警的重发，对于没有发送成功的告警将不断重试，直到超时。

#### Set Notify
- `Set Notifies Stage`用来设置发送告警的信息到`nfLog`，该模块仅仅用于被该`Alert Manager`发送的告警的记录（`Retry`组件传递的`alerts`和`Dedup`组件中发送出去的告警信息）。

- `Integration`定义一个集成路由组件，包含用户的配置信息和名称以及发送告警的实现。自定义的notify路由需要满足该Notifier接口，实现Notify方法。 

---
### Examples
```
groups:

- name: httpd
  rules:
  - alert: httpd_down
    expr: probe_success{instance="http://httpd:80",job="httpd"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "httpd is down"

```
- Prometheus Server will wait for *1m* and if the expression meets the condition for 1m, now Prometheus Server has to fire alert and forward to AlertManager. Until now, Prometheus knows how to connect to AlertManager and when to fire and forward alerts to AlertManager.
```
route:
  repeat_interval: 2h
  receiver: email-1
  routes:
    - match:
        alertname: httpd_down
      receiver: email-1

    - match:
        alertname: nginx_down
      receiver: email-2
```
- These routes and receivers are defined in the AlertManager configuration file by the parent element called **route**. The parent route element has child **routes** which an alert follows in order to reach to its receiver based upon the *match label* as we will see in a bit. 

```
http://localhost:9090/api/v1/alertmanagers
{
  "status": "success",
  "data": {
    "activeAlertmanagers": [
      {
        "url": "http://127.0.0.1:9093/api/v1/alerts"
      }
    ],
    "droppedAlertmanagers": []
  }
}
```
```
curl -X GET http://10.255.101.73:9090/api/v1/alerts
{
  "status": "success",
  "data": {
    "alerts": [
      {
        "labels": {
          "alertname": "内存使用率过高",
          "instance": "127.0.0.1:9100",
          "job": "node",
          "severity": "warning"
        },
        "annotations": {
          "description": "127.0.0.1:9100 of job node内存使用率超过80%,当前使用率[59.74335527485338].",
          "summary": "Instance 127.0.0.1:9100 内存使用率过高"
        },
        "state": "firing",
        "activeAt": "2019-08-23T11:27:34.027571952Z",
        "value": 59.74335527485338
      }
    ]
  }
}
```
```
http://10.255.101.73:9093/api/v2/alerts
[
  {
    "annotations": {
      "description": "10.255.101.74:8051 of job default go_goroutines > 100,当前go_goroutines: [137].",
      "summary": "Instance 10.255.101.74:8051 go_goroutines > 100"
    },
    "endsAt": "2019-08-31T06:19:27.344Z",
    "fingerprint": "0d38358ac4713623",
    "receivers": [
      {
        "name": "default-receiver"
      }
    ],
    "startsAt": "2019-08-23T11:28:27.344Z",
    "status": {
      "inhibitedBy": [],
      "silencedBy": [],
      "state": "active"
    },
    "updatedAt": "2019-08-31T14:16:27.348+08:00",
    "generatorURL": "http://ceph-1:9090/graph?g0.expr=go_goroutines+%3E+100&g0.tab=1",
    "labels": {
      "alertname": "go_goroutines大于100",
      "instance": "10.255.101.74:8051",
      "job": "default",
      "severity": "warning"
    }
  }
]
```

```
http://10.255.101.73:9093/api/v1/alerts
{
  "status": "success",
  "data": [
    {
      "labels": {
        "alertname": "go_goroutines大于100",
        "instance": "10.255.101.74:8051",
        "job": "default",
        "severity": "warning"
      },
      "annotations": {
        "description": "10.255.101.74:8051 of job default go_goroutines > 100,当前go_goroutines: [135].",
        "summary": "Instance 10.255.101.74:8051 go_goroutines > 100"
      },
      "startsAt": "2019-08-23T11:28:27.344378042Z",
      "endsAt": "2019-08-31T06:21:27.344378042Z",
      "generatorURL": "http://ceph-1:9090/graph?g0.expr=go_goroutines+%3E+100&g0.tab=1",
      "status": {
        "state": "active",
        "silencedBy": [],
        "inhibitedBy": []
      },
      "receivers": [
        "default-receiver"
      ],
      "fingerprint": "0d38358ac4713623"
    }
  ]
}
```

### References
- [AlertManager github](https://github.com/prometheus/alertmanager)
- [SENDING ALERTS](https://prometheus.io/docs/alerting/clients/)
- [CONFIGURATION](https://prometheus.io/docs/alerting/configuration/)
- [NOTIFICATION TEMPLATE REFERENCE](https://prometheus.io/docs/alerting/notifications/)
- [Understanding Prometheus AlertManager](https://kjanshair.github.io/2018/07/26/prometheus-alert-manager/)
- [Prometheus高可用(4)：Alertmanager高可用](https://ylzheng.com/2018/03/12/prometheus-alertmanager-ha/)
- [Prometheus AlertManager代码阅读笔记](https://blog.csdn.net/u014029783/article/details/80654727)
- [Prometheus AlertManager代码阅读笔记 Notify组件](https://blog.csdn.net/u014029783/article/details/80663092)
- [Alertsnitch](https://gitlab.com/yakshaving.art/alertsnitch)
- [Prometheus integrations](https://github.com/prometheus/docs/blob/master/content/docs/operating/integrations.md)
- [Understanding and extending prometheus alertmanager](https://www.slideshare.net/leecalcote/understanding-and-extending-prometheus-alertmanager)
- [Hands-On Infrastructure Monitoring with Prometheus](https://books.google.com.pe/books?id=vTmbDwAAQBAJ&pg=PA295&lpg=PA295&dq=alertmanager+notify+provider&source=bl&ots=m45ik8xem9&sig=ACfU3U0hHJ9VxdDip60jq0dTQ0Ka2HjI3g&hl=zh-CN&sa=X&ved=2ahUKEwi6tK_l6a_kAhWPtlkKHTwqB-kQ6AEwCXoECAkQAQ#v=onepage&q=alertmanager%20notify%20provider&f=false)
- [搞搞 Prometheus: Alertmanager](https://aleiwu.com/post/alertmanager/)