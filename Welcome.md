# 基于Prometheus + Grafana + Exporter 的监控方案

本文主要介绍如何使用prometheus+grafana+exporter实现系统监控

[TOC]

## 第1章 背景

监控系统对于大数据平台的重要性不言而喻。那要实现这样一种系统，我们需要解决哪些问题？

- 首先我们要知道如何采集监控数据，监控数据主要有三种
  - 系统本身的运行状态，例如CPU、内存、磁盘、网络的使用情况
  - 各种应用的运行状况，例如数据库、容器等
  - 处理网络上发送过来的数据
- Ss监控数据存储：我们需要采用合适的存储方案来保存海量的监控数据
- 数据可视化：然后需要把这些数据在web界面进行展示，把监控指标的变化情况可视化
- 告警：如果监控系统只能看而不能及时发出告警（以邮件／微信等通知方式），价值也大打折扣
- 最后，对于这样的大型架构，我们同样需要考虑高可用／高并发／可伸缩性

## 第2章 为什么选择 Prometheus

[Prometheus](https://prometheus.io/) 是一个开源的完整监控解决方案，作为新一代开源解决方案，很多理念与 Google SRE 运维之道不谋而合，其对传统监控系统的测试和告警模型进行了彻底的颠覆，形成了基于中央化的规则计算、统一分析和告警的新模型。

**主要功能**

-  Prometheus 核心部分只有一个单独的二进制文件，不存在任何的第三方依赖，因此不会有潜在级联故障的风险。
-  Prometheus 基于 HTTP协议+Pull 模型的架构方式，拉取数据，简单易懂。

-  支持多种统计数据模型，图形化友好。

**Prometheus vs Zabbix**

- Zabbix 使用的是 C 和 PHP, Prometheus 使用 Golang, 整体而言 Prometheus 运行速度更快一点
- abbix 属于传统主机监控，主要用于物理主机，交换机，网络等监控，Prometheus 不仅适用主机监控，还适用于 Cloud, SaaS, Openstack，Container 监控。

**Prometheus vs Graphite**

- Graphite功能较少，它专注于两件事，存储时序数据， 可视化数据，其他功能需要安装相关插件，而 Prometheus 属于一站式，提供告警和趋势分析的常见功能，它提供更强的数据存储和查询能力。
- 在水平扩展方案以及数据存储周期上，Graphite 做的更好。

**Prometheus 性能消耗较低**

 prometheus 2.19 版以后，进行了捏村优化，整体对实例资源消耗较低，避免对其他服务的影响

## 第3章 方案框架

**基础架构**



![Prometheus architecture](https://prometheus.io/assets/architecture.png)



从这个架构图，也可以看出 Prometheus 的主要模块包含， 采集层、存储计算层、应用层

**采集层**

采集层分为两类，一类是生命周期较短的作业，还有一类是生命周期较长的作业。

-  短作业：直接通过 API，在退出时间指标**推送**给PushGateway 。
- 长作业：Retrieval 组件直接从 Job 或者 Exporter **拉取**数据，例如汇报机器数据的 node_exporter, 汇报 MongoDB 信息的 [MongoDB exporter](https://github.com/dcu/mongodb_exporter) 等等。

**存储计算层**

-  Prometheus Server，里面包含了存储引擎和计算引擎。
-  Retrieval 组件为取数组件，它会主动从 PushGateway 或者 Exporter 拉取指标数据。
- Service discovery，可以动态发现要监控的目标。
- TSDB，数据核心存储与查询。

**应用层**

- AlertManager：监控报警系统，通过定制可实现短信报警、5 分钟无人 ack 打电话通知、仍然无人 ack，通知值班人员 Manager…Emial，发送邮件… …
-  数据可视化：Prometheus build-in WebUI、Grafana、其他基于API 开发的客户端

## 第2章 Prometheus 安装

#### **2.1.下载 Prometheus**

进入https://prometheus.io/download/官网 选取部署平台和版本，我们将Prometheus部署在了Mac机器，故选取darwin版本安装包，此外Prometheus也支持Windows、Linux等平台。

![image-20220107212314295](/Users/geren/Library/Application Support/typora-user-images/image-20220107212314295.png)

将安装包解压成功后，可以运行version 检查运行环境是否正常

```
prometheus.exe --version 
```

如果你看到类似输出，表示你已安装成功:

```
prometheus, version 2.32.0-rc.1 (branch: HEAD, revision: 755efd38f9a716bea5127d13319cf5c51112b4eb)
  build user:       root@9cb2acf4df0c
  build date:       20211207-16:43:10
  go version:       go1.17.3
  platform:         windows/amd64
```

#### **2.2 启动 Prometheus Server**

```
./prometheus --config.file=prometheus.yml
```

这里使用默认配置文件启动，默认端口为9090，后面章节介绍自定义配置文件方法

**打开** **web** **页面查看**

浏览器输入：http://127.0.0.1:9090， 将看到如下页面

![prometheus-graph.png](https://songjiayang.gitbooks.io/prometheus/content/images/install/prometheus-graph.png)



## 第3章 监控Linux机器

在 Prometheus 中负责数据汇报的程序统一叫做 Exporter, 而不同的 Exporter 负责不同的业务。 它们具有统一命名格式，即 xx_exporter, 例如负责Linux系统监控的 node_exporter。

目前我们在测试环境10.237.6.94 10.237.6.84 10.237.6.85 10.237.6.86 10.237.6.87 10.237.120.44等六台机器部署了node_exporter

#### **3.1 安装Node Exporter**

进入https://prometheus.io/download/ 获取安装包，或者通过wget方式获取

![image-20211222095459031](C:\Users\liding\AppData\Roaming\Typora\typora-user-images\image-20211222095459031.png)

下载完成后，将node_exporter-1.3.1.linux-amd64.tar.gz文件存在至Linux路径并进行解压

```
tar -xvzfnode_exporter-1.3.1.linux-amd64.tar.gz
cd node_exporter-0.14.0.linux-amd64
```

#### **3.2 启动Node Exporter**

后台运行启动node_exporter

```
nohup ./node_exporter > node_exporter.log 2>&1 &
```

此时可以通过其他浏览器访问的9100端口查询，如下页面即为安装成功

![image-20211222100359388](C:\Users\liding\AppData\Roaming\Typora\typora-user-images\image-20211222100359388.png)

#### **3.3 注册服务至Prometheus**

打开部署Prometheus机器的安装目录，将所有Node Exporter的ip:port 添加至prometheus.yml，如下

![image-20211222102340444](C:\Users\liding\AppData\Roaming\Typora\typora-user-images\image-20211222102340444.png)

重启Prometheus

```
./prometheus --config.file=prometheus.yml
```

#### **3.4 检查Node Exporter状态**

此时可以通过其他浏览器访问的Prometheus服务端口查询所有Node Exporter状态，进入页面后点击Status->Targets，即可查询到所有Node Exporter节点状态

![image-20211222102608575](C:\Users\liding\AppData\Roaming\Typora\typora-user-images\image-20211222102608575.png)



## 第4章 监控Windows机器

在 Prometheus 中负责数据汇报的程序统一叫做 Exporter, 而不同的 Exporter 负责不同的业务。 它们具有统一命名格式，即 xx_exporter, 例如负责Windows系统监控的wmi_exporter

目前我们在测试环境10.237.6.95 10.237.6.91 10.237.6.92 10.237.6.93 等四台机器部署了node_exporter

#### **4.1 安装wmi_exporter**

进入https://github.com/prometheus-community/windows_exporter/releases获取安装包，或者通过wget方式获取, 双击安装后，exporter 自动运行，可以在服务列表里看到。

![image-20211222101627385](C:\Users\liding\AppData\Roaming\Typora\typora-user-images\image-20211222101627385.png)

Windows 默认 9182 端口。访问 http://127.0.0.1:9182/metrics 显示以下数据说明数据采集器安装成功。

![image-20211222101441758](C:\Users\liding\AppData\Roaming\Typora\typora-user-images\image-20211222101441758.png)

#### **4.2 注册服务至Prometheus**

打开部署Prometheus机器的安装目录，将所有Windows Exporter的ip:port 添加至prometheus.yml，如下

![image-20211222102824677](C:\Users\liding\AppData\Roaming\Typora\typora-user-images\image-20211222102824677.png)

重启Prometheus

```
./prometheus --config.file=prometheus.yml
```

#### **4.3 检查Windows Exporter状态**

此时可以通过其他浏览器访问的Prometheus服务端口查询所有Windows Exporter状态，进入页面后点击Status->Targets，即可查询到所有Windows Exporter节点状态

![image-20211222102945774](C:\Users\liding\AppData\Roaming\Typora\typora-user-images\image-20211222102945774.png)

## 第5章 监控MongoDB机器

在 Prometheus 中负责数据汇报的程序统一叫做 Exporter, 而不同的 Exporter 负责不同的业务。 它们具有统一命名格式，即 xx_exporter, 例如负责MongoDB监控的mongodb _exporter。

目前我们在测试环境10.237.6.84机器上部署了mongodb，其中10.237.6.84:10001为Primay，10.237.6.84:10002为副本集

#### **5.1 安装mongodb_exporter**

进入 https://github.com/percona/mongodb_exporter/releases获取安装包

或者通过wget方式获取： wget https://github.com/percona/mongodb_exporter/releases/download/v0.7.1/mongodb_exporter-0.7.1.linux-amd64.tar.gz

**解压压缩包至安装目录**

```
tar xvzf mongodb_exporter-0.7.1.linux-amd64.tar.gz
```

#### **5.2 启用 MongoDB 身份验证**

为exporter创建一个具有集群监视器角色的**管理员帐户**

```
use admin
db.createUser(
  {
    user: "mongodb_exporter",
    pwd: "mongodb_exporter",
    roles: [
        { role: "clusterMonitor", db: "admin" },
        { role: "read", db: "local" }
    ]
  }
)
```

#### **5.3 启动 Mongodb_Exporter**

```
./mongodb_exporter --mongodb.uri=mongodb_exporter:mongodb_exporter@10.237.6.84:10001,10.237.6.84:10002 --web.listen-address=:9001 #配置副本集启动
```

#### **5.4 注册服务至Prometheus**

打开部署Prometheus机器的安装目录，将所有Mongodb_Exporter的ip:port 添加至prometheus.yml，如下

![image-20211222110532050](C:\Users\liding\AppData\Roaming\Typora\typora-user-images\image-20211222110532050.png)

重启Prometheus

```
./prometheus --config.file=prometheus.yml
```

#### **5.6 检查Mongodb_Exporter 状态**

此时可以通过其他浏览器访问的Prometheus服务端口查询所有Mongodb_Exporter 状态，进入页面后点击Status->Targets，即可查询到所有Mongodb_Exporter 节点状态

![image-20211222110615552](C:\Users\liding\AppData\Roaming\Typora\typora-user-images\image-20211222110615552.png)

#### **5.7 常见问题**

1. **mongodb_exporter**用户没有必要的权限来对 admin 数据库执行查询。

   ![无法获取服务器状态：管理员未授权执行命令](https://devconnected.com/wp-content/uploads/2019/06/errors.png)

   使用 admin 数据库，并确保正确配置了**mongodb_exporter**用户（必须在 admin 数据库上具有集群监视器权限）

   **重启Prometheus**

   ```
   $ mongo --port 10001
   $ use admin;
   $ db.getUsers()
   {
           "_id" : "admin.mongodb_exporter",         
           "user" : "mongodb_exporter",              
           "db" : "admin",                           
           "roles" : [                               
                   {                                 
                           "role" : "clusterMonitor",
                           "db" : "admin"            
                   },                                
                   {                                 
                           "role" : "read",          
                           "db" : "local"            
                   }                                 
           ]                                         
   }               
   ```

   

2. 启动**mongodb_exporter** 报错： Could not get MongoDB BuildInfo: no reachable servers!

   ![Could not get MongoDB BuildInfo: no reachable servers!](https://devconnected.com/wp-content/uploads/2019/06/error-2.png)

   原因为： mongodb实例未启动或者未bindip，默认为localhost不允许其他连接

3. 启动mongodb_exporter报错： `Failed to get local.oplog_rs collection stats.`

   请确保将所有的副本集都加入到启动命令中

   ```
   ./mongodb_exporter --mongodb.uri=mongodb_exporter:mongodb_exporter@10.237.6.84:10001,10.237.6.84:10002 --web.listen-address=:9001 #配置副本集启动
   ```

   **将所有Mongodb_Exporter 信息注册至Prometheus**

   请确认监控账户用户是否有合理的权限

   ```
   use admin
   db.createUser(
     {
       user: "mongodb_exporter",
       pwd: "mongodb_exporter",
       roles: [
           { role: "clusterMonitor", db: "admin" },
           { role: "read", db: "local" }
       ]
     }
   )
   ```



## 第6章 数据可视化

收集到数据只是第一步，如果没有很好做到数据可视化，有时很难发现问题。本章将介绍使用 Prometheus 自带的 web console 以及 grafana 来查询和展现数据。

#### **6.1 Prometheus Web**

Prometheus 自带了 Web Console， 安装成功后可以访问 `http://localhost:9090/graph` 页面，用它可以进行任何 PromQL 查询和调试工作，非常方便，例如：

![prometheus web console ](https://songjiayang.gitbooks.io/prometheus/content/images/visualiztion/prometheus-web-console.png)

![prometheus web graph ](https://songjiayang.gitbooks.io/prometheus/content/images/visualiztion/prometheus-web-graph.png)



通过上图不难发现，Prometheus 自带的 Web 界面比较简单，因为它的目的是为了及时查询数据，方便 PromeQL 调试。它并不是像常见的 Admin Dashboard，在一个页面尽可能展示多的数据，如果你有这方面的需求，不妨试试 Grafana。

#### 6.2 Grafana 使用

Grafana 是一套开源的分析监视平台，支持 Graphite, InfluxDB, OpenTSDB, Prometheus, Elasticsearch, CloudWatch 等数据源，其 UI 非常漂亮且高度定制化。这是 Prometheus web console 不具备的，在上一节中已经说明了选择它的原因。

##### 6.1 安装Grafana

从官网https://grafana.com/grafana/download下载grafana安装包，然后界面安装即可

![image-20220107213539150](/Users/geren/Library/Application Support/typora-user-images/image-20220107213539150.png)

##### 6.2 登录并设置 Prometheus 数据源

安装成功后，你可以打开页面 `http://localhost:3000`， 访问 Grafana 的 web 界面

使用默认账号 admin/admin 登录 grafana

添加data sources，点击添加选择prometheus即可

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190613123218854.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWR1XzM2OTQzMDc1,size_16,color_FFFFFF,t_70)

添加后配置相关信息即可，写入prometheus的URL，点击“Save&Test”提示绿色成功即可![image-20211222131748929](C:\Users\liding\AppData\Roaming\Typora\typora-user-images\image-20211222131748929.png)

配置成功后，点击Save&test选项进行测试，若出现“Data source is working”即证明Grafana 已经和 Prometheus 连上了

![image-20211222133021748](C:\Users\liding\AppData\Roaming\Typora\typora-user-images\image-20211222133021748.png)



此时仍然无法观察各监控节点的信息dashboard，需要我们导入dashboard模板。 

##### 6.3 添加Dashboard模板

进入 https://grafana.com/dashboards 页面，基于喜好下载模板json文件，在grafana中导入json文件，即可以看到酷炫又详细的监控页

导入Prometheus仪表版，Dashboards–Manage–import

![image-20211222132029183](C:\Users\liding\AppData\Roaming\Typora\typora-user-images\image-20211222132029183.png)

改仪表版名称和选择数据源为Prometheus name即可（如果这里提示没有数据库，就是前面的data sources没有添加好需要重新检查）

![image-20211222151137154](C:\Users\liding\AppData\Roaming\Typora\typora-user-images\image-20211222151137154.png)

进入仪表板就可以在仪表版看到相应的监控

![image-20211222132307407](C:\Users\liding\AppData\Roaming\Typora\typora-user-images\image-20211222132307407.png)

##### 6.3 常见问题

1. Exporter采集的部分数据无法在dashboard显示

   由于Exporter与Dashboard json模板中相同的metric值定义方式不相同，比如空闲内存属性举例，在node_exporter中定义为node_memory_MemAvailable_bytes，而在json文件中该指标定义为node_memory_MemFree_bytes，需要修改json文件中字段值即可。

## 第7章 监控告警

目前，测试环境中的监控平台未接入企业微信，下述为生产环境监控的告警模块解决方案

在 Prometheus 中告警分为两部分:

- Prometheus 服务根据所设置的告警规则将告警信息发送给 Alertmanager。
- Alertmanager 对收到的告警信息进行处理，包括去重，降噪，分组，策略路由告警通知。

使用告警服务主要的步骤如下：

- 下载配置 Alertmanager。
- 通过设置 `-alertmanager.url` 让 Prometheus 服务与 Alertmanager 进行通信。
- 在 Prometheus 服务中设置告警规则。
- 目前Alertmanager支持Email、企业微信、Slack、Webhook接收告警，但由于生产环境网络隔离，需要借助网关实现，借鉴eqcollect eqlog实现方式，配置alertmanager.yml中的webhook_configs为网关地址即可实现企业微信告警提示。



## 第8章 扩展

除了 node_exporter 我们还会根据自己的业务选择安装其他 exporter 或者自己编写，比较常用的 exporter 有 MySQL、JMX、Redis、Flink等，更多 exporter 请参考[链接](https://prometheus.io/docs/instrumenting/exporters/)

此外Prometheus支持自定义程序exporter进行监控，使用方式参考[链接](https://prometheus.io/docs/introduction/overview/)



## 第9章 环境汇总

**软件版本**

prometheus: ***2.32.1.windows-amd64***

node_exporter: ***1.3.1.linux-amd64***

wmi_exporter: ***0.6.0.-amd64***

mongodb_exporter: **0.7.1.linux-amd64**



Demo体验地址：

Demo地址： **http://10.237.6.95:3000**  

账号：**admin**

密码：**admin** （请勿更改）

