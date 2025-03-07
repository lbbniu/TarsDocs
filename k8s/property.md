# TarsCloud K8SFramework 特性

对 TarsCloud K8SFramework 项目目标来讲, 其需要实现的核心工作主要有三个

+ 定义 CRD 抽象 Tars 框架的的概念,动作.
+ 通过准入控制校验 CRD 对象的合法性,合理性.
+ 调谐 CRD 对象, 生成 Kubernetes 原生对象 (Service,StatefulSet,DaemonSet)
  基于此,我们主要从
+ CRD 的定义,准入控制和调谐
+ TServer 与Service ,StatefulSet,DaemonSet 的映射
+ 其他在开发,使用,运维中需要了解的特性
  等方面阐述 **TarsCloud K8SFramework** 特性

* [CRD](#crd)
  + [tserver](#tserver)
  + [tconfig](#tconfig)
  + [ttemplate](#ttemplate)
  + [taccount](#taccount)
  + [timage](#timage)
  + [tendpoint](#tendpoint)
  + [texitedrecord](#texitedrecord)
  + [tframeworkconfig](#tframeworkconfig)
* [TServer 与 Service 的映射](#TServer与Service的映射)
* [TServer 与 StatefulSet 的映射](#TServer与StatefulSet的映射)
  + [tserver.k8s.affinity](#tserverk8saffinity)
  + [tserver.spec.k8s.env](#tserverspeck8senv)
  + [tserver.spec.k8s.envFrom](#tserverspeck8senvfrom)
  + [tserver.spec.k8s.Resources](#tserverspeck8sresources)
  + [tserver.spec.k8s.ImagePullPolicy](#tserverspeck8simagepullpolicy)
  + [tserver.spec.k8s.hostIPC](#tserverspeck8shostipc)
  + [tserver.spec.k8s.hostNetwork](#tserverspeck8shostnetwork)
  + [tserver.spec.k8s.serviceAccount](#tserverspeck8sserviceaccount)
  + [tserver.spec.k8s.podManagementPolicy](#tserverspeck8spodmanagementpolicy)
  + [tserver.spec.k8s.replicas](#tserverspeck8sreplicas)
  + [tserver.spec.k8s.updateStrategy](#tserverspeck8supdatestrategy)
  + [tserver.spec.k8s.nodeSelector](#tserverspeck8snodeselector)
  + [tserver.spec.k8s.notStacked](#tserverspeck8snotstacked)
  + [tserver.spec.k8s.hostPorts](#tserverspeck8shostports)
  + [tserver.spec.k8s.readinessGate](#tserverspeck8sreadinessgate)
  + [tserver.spec.k8s.launcherType](#tserverspeck8slaunchertype)
  + [tserver.spec.k8s.mounts](#tserverspeck8smounts)
* [TServer 与 DaemonSet 的映射](#TServer与DaemonSet的映射)
* [其他特性](#其他特性)
  + [镜像分类](#镜像分类)
  + [镜像构建](#镜像构建)
  + [磁盘管理](#磁盘管理)
  + [时区管理](#时区管理)

## CRD

### tserver

#### 描述

tserver 抽象了 tars 服务, 每个 tserver 对象代表了一项 tars 服务.

其定义 schema 位于 helm/tarscontroller/crds/tserver.yaml

典型的 tserver对象 如下:

```yaml
apiVersion: k8s.tars.io/v1beta3
kind: TServer
metadata:
  labels:
    tars.io/ServerApp: tars
    tars.io/ServerName: tarsconfig
    tars.io/SubType: tars
    tars.io/Template: tars.cpp
  name: tars-tarsconfig
  namespace: tars-dev
spec:
  app: tars
  server: tarsconfig
  subType: tars
  important: 3
  tars:
    template: tars.cpp
    profile: ""
    asyncThread: 3
    servants:
      - capacity: 10000
        connection: 10000
        isTars: true
        isTcp: true
        name: ConfigObj
        port: 11111
        thread: 3
        timeout: 60000
  normal:
    ports:
      - isTcp: true
        name: http
        port: 3000
  k8s:
    abilityAffinity: AppOrServerPreferred
    daemonSet: false
    env:
      - name: Namespace
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
      - name: PodName
        valueFrom:
          fieldRef:
            fieldPath: metadata.name
    hostIPC: false
    hostNetwork: false
    imagePullPolicy: Always
    launcherType: background
    mounts:
      - mountPath: /usr/local/app/tars/app_log
        name: host-log-dir
        readOnly: false
        source:
          hostPath:
            path: /usr/local/app/tars/app_log
            type: DirectoryOrCreate
        subPathExpr: $(Namespace)/$(PodName)
    nodeSelector: [ ]
    notStacked: false
    podManagementPolicy: Parallel
    readinessGate: [ tars.io/active ]
    replicas: 1
    resources: { }
    serviceAccount: tars-tarsconfig
  release:
    id: v1.2.2
    image: docker.tarsyun.com/tarscloud/tars.tarsconfig:v1.2.2
    nodeImage: docker.tarsyun.com/tarscloud/tars.tarsnode:v1.2.2
    secret: ""
    time: "2022-03-01T08:00:00Z"
status:
  currentReplicas: 1
  readyReplicas: 1
  replicas: 1
  selector: tars.io/ServerApp=tars,tars.io/ServerName=tarsconfig

```

tserver 对象的组成:

+ spec.app, spec.server 确定了一个 tars 服务的名字

+ spec.subType 确定了一个 tars 服务的子类型

  > spec.subType 支持的值有 [ tars, normal ]
  > 通常情况下, 一套 tars 框架中可能包含了若干非 tars 类型的服务,比如 tarsweb, mysql 或其他第三方服务.
  > 在原生 tars 中, 这些非 tars 类型的服务无法通过 tars 管理平台运维.
  > 在 tarsk8s 中,我们对此做了改进, 并做了如下设定:
  > 如果服务程序的进程由 tarsnode 守护则其 subType 为 tars
  > 如果服务程序进程非由 tarsnode 守护则其 subType 为 normal

+ spec.tars 域

  > sepc.tars 域描述一个 tars 服务必须包含的属性, 比如模板,私有模板,Servants等
  > 只有 spec.subType 值为 tars 时, spec.tars 域才有合法值.

+ sepc.normal 域

  > spec.normal 域描述一个 normal 服务需要对外暴露的网络端口
  > 只有 spec.subTyep 值为 normal 时, spec.normal 域才有合法值

+ spec.k8s 域

  > spec.k8s 域的每个字段都会影响 tars 服务如何在 k8s 集群内以什么样的形式运行,
  > 比如服务在运行时需要定义哪些环境变量, 需要挂载磁盘或外部文件, 服务的运行副本数, 服务的节点筛选条件等.
  > 我们在 "TServer 与 Service ,Statefulset 的映射" ,"TServer 与 Daemonset 的映射" 中介绍该域每个字段的含义和影响

+ spec.release 域

  > 在 tars 的使用中, 成功部署一个服务, 还需要用户发布指定某个版本, 然后服务才会开始启动.
  > tserver.spec.release 模拟了这一个过程, 用户填充该域意味着发布了一个服务版本,然后才会有 Pod 生成

+ statud 域

  > spec.status 域描述了属于该服务的当前 pod 数量情况.

#### 准入控制:

Mutating

+ 为每个 tserver 添加 tars.io/ServerApp=$(spec.app) 标签
+ 为每个 tserver 添加 tars.io/ServerName=$(spec.server) 标签
+ 为每个 tserver 添加 tars.io/SubType=$(spec.subType) 标签
+ 为每个 spec.subType==tars 的 tserver 添加 tars.io/Template=$(spec.tars.template) 标签
+ 为每个 spec.subType==tars 的 tserver 重设 spec.k8s.readinessGate=tars.io/active
+ 为每个 spec.release == null 的 tserver 重设 spec.k8s.replicas=0
+ 为每个 spec.k8s.hosPorts!=null 或 spec.k8s.hostIPC==true 的 tserver 重设 spec.k8s.notStacked=true
+ 为每个 spec.k8s.replicas > $(annotations["tars.io/MaxReplicas"]) 的 tserver 重设 spec.k8s.replicas = $(
  annotations["tars.io/MaxReplicas"])
+ 为每个 spec.k8s.replicas < $(annotations["tars.io/MinReplicas"]) 的 tserver 重设 spec.k8s.replicas = $(
  annotations["tars.io/MinReplicas"])

Validating

+ 禁止修改 spec.app, spec.server, spec.subType 字段
+ 禁止删除 spec.tars ,spec.normal, spec.k8s 域
+ 禁止 spec.tars.servant 之间有重复的 name 值
+ 禁止 spec.tars.servant 之间有重复的 port 值
+ 禁止 spec.tars.servant 的 port 等于保留值  (19385)
+ 禁止 spec.normal.port 之间有重复的 name 值
+ 禁止 spec.normal.port 之间有重复的 port 值
+ 禁止 spec.k8s.mounts 之间有重复的 name 值
+ 如果 spec.subTyep == tars, 禁止 spec.tars.template 引用不存在 ttemplate 资源
+ 如果 spec.k8s.hostPorts!=null , 禁止 spec.k8s.hosPorts 引用不存在的 spec.tars.servants.name 或 spec.normal.ports.name
+ 如果 spec.k8s.hostPorts!=null , 禁止 spec.k8s.hosPorts 之间有重复的 port 值
+ 如果 spec.k8s.daemonSet == true, 禁止使用 persistentvolumeclaimtemplate 和 tlv 类型的磁盘挂载

#### 调谐过程

1. controller 监测到新的 tserver 对象后, 将 spec.app 值, 追加到 ttree/tars-tree.apps 中
2. controller 监测 tserver 对象新建后, 新建同名 tars,k8s.io/tendpoint, core/service, apps/statefulset 或 apps/daemonset 对象
4. controller 监测 tserver 对象变动后, 重设同名 tars,k8s.io/tendpoint, core/service, apps/statefulset 或 apps/daemonset 对象
5. controller 监测 tserver 对象删除后, 删除同名 tars,k8s.io/tendpoint, core/service, apps/statefulset 或 apps/daemonset 对象

### tconfig

#### 描述

tconfig 抽象 tars 服务配置属性, 每个 tconfig 对象代表了一项 tars 服务配置.

典型的 tconfig 配置如下:

```yaml
apiVersion: k8s.tars.io/v1beta3
kind: TConfig
metadata:
  labels:
    tars.io/Activated: "false"
    tars.io/ConfigName: config.json
    tars.io/PodSeq: m
    tars.io/ServerApp: Git
    tars.io/ServerName: GitBookServer
    tars.io/Version: 20220117113532-77fcbf18
  name: git-gitbookserver-xesdig
  namespace: tars-dev
app: Git
server: GitBookServer
podSeq: m
configContent: |
  {
      "cn_git": "/usr/local/server/bin/docs/TafDocs",
      "en_git": "/usr/local/server/bin/docs/TafDocs_en"
  }
configName: config.json
activated: false
version: 20220117113532-77fcbf18
```

- tconfg.app, tconfg.server 表明 这是属于 tserver/${tconfig.app}-$(tconfig.server) 的配置

- 如果 tconfig.server="", 则这是一项应用级 tconfig

- 如果 tconfig.podSeq==[%d]+ , 表明这是一项节点级 tconfig , 从属于另一项具有相同 tconfig.app, tconfig.server, tconfig.configName 值但
  tconfig.podSeq=m 的 主 tconfig

- tconfig.version 由 mutating 服务管理, validating 服务禁止用户修改此字段

- validating 服务禁止用户修改一个已有 tconfig. 如果有需要更新 tconfig.configContent, 请遵循 "新建替代修改" 策略

  > "新建替代修改" 策略是指, 如果您有一项 tconfig.configContent 需要更新,那么您需要按以下步骤操作:
  >
  > 1. 新建一个 tconfig.app, tconfig.server, tconfig.configName, tconfig.podSeq 与待更新 tconfig 相同的 tconfig
  > 2. 将新的 tconfig.configContent 设定为目标内容
  > 3. 提交新的 tconfig 到 k8s 集群
  > 4. validating 服务和 controller 服务会自动激活新提交的 tconfig , 并且将其他具有相同 tconfig.app, tconfig.server, tconfig.configName, tconfig.podSeq 值的 tconfig 重设 tconfig.activate=false

  > "新建替代修改" 策略的目标是为了实现配置回滚功能

- 如果你需要回滚某项配置到历史版本, 可以通过标签筛选出该项配置的所有历史版本, 选定目标 tconfig 后, 更改其 tconfig.activated=true

  > 具体操作步骤:

  > 1. 筛选某项配置的所有版本:
  >
  > ```shell
  > kubectl get tconfig -n ${namespace} -l 'tars.io/ServerApp=$(app),tars.io/ServerName=$(server),tars.io/configName=$(configName),tars.io/podSeq=$(podSeq)'
  > ```
  >
  > 2. 修改目标项的 tconfig.activate=true

  >  ```shell
  >  kubectl edit tconfig -n ${namespace} ${name}
  >  ```

#### 准入控制

Mutating

+ 为每个 tserver 添加 tars.io/ServerApp=$(tconfig.app) 标签
+ 为每个 tserver 添加 tars.io/ServerName=$(tconfig.server) 标签
+ 为每个 tconfig 添加 tas.io/Activate=$(tconfig.activated) 标签
+ 为每个 tconfig 添加 tas.io/ConfigName=$(tconfig.configName) 标签
+ 为每个 podSeq="" 或 podSeq=null 的 tconfig 添加 tar.io/PodSeq: m 标签
+ 为每个 podSeq!="" 的 tconfig 添加 tar.io/PodSeq=$(tconfig.podSeq) 标签
+ 添加 tars.io/Version=$(tconfig.version) 标签

Validating

+ 禁止修改 tserver.app, tserver.server, tserver.configName, tserver.podSeq, tserver.version 值

+ 禁止新建主配置不存在的节点配置

+ 禁止删除被节点配置引用的主配置

+ 每当新增了一项 activated=true 的 tconfig 时, 为具有相同 tserver.app, tserver.server, tserver.configName, tserver.podSeq 值的 其他
  tconfig 添加 tars.io/Deactivating="" 标签

+ 每当删除一项 activated=true 的 tconfig 时, 为具有相同 tserver.app, tserver.server, tserver.configName, tserver.podSeq 值的 其他 tconfig
  添加 tars.io/Deleting="" 标签

#### 调谐流程

+ 删除所有包含 tars.io/Deleting="" 标签的 tconfig
+ 为所有包含 tars.io/Deactivating="" 标签的 tconfig 重设 tconfig.activated=false
+ 根据 tframework.recordLimit.tconfigHistory 限定每项业务配置的的最大版本数

### ttemplate

#### 描述

ttemplate 定义了一项 tars 模板配置属性, 每提交一个 ttemplate 对象意味则为增加了一项模板配置

典型的 ttemplate 配置如下:

```yaml
apiVersion: k8s.tars.io/v1beta3
kind: TTemplate
metadata:
  labels:
    tars.io/Parent: tars.default
  name: tars.cpp
  namespace: tars
spec:
  content: ""
  parent: tars.default
```

spec.content 字段描述了模板的内容 spec.parent 字段描述了模板的父模板, 如果没有父模板,则填充为自己

#### 准入控制

Mutating

+ 为 spec.parent!=$(metadata.name) 的 ttemplate 添加 tars.io/Parent: $(spec.parent) 标签

Validating

+ 禁止 spec.parent =null 或 spec.parent = ""
+ 禁止 spec.parent 引用不存在的 ttemplate
+ 禁止删除作为某项 ttemplate parent 的 ttemplate

#### 调谐流程

无

### taccount

#### 描述

每个 taccount 对象都代表这一个 tarsweb 用户账号

典型的 taccount 如下:

```yaml
apiVersion: k8s.tars.io/v1beta3
kind: TAccount
metadata:
  name: 21232f297a57a5a743894a0e4a801fc3
  namespace: tars-dev
spec:
  username: admin
  authentication:
    activated: true
    bcryptPassword: $2a$10$XFGqxTkb4GKsAYc9gYQpGutM9RIshW5KlkR.9hL0l9qPdDLyCeJXm
    tokens: [ ]
  authorization: [ ]
```

spec.username 表示账号名字.
spec.authentication.bcryptPassword 表示了账号的的认证密码, 其生成方式为 原始密码->sha1->bcrypt
spec.authentication.tokens 由 tarsweb 填充和管理.表示 用户登陆后的 一些 token信息
spec.authorization 表示用户的权限信息, 由tarsweb 填充和管理

在 spec.authorization 域中实际一个隐藏字段, spec.authorization.password ,表示原始密码.

如果您需要重设账号密码,可以通过此字段重新设置, 若如此, spec.authorization.bcryptPassword, spec.authorization.tokens 会被重置

#### 准入控制

Mutating

+ 如果 spec.authorization.password !="", 重置 spec.authorization.bcryptPassword, spec.authorization.tokens
+ 如果 spec.authorization.bcryptPassword 发生改变, 重置spec.authorization.bcryptPassword, spec.authorization.tokens

Validating

无

#### 调谐流程

无

### timage

#### 描述

每个 tserver 对象通过 label 与一个或多个timage 对象关联, 并在 timage 对象中记录服务发布版本及镜像地址等信息.

典型的 timage 如下:

```yaml
apiVersion: k8s.tars.io/v1beta3
kind: TImage
metadata:
  labels:
    tars.io/ImageType: server
    tars.io/ServerApp: tars
    tars.io/ServerName: tafconfig
  name: tars-tarsconfig
  namespace: tars-dev
imageType: server
releases:
  - createTime: "2022-01-17T02:22:15Z"
    id: v1b2b3
    image: tarscloud.com/tars-helm/tars.tarsconfig:v1b2b3
    secret: ""
  - createTime: "2021-12-30T07:04:35Z"
    id: beta202
    image: tarscloud.com/tars-helm/tars.tarsconfig:beta202
    secret: ""
  - createTime: "2021-12-30T05:49:22Z"
    id: alpha201
    image: tarscloud.com/tars-helm/tars.tarsconfig:alpha201
    secret: ""
```

timage.imageType 的可选值有 [ base, server, node ]

> tarsk8s 将镜像类型分为 [ 基础镜像, 服务镜像, Node镜像 ] , 分别有 base, server ,node 表示
> 服务镜像内包含了服务的启动程序或启动包
> 基础镜像内包含了服务程序启动需要的基础环境, 比如 nodejs 程序需要 node环境, java 程序需要 jdk/jre 环境
> 相比 imageType=server 的 timage , imageType=base 的 timage 增加了 timage.supportedType 域, 用于表明该 timage 包含哪些运行环境
> Node 镜像包含了 tarsnode 程序以及 pod 的 启动脚本

timage.releases 域用于记录了服务的每个发布版本

注意: 您需要自行维护 timage 对象与 tserver 对象的关联关系(通过标签)

#### 准入控制

Mutating

+ 为每个 timage 添加 tars.io/ImageType=$(timage.imageType) 标签
+ 如果 timage.releases 中存在 重复的 id , 保留位序靠前的 id, 删除位序靠后的 id
+ 如果 timage.releases 中 createTime 为空 , 填充为当前时间
+ 如果 timage.imageType=server , 添加 tars.io/Supported 标签

Validating

无

#### 调谐流程

无

### tendpoint

#### 描述

tendpont 的是 controller 为了记录服务状态而创建出来的对象, 用户不需要关注和使用此对象. tarsk8s不保证 tendpoint 的版本兼容性

#### 准入控制

无

Mutating

无

Validating

无

#### 调谐流程

controller 程序检测到 新建 tserver 后 ,会创建出 同名 tendpoint 对象

### texitedrecord

#### 描述

texitedrecord 记录了同名 tserver 的 pod 生命周期, 主要便于查询问题定位日志问题

典型的 texitedrecord 对象 如下:

```yaml
apiVersion: k8s.tars.io/v1beta3
kind: TExitedRecord
metadata:
  labels:
    tars.io/ServerApp: tars
    tars.io/ServerName: tafnotify
  name: tars-tafnotify
  namespace: tars
  ownerReferences:
    - apiVersion: k8s.tars.io/v1beta3
      blockOwnerDeletion: true
      controller: true
      kind: TServer
      name: tars-tafnotify
      uid: 38bac10c-0209-4866-ac33-dfbb39c5dd93
  resourceVersion: "15465766"
  uid: 1b0624bc-4053-481e-b005-41e83f11e739
app: tars
server: tafnotify
pods:
  - createTime: "2022-01-17T02:22:19Z"
    deleteTime: "2022-02-10T09:07:22Z"
    id: v1b2b3
    name: tars-tafnotify-0
    nodeIP: 172.16.8.94
    podIP: 10.0.2.163
    uid: 2bb730dc-9a48-4dbb-9125-4c25181c9a6e
  - createTime: "2022-01-14T07:27:02Z"
    deleteTime: "2022-01-17T02:22:46Z"
    id: beta202
    name: tars-tafnotify-0
    nodeIP: 172.16.8.94
    podIP: 10.0.2.121
    uid: 2ae04280-f076-4d6f-b828-27ddd6cfa16b
  - createTime: "2021-12-30T07:04:38Z"
    deleteTime: "2022-01-14T07:27:29Z"
    id: beta202
    name: tars-tafnotify-0
    nodeIP: 172.16.8.94
    podIP: 10.0.2.15
    uid: bee641fe-3db4-453f-8d16-ab480f5dd2c2
```

+ texitedrecord.app, texitedrecord.server 用于关联 tserver
+ texitedrecord.pods 记录了 tserver 的 pod生命周期信息及部分元信息

#### 准入控制

Mutating

无

Validating

无

#### 调谐流程

+ controller 检测到新增 tserver 后,新建同名 texitedrecord
+ controller 检测到属于 某个 tserver 的 pod 生命结束后, 收集pod 信息并追加到 texitedrecord 中
+ tframeworkconfig.recordLimit.texitedPod 限定了每项 texitedrecord.pods 最大长度

### tframeworkconfig

#### 描述

tframeworkconfig 用于定义 framework 级 配置, 每套 framework 有且只有唯一的 tframeworkconfig 对象 tars-framework

典型的 tframeworkconfig 对象如下:

```yaml
apiVersion: k8s.tars.io/v1beta3
kind: TFrameworkConfig
metadata:
  name: tars-framework
  namespace: tars
imageBuild:
  idFormat: ""
  maxBuildTime: 600
  executor:
    image: ""
    secret: ""
imageRegistry:
  registry: tarscloud/tars-helm
  secret: tars-image-secret
nodeImage:
  image: tarscloud/tars-helm/tars.tafnode:v1b2b3
  secret: ""
recordLimit:
  tconfigHistory: 32
  texitedPod: 32
  timageRelease: 60
upChain:
  Test.GetSum.GetSumObj:
    - host: 172.16.8.73
      port: 10007
      timeout: 3000
    - host: 172.16.8.74
      port: 10008
      timeout: 3000
  default:
    - host: 172.16.8.72
      port: 8888
      timeout: 3000
expand:
  nativeDBConfig: |
    {
      "show": true,
      "enable": true,
      "dbConf": {
        "host": "172.16.11.33",
        "port": "3306",
        "user": "tars",
        "password": "tars2015",
        "charset": "utf8",
        "pool": {
          "max": 10,
          "min": 0,
          "idle": 10000
          }
       }
     }
  nativeFrameworkConfig: |
    <taf>
        <application>
            #proxy需要的配置
            <client>
                #地址
                locator = tars.tafregistry.QueryObj@tcp -h 172.16.11.33 -t 60000 -p 17890 -t 3000
                sync-invoke-timeout = 20000
                #最大超时时间(毫秒)
                max-invoke-timeout = 60000
                #刷新端口时间间隔(毫秒)
                refresh-endpoint-interval = 300000
                #模块间调用[可选]
                stat = tars.tafstat.StatObj
                #网络异步回调线程个数
                asyncthread = 3
                modulename = tars.system
            </client>
        </application>
    </tars>
```

tframeworkconfig.imageBuild 定义 tarsimage 服务构建镜像时的参数

tframeworkconfig.imageRegistry 定义了tarsimage 服务上传镜像的仓库信息

tframeworkconfig.nodeImage 定义 启动 tars 服务时使用的默认 tarsnode 镜像信息

tframeworkconfig.recordLimit 定义了一些限制信息

+ tconfigHistory 描述了一项业务配置能保留的最大历史版本数
+ texitedPod 描述了一项 texitedrecord 能记录的最大 pod 数
+ timageRelease 描述了一项 timage 能记录的最大 releases 数

tframeworkconfig.upChain 定义了与外部集群的联通信息

> 在使用 tarsk8s 时, 您可能已有一套 tars 集群且运行了很多关键服务
> 您可以配置 upchain 来实现 tarsk8s 与 tars 的联通
> upchain 的配置格式请参考上文示例:
> Test.GetSum.GetSumObj 是指, 如果 tarsk8s 服务在 集群内寻址 "Test.GetSum.GetSumObj" 失败时, 取值 $(Test.GetSum.GetSumObj)  作为替代
> default 是指, 如果 tarsk8s 服务在集群内无法寻址任意 obj 失败, 且该 obj 没在 upchain 配置时, 取值 $(default) 作为替代

tframeworkconfig.expand 用于扩展域, 用户可以在这里配置任意 kv 对

> 示例 配置中 的 nativeDBConfig ,nativeFrameworkConfig 是 tarsweb 用来 控制是否展示 原生 tars 集群信息, 具体请参考 [<<TarsWeb 管理平台>>](tarsweb.md)
> 您如果有需要, 可以在此使用自定义的 kv 值

#### 准入控制

Mutating

无

Validating

无

#### 调谐流程

无

## TServer与Service的映射

当 tserver.k8s.daemonset==false 为时, controller 从一个 tserver 对象, 映射出同名 service 对象

典型的 service 对象 如下:

```
apiVersion: v1
kind: Service
metadata:
  labels:
    tars.io/ServerApp: tars
    tars.io/ServerName: tafconfig
  name: tars-tafconfig
  namespace: tars
spec:
  clusterIP: None
  selector:
    tars.io/ServerApp: tars
    tars.io/ServerName: tafconfig
  sessionAffinity: None
  type: ClusterIP
  ports:
  - name: configobj
    port: 11111
    protocol: TCP
```

spect.type 固定为 ClusterIP

clusterIP 固定为 None

sessionAffinity 固定为 None

selector 被映射为 tars.io/ServerApp: $(tserver.app)  , tars.io/ServerName: $(tserver.spec.app)

如果 tserver.subType==tars, 那么 ports 会被 映射为 name: $(tserver.tars.servants.name), port: $(tserver.tars.servants.port) ,
protocol: TCP|UDP

## TServer与StatefulSet的映射

当 tserver.k8s.daemonset==false 为时, controller 从一个 tserver 对象, 映射出同名 statefulset 对象.

如果 tserver.spec.subType==tars, 目标 statefulset的 pod.spec 中会固定拥有一个名字 "tarsnode" 的 initContainer 与 一个与 tserver 同名的
container 其中 initContainers.image = tserver.spec.releaset.nodeImage, container.image=tserver.spec.release.image

---

如果 tserver.spec.subType==normal, 目标 statefulset的 pod template 中有一个与 tserver 同名的 container 且
container.image=tserver.spec.release.image

下文逐一介绍这些字段 tserver.spec.k8s 的域值与statefulset,initContainers,container 的映射关系

### tserver.k8s.affinity

tserver.k8s.affinity 字段描述的是 tars 服务的节点亲和性功能.

在一些部署场景中,我们希望 tars 服务运行在固定范围的节点上, 这个希望有可能是''必须的",也可能是"建议"性的.基于此, tarsk8s提供了四个选项

+ AppRequired

  运行该服务的节点必须有 tars.io/ability.$(tserver.namespace).$(tserver.spec.app) 标签

  statefulset.spec.template.spec.affinity 会被 映射为:
  ```yaml
  affinity:
      nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
                  - key: tars.io/node.tars
                    operator: Exists
                  - key: tars.io/ability.$(tserver.namespace).$(tserver.spec.app)
                    operator: Exists
  ```

+ ServerRequired

  运行该服务的节点必须有tars.io/ability.$(tserver.namespace).$(tserver.spec.app)-$(tserver.spec.server) 标签

  statefulset.template.spec.affinity 被映射为:

  ```yaml
  affinity:
      nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: tars.io/node.tars
                  operator: Exists
                - key: tars.io/ability.$(tserver.namespace).$(tserver.spec.app)-$(tserver.spec.server)
                  operator: Exists
  ```


+ AppOrServerPreferred

  有 tars.io/ability.$(tserver.spec.app)-$(tserver.spec.server) ,tars.io/ability.$(tserver.spec.app) 标签的节点可以优先运行该服务

  statefulset.template.spec.affinity 会被 映射为:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: tars.io/node.tars
              operator: Exists
    preferredDuringSchedulingIgnoredDuringExecution:
      - preference:
          matchExpressions:
            - key: tars.io/ability.tars.tars-tafconfig
              operator: Exists
        weight: 60
      - preference:
          matchExpressions:
            - key: tars.io/ability.tars.tars
              operator: Exists
              weight: 30
```

+ None

  运行该服务的节点没有特别的标签需求

statefulset.spec.template.spec.affinity 会被 映射为:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: tars.io/node.tars
              operator: Exists
```

注意: 无论 tserver.k8s.affinity 的选项如何. 节点可以运行该 tars 服务的前置条件是必须有 tars.io/node.$(tserver.namespace) 标签

### tserver.spec.k8s.env

tserver.spec.k8s.env 会被复制到 containers[$(tserver.name)].env

### tserver.spec.k8s.envFrom

tserver.spec.k8s.env 会被复制到 containers[$(tserver.name)].envFrom

### tserver.spec.k8s.Resources

tserver.spec.k8s.env 会被复制到 containers[$(tserver.name)].Resources

### tserver.spec.k8s.ImagePullPolicy

tserver.spec.k8s.eImagePullPolicynv 会被复制到 containers[$(tserver.name)].ImagePullPolicy

### tserver.spec.k8s.hostIPC

tserver.spec.k8s.hostIPC 会被复制到 statefulset.spec.template.spec.hostIPC

### tserver.spec.k8s.hostNetwork

tserver.spec.k8s.hostNetwork 会被复制到 statefulset.spec.template.spec.hostNetwork

### tserver.spec.k8s.serviceAccount

tserver.spec.k8s.updateStrategy 会被复制到 statefulset.spec.template.spec.ServiceAccountName

### tserver.spec.k8s.podManagementPolicy

tserver.spec.k8s.hostNetwork 会被复制到 statefulset.spec.podManagementPolicy

### tserver.spec.k8s.replicas

tserver.spec.k8s.replicas 会被复制到 statefulset.spec.replicas

### tserver.spec.k8s.updateStrategy

tserver.spec.k8s.updateStrategy 会被复制到 statefulset.spec.updateStrategy

### tserver.spec.k8s.nodeSelector

tserver.spec.k8s.nodeSelector 声明了 tars 服务的对节点除亲和性以外的其他要求,比如要求节点有 ssd, zone标签等 tserver.spec.k8s.nodeSelector 会被追加到
statefulset.spec.template.spec.affinity.requiredDuringSchedulingIgnoredDuringExecution

### tserver.spec.k8s.notStacked

tserver.spec.k8s.notStacked 声明了同一节点上是否可以运行同一 tserver 不同 pod. 如果 notStacked = true,
statefulset.spec.template.spec.affinity 会被映射为

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: tars.io/node.tars
              operator: Exists
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            tars.io/ServerApp: $(tserver.spec.app)
            tars.io/ServerName: $(tserver.spec.server)
        namespaces: $(tserver.namespace)
        topologyKey: kubernetes.io/hostname
```

### tserver.spec.k8s.hostPorts

tserver.spec.k8s.hostPorts 声明了需要在节点上公开tserver pod 的端口.

以如下的 tserver 配置为例:

```yaml
tars:
  servants:
    - name: Test1Obj
      port: 10001
    - name: Test2Obj
      port: 10002
k8s:
  hostPorts:
    - nameRef: Test1Obj
      port: 3323
```

containers[$(tserver.name)].ports 将被映射为:

```yaml
ports:
  - name: test1obj
    containerPort: 10001
    protocol: TCP
    hostPort: 3323
  - name: test2obj
    containerPort: 10002
    protocol: TCP
```

注意: 某些 k8s 集群网络插件可能并不支持或默认未开启 hostPort 功能. 您需要联系集群管理员确认信息

### tserver.spec.k8s.readinessGate

tserver.spec.k8s.readinessGate 会被追加到 statefulset.spc.template.spec.readinessGates

### tserver.spec.k8s.launcherType

在原生 tars 中, tarsnode 为业务进程提供守护服务以及生命周期管理,

tarsk8s 保留了这一个方式.,让 tarsnode 作为 pod 内的 1 号进程，负责业务进程的启动,暂停,重启,心跳监听工作. 这种进程模型被命名为 background tarsk8s 同时提供了另一种进程模型, 即业务进程作为
pod 内的 1号进程，tarsnode 仅负责心跳监听工作, 这种进程模型被命名为 foreground

### tserver.spec.k8s.mounts

tarsk8s支持常见的磁盘挂载方式,包括 [ configMap, secret, emptyDir, hostPath, persistentVolumeClaim]

您可以在 tserver.spec.k8s.mounts.source 填充一种挂载方式, 在 tserver.spec.k8s.mounts.mountPath 中填写挂载到 pod 的路径.

tserver.spec.k8s.mounts.name, tserver.spec.k8s.mounts.source 域会被 controller 复制到 statefulset.spec.template.spec.volumes

tserver.spec.k8s.mounts.source 之外的域会被 controller 复制到 containers[tserver.name].volumeMounts

以如下 teserver.k8s.mounts 为例:

```yam
  k8s:
    mounts:
    - name: host-log-dir
      mountPath: /usr/local/app/tars/app_log
      subPathExpr: $(Namespace)/$(PodName)
      source:
        hostPath:
          path: /usr/local/app/tars/app_log
          type: DirectoryOrCreate
```

会被用映射为:

```yam
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: tars-tafconfig
spec:
  template:
    metadata:
      name: tars-tafconfig
    spec:
      volumes:
      - name: host-log-dir
        hostPath:
          path: /usr/local/app/tars/app_log
          type: DirectoryOrCreate
      containers:
        volumeMounts:
        - name: host-log-dir
          mountPath: /usr/local/app/tars/app_log
          subPathExpr: $(Namespace)/$(PodName)
```

除了支持常规磁盘挂载, tarsk8s 还支持了 PersistentVolumeClaimTemplate 和 TLocalVolume 方式的磁盘挂载.

PersistentVolumeClaimTemplate 是 statefulset.spec.VolumeClaimTemplates 的别名.

以如下 tserver 为例:

```yaml
  k8s:
    mounts:
      - name: host-log-dir
        mountPath: /usr/local/app/tars/app_log
        subPathExpr: $(Namespace)/$(PodName)
        source:
          persistentVolumeClaimTemplate:
            metadata:
              annotations:
                zone: sounth-03
                disk_type: ssd
              name: remote-log-dir
              namespace: tars
            spec:
              accessModes:
                - ReadWriteOnce
              resources:
                requests:
                  storage: 1G
```

会被映射为如下 statefulset:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: tars-taflog
  namespace: tars
spec:
  template:
    spec:
      containers:
        volumeMounts:
          - mountPath: /usr/local/app/tars/remote_app_log
            name: remote-log-dir
            subPathExpr: $(PodName)
  volumeClaimTemplates:
    - apiVersion: v1
      kind: PersistentVolumeClaim
      metadata:
        name: remote-log-dir
        namespace: tars
        annotations:
          zone: sounth-03
          disk_type: ssd
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1G
```

TLocalVolume 是 tarsk8s 为了 适应私有环境 K8S 集群的磁盘挂载需求而设计出来的. 其本质是一个特化版标签以及其他挂载参数的 PersistentVolumeClaimTemplate. 以如下 tserver为例:

```
  k8s:
    mounts:
    - name: remote-log-dir
      mountPath: /usr/local/app/tars/remote_app_log
      readOnly: false
      source:
        tLocalVolume:
          gid: "0"
          mode: "755"
          uid: "0"
```

会被映射为:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: tars-taflog
spec:
  template:
    spec:
      containers:
        volumeMounts:
          - mountPath: /usr/local/app/tars/remote_app_log
            name: remote-log-dir
  volumeClaimTemplates:
    - apiVersion: v1
      kind: PersistentVolumeClaim
      metadata:
        labels:
          tars.io/LocalVolume: remote-log-dir
          tars.io/ServerApp: $(tserver.spec.app)
          tars.io/ServerName: $(tserver.spec.server)
        name: remote-log-dir
        namespace: tars
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1G
        selector:
          matchLabels:
            tars.io/LocalVolume: remote-log-dir
            tars.io/ServerApp: tars
            tars.io/ServerName: taflog
        storageClassName: t-storage-class  # TLocalVolume 的固定 StorageClassName
        volumeMode: Filesystem
```

TLocalVolume 的实现:

1. controller 将 tlocalvolume 映射到 statefulset.spec.volumeClaimTemplates
2. statefulset 控制器根据 statefulset.spec.replicas , volumeClaimTemplates 新建 persistentVolumeClaims
3. 在每个tarsk8s 可用节点上会运行 tars-agen 有服务，该服务监听到 persistentVolumeClaims 资源, 且自身满足分配 K8S LocalPV 的条件(有
   tars.io/SupportLocalVolume 标签), 会新建与 persistentVolumeClaim 匹配的 localPV 资源
4. k8s 调度器自动绑定persistentVolumeClaim 和 localPV, 进而将 localPV 分配给 tserver 关联 Pod

注意: 只有 spec.k8s.daemonset 的 tserver 才可以使用 PersistentVolumeClaimTemplate ,TLocalVolume

## TServer与DaemonSet的映射

如果 tserver.spec.k8s.daemonset==true . controller 从一个 tserver 对象衍生出同名 daemonset 对象. tserver 到 daemonset 的映射规则与 tserver 到
daemonset 的映射规则基本一致. 除了以下两种情况:

1. tserver.k8s.affinity, tserver.k8s.nodeSelector, tserver.k8s.notStacked 不再生效
2. 无法使用 persistentVolumeClaimTemplat , tLocalVolume

### 其他特性

#### 镜像分类

**TarsCloud K8SFramework**  中涉及到了很多种镜像, 下文为您整体介绍这些镜像的种类

+ tarsnode 镜像

  每个最终运行的 tars 服务都是由 tarsnode 进程启动和守护的, 换句话说

  在最终启动的 tars 服务容器中, 都是先启动 tarsnode 进程, 随后由  tarsnode 进程启动和管理 tars 业务服务进程.

  这种启动方式在客观上要求  **TarsCloud K8SFramework**   项目提供一种单独 tarsnode 镜像,

  然后由控制器将 tarsnode 程序从 tarsnode 镜像中提取出来,与tars 业务服务镜像 合并到一起, 生成最终的 tars 服务容器

  > 尽管 "由 tarsnode 启动以及守护业务服务" 给人的第一印象是 "不够云原生 " , 但仍然有足够的理由这么做:
  >
  > 1. 可以最大程度的兼容原生 Tars框架的 使用习惯，包括 暂停/重启服务,发送命令等操作
  >
  > 2. 由业务程序作为前台进程在容器内运行时可行的, 但 "程序运行崩溃, 容器随即就会被删除" 会给故障处理带来挑战.
  > 3. tarsnode 能监听 业务进程的心跳状态, 避免需要在 Kubernetes 层面上配置非常多的 存活探针
  >
  > 针对 "不够云原生" 的指责,我们提供了 "FrontGround" , "BackGround" 两种启动方式:
  >
  >     BackGround:  tarsnode 做为容器内前台(1号)进程,业务进程被 tarsnode 管理和守护,完全兼容 原生 Tars 框架的管理方式(重启/暂停服务 ，发送命令操作)
  >             
  >     FrontGround:  业务进程作为容器内前台(1号)进程, tarsnode 随后启动作为心跳监听进程, 此时,只能兼容原生Tars框架的发送命令操作



+ base 镜像

  每个业务服务运行都离不开一个基础运行环境, 我们将这个基础运行环境固化到单独的镜像中, 称之为 base 镜像

  基于某个 base 构建您的 业务服务镜像, 可以降低构造业务服务镜像的难度,也可以提供资源复用率.

  **TarsCloud K8SFramework**   项目内置了一些 base 镜像 ,您可以根据需要定制您自己的 base 镜像



+ server 镜像

  每个业务服务的每个版本都是一个独立的镜像, 我们称之为 server 镜像.

  您可以自行构建 server 镜像或者 通过 tarsweb 上传发布包的方式构建出服务镜像.



+ executor 镜像

  **TarsCloud K8SFramework**   项目内置了一个镜像编译服务, 最终的镜像编译工作是由  executor 镜像的的 tarskaniko 程序执行的.



#### 镜像构建

我们只建议您 自行构建 base 镜像和 server 镜像, 下文只阐述 base 镜像与 server 镜像的构建方法



##### Base 镜像构建

必要性:  如果内置的 base 镜像不满足的程序运行时需求,或者您需要在大量服务镜像中内置某个基础组件时可添加基础镜像

操作步骤:

1. 确定基础镜像能支持 Tars 服务类型

   > 当前 TarsCloud K8SFramework 支持了 四种 Tars 服务类型 分别为:  cpp/go/nodejs/(java-jar,java-war)
   >
   > 您需要明确目标基础镜像可以支持的  Tars 服务类型, 为此种类型安装合适的运行环境,比如针对 nodejs ,你需要在镜像中包含 node程序和基础  nodejs 库

2.  生成 Dockerfile 文件，已经构建您的镜像,提交到镜像仓库

3.  添加基础镜像到框架中

> 云原生方式添加:
>
> 新建  Yaml文件 mybase.yaml ,并按说明填充内容
>
> ```yaml
   > apiVersion: k8s.tars.io/v1beta3
   > kind: TImage
   > metadata:
   > name: mybase #按需填充您希望的对象名
   > namespace: #填充您的 Framework 安装的命名空间
   > imageType: base
   > releases:
   > - id: #请设置和填充一个 id 
   >   image: #请填充镜像地址
   >   mark: #请填充一些说明信息
   > supportedType: [] #填充您的镜像支持的 Tars 服务类型
   > ```
>
> ```she
   > 执行 kubectl apply -f mybase.yaml
   > ```
> 管理平台方式:  
> 在管理平台, 依次点击 "运维管理"->"基础镜像管理"->"添加"，并根据提示内容填入信息

##### Server 镜像构建

操作步骤:

1:  确认您的服务类型

> 当前 TarsCloud K8SFramework 支持了 四种服务类型 分别为:  cpp/go/nodejs/(java-jar,java-war)  ,您根据服务选择

2:  选择以下一种方式构建镜像

+ 通过管理平台构建

  > 在管理平台,进入目标服务界面, 依次点击 "发布管理"->"上传发布包", 然后根据提示填入内容信息

+ 自行构建

  > 1.  新建和编辑 Dockerfile, 按说明填充 内容
  > ```dockerfile
  > FROM base                          #请按需填写您的基础镜像
  > ....                               #请按需填写您的构建流程
  > RUN rm -rf /etc/localtime          #镜像内的 /etc/localtime 文件会影响 挂载宿主机的 /etc/localtime 文件
  > COPY server /usr/local/server/bin  #请按需更改您的服务程序目录名
  > ENV ServerType=cpp                 #请按需填充您的服务类型
  > ```
  >
  > 2. 构建镜像并推送到镜像仓库
  > 3. 通过管理平台将镜像提交到服务版本列表

  如果的您的服务需要第三方库, 可以在 程序同级新建 lib 目录,然后将 so 文件放置其中


#### 磁盘管理

在 Framework 程序部署和运行时,会挂载所在宿主机的 /usr/local/app/tars 目录, 并使用一些磁盘空间,

您可以单独硬盘到此目录, 或者软连接一块较大的磁盘空间到此目录.

目录使用情况如下:

+  日志占用

为了便于故障排查, 我们使用 Kubernetes HostPath 存储 业务容器的运行日志

容器内的日志存储目录为:  /usr/local/app/tars/app_log/${App}/${Server}

宿主机的日志存储目录为:  /usr/local/app/tars/app_log/${Namespace}/${PodName}/${App}/${Server}



+ 镜像缓存占用

  我们内置的镜像编译服务,为了加快编译速度,我们通过  Kubernetes HostPath  共享一些缓存数据

  宿主机存储目录为: /usr/local/app/tars/image_build



+ TLV 占用

  **TarsCloud K8SFramework** 提供了 TLV 的功能, 会在宿主机新建目录作为  Kubernetes LocalPV 分配给使用者

  宿主机存储账号目录为:  /usr/local/app/tars/host-mount

#### 时区管理

Controller 在映射 Tars 服务的 Pod时，总是会尝试将 宿主机的 /etc/localtime 挂载到 Pod内, 大多数情况下，这种方式可以保证 Pod与宿主机使用的是同一时区

但是我们发现在某些特殊场景无效，我们正才探索更好的方法.
