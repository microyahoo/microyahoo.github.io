## Open Faclon
---

### Architecture
![image](http://book.open-falcon.com/zh_0_2/image/func_intro_1.png)
![系统流图](https://raw.githubusercontent.com/niean/niean.common.store/master/images/open-falcon/nodata/nodata.flow.png)
![模块结构](https://raw.githubusercontent.com/niean/niean.common.store/master/images/open-falcon/nodata/nodata.module.png)
![部署架构](https://raw.githubusercontent.com/niean/niean.common.store/master/images/open-falcon/nodata/nodata.deploy.png)

---
#### Faclon-agent
`Falcon-agent`用于数据的采集，它会定期地将`metric`数据通过`jsonRPC`上报到`Transfer`，其上报的`metric`数据格式为：
##### Data model
`open-falcon`采用和`opentsdb`相同的数据格式：metric、endpoint加多组key value tags。例如：
```
{
    metric: load.1min,
    endpoint: open-falcon-host,
    tags: srv=falcon,idc=aws-sgp,group=az1,
    value: 1.5,
    timestamp: `date +%s`,
    counterType: GAUGE,
    step: 60
}
```
---
#### Transfer
- `Transfer`接收到`agent`上报的`metric`数据后，经过一定的处理（将tags字符串转化为map），然后上报到`Graph`, `Judge`和`TSDB`。
    - 提供数据接收接口供Agent和自定义脚本push监控数据。
    - 对接收到的数据做合法性校验，规整。
    - 针对每个后端实例维护一个RPC连接池。
    - 准备内存Queue中转监控数据。
    - 根据一致性哈希将Queue中的数据转发给Judge和Graph。
    - 当后端宕机的时候做少量数据缓存，提供重试机制。

--- 
#### Heartbeat Server
- HBS主要有如下功能：
    - `agent`发送心跳信息给`HBS`的时候，会把hostname、ip、agent version、plugin version等信息告诉`HBS`，`HBS`负责更新host表。
    - 将IP白名单分发到所有的agent。
    - 告诉各个agent应该执行哪些插件。
    - 告诉各个agent应该监听哪些端口，进程。
    - 缓存监控策略。
        - `HBS`去获取所有的报警策略缓存在内存里，然后`Judge`去向`HBS`请求。其中`Judge`通过rpc调用来获取。
            - 在配置报警策略的时候配置了报警级别，比如P0/P1/P2等等，每个级别的报警都会对应不同的redis队列。 

---
#### Judge
- 因为监控系统数据量比较大，一台机器显然是搞不定的，所以必须要有个数据分片方案。`Transfer`通过`一致性哈希`来分片，每个`Judge`就只需要处理一小部分数据就可以了。
- 同时`Judge`会定期地通过RPC从`Heartbeat Server`同步`Strategy`和`Expression`用于告警的判定。
- `Judge`接收到`Transfer`的`metric`数据后首先是否有针对其的`Strategy`和`Expression`，如果没有则直接跳过，如果有则进行告警判定。
- `Judge`会将判定结果以`Event`的形式发送到Redis缓存中。

--- 
#### Alarm
- `Alarm`从redis缓存中获取`Judge`推送的Event后，根据Event的优先级分别进行处理。
- `Alarm`从redis队列分别获取高优先级的Event和低优先级的Event，其分别位于`highQueues`和`lowQueues`，其中`highQueues`中配置的几个event队列中的事件是不会做告警合并的，`lowQueues`会做告警合并。
    - 按照优先级轮询Redis读取告警事件。
    - 对需要回调的告警事件回调业务系统接口。
    - 对高优先级告警事件直接生成告警邮件和短信。
    - 对低优先级告警事件做合并。
    - 提供web页面展示当前未恢复的告警。
- 制定邮件，短信发送的接口规范。

---
#### Nodata
- nodata用于检测监控数据的上报异常。nodata和实时报警judge模块协同工作，过程为: 配置了nodata的采集项超时未上报数据，nodata生成一条默认的模拟数据；用户配置相应的报警策略，收到mock数据就产生报警。采集项上报异常检测，作为judge模块的一个必要补充，能够使judge的实时报警功能更加可靠、完善。
- 


---
#### Graph


---
### References
- [Open-falcon](http://open-falcon.org/)
- [falcon-plus](https://github.com/open-falcon/falcon-plus/)
- [Falcon+ API](http://open-falcon.org/falcon-plus/)
- [Nodata](https://github.com/open-falcon-archive/nodata)
- [文档](http://book.open-falcon.org/zh/install_from_src/hbs.html)
- [心跳服务器 Falcon-HBS 源码解读视频](http://www.jikexueyuan.com/course/1873.html)
- [open-falcon 2.0文档](http://book.open-falcon.com/zh_0_2/intro/)
- [open-falcon部署与架构解析](http://www.jikexueyuan.com/course/1651.html)
- [数据采集模块 Falcon-Agent 源码解读](http://www.jikexueyuan.com/course/2242.html)
- [运维监控视频](http://www.jikexueyuan.com/course/operationmonitor/)