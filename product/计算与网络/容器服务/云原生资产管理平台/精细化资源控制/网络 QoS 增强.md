网络 QoS 增强能力提供了一系列功能，保证业务网络方面的服务质量保证。全方位提升网络表现，以及灵活限制容器对网络的使用。

## 功能一：出入方向限速

### 功能介绍

限制某个容器的入、出带宽。

### 使用方式

1. 部署 [QoS Agent](https://cloud.tencent.com/document/product/457/79774)。
2. 在集群里的“扩展组件”页面，找到部署成功的 QoS Agent，单击右侧的**更新配置**。
3. 在修改 QoS Agent 的组件配置页面，勾选**网络 QoS 增强**。
4. 单击**完成**。
5. 部署业务。
6. 部署关联该业务的 PodQOS 对象，选择需要作用的业务，示例如下：
```yaml
apiVersion: ensurance.crane.io/v1alpha1
kind: PodQOS
metadata:
  name: a
spec:
  labelselector:
    matchLabels:
      k8s-app: a	# 选择需要降低优先级的业务的 Label
  resourceQOS:
     netIOQOS:
       netIOLimits:
         rxBps: 50	# rxBps 代表入方向的最大带宽，单位为 Mbps，0表示无限制，最大可以输入13位的整数
         txBps: 50	# txBps 代表出方向的最大带宽，单位为 Mbps，0表示无限制，最大可以输入13位的整数
```



## 功能二：优先级（带宽绝对抢占）

### 功能介绍

用户为不同的容器设置不同的优先级，按照优先级分配网口的带宽资源。以三档优先级为例说明网络入带宽的调度策略，可扩展到更多档优先级：

- 最高优先级（优先级为0）的容器不受网络 QoS 限制，可任意使用带宽资源。
- 所有较低优先级（优先级为1、2）容器使用的带宽之和小于 rx_bps_max 减最高优先级带宽时，每个容器可以使用的带宽不受限制，并都可以超过 rx_bps_min。
- 所有较低优先级（优先级为1、2）容器使用的带宽之和大于 rx_bps_max 时，最多可以使用 rx_bps_max 减较高优先级容器的总流量的差值，如果差值小于 rx_bps_min，则保障低优先级容器至少可使用 rx_bps_min 的带宽。
- 最高优先级（优先级为0）容器的带宽可以突破 rx_bps_max/tx_bps_max，较低优先级容器不能突破此参数。

### 用户场景

#### 场景一

- 高优先级容器使用带宽较低时，将空闲带宽分配给低优先级容器。
- 高优先级容器使用带宽较高，超过 rx_bw_max 时，只给低优先级容器分配最低保障带宽。
![img](https://qcloudimg.tencent-cloud.cn/raw/6a11a7f08075001bbd2d30c6444e10b2.png)        

#### 场景二

- 高优先级容器使用带宽较低时，将空闲带宽分配给低优先级容器。
- 新增一个中间优先级的容器时，抢占相对较低优先级容器的带宽。
![img](https://qcloudimg.tencent-cloud.cn/raw/fee209c64f003997a33252668d85211e.png)        

#### 场景三

- 高优先级容器使用带宽较低时，将空闲带宽分配给低优先级容器。
- 新增一个中间优先级的容器时，抢占相对较低优先级容器的带宽。最多能抢占的带宽要减去较高优先级容器所用带宽，还要减去较低优先级容器最低保障带宽。
![img](https://qcloudimg.tencent-cloud.cn/raw/d9fff49d11746f870bf2aaf3ab10fafc.png)        

  

### 使用方式

1. 部署 [QoS Agent](https://cloud.tencent.com/document/product/457/79774)。
2. 在集群里的“扩展组件”页面，找到部署成功的 QoS Agent，单击右侧的**更新配置**。
3. 在修改 QoS Agent 的组件配置页面，勾选 **网络 QoS 增强**。
4. 单击**完成**。
5. 部署业务 A。
6. 部署关联该业务的 PodQOS 对象，选择需要作用的业务，示例如下：
```yaml
apiVersion: ensurance.crane.io/v1alpha1
kind: PodQOS
metadata:
  name: a
spec:
  labelselector:
    matchLabels:
      k8s-app: a	# 选择业务的 Label
  resourceQOS:
    netIOQOS:
      netIOPriority: 6	# 选择网络优先级，0-7，总共8级
```
7. 部署业务 B。
8. 部署关联该业务的 PodQOS 对象，选择需要作用的业务，示例如下：
```yaml
apiVersion: ensurance.crane.io/v1alpha1
kind: PodQOS
metadata:
  name: b
spec:
  labelselector:
    matchLabels:
      k8s-app: b	# 选择业务的 Label
  resourceQOS:
    netIOQOS:
      netIOPriority: 7	# 选择网络优先级，0-7，总共8级
```

该功能需要搭配 NodeQOS 中的节点带宽限制联合使用，配合 NodeQOS 指定 rxBpsMin，rxBpsMax，txBpsMin，txBpsMax。

- rxBpsMin：per-priority 级别的入方向最低保障带宽。为该级别的容器所共享。目前不同优先级的入方向最低保障带宽相同。优先级0除外。
- txBpsMin：per-priority 级别的出方向最低保障带宽。为该级别的容器所共享。目前不同优先级的出方向最低保障带宽相同。优先级0除外。
- rxBpsMax：网口的最大入带宽。
- txBpsMax：网口的最大出带宽。

>! 单位为Mbps。
>
```yaml
apiVersion: ensurance.crane.io/v1alpha1
kind: NodeQOS
metadata:
  name: total
spec:
  netLimits:
    rxBpsMin: 10
    rxBpsMax: 10240
    txBpsMin: 10
    txBpsMax: 10240
```







## 功能三：端口白名单

### 功能介绍

为了防止低优先级容器被饿死，有两种机制：

1. per-priority 的最低保障带宽：由该优先级的容器所共享。
2. 端口白名单机制：用户可以将容器中的某个本地端口、或对端端口添加到白名单，这个端口的流量将不被限流。通常用于保护特定协议的控制报文。

### 使用方式

1. 部署 [QoS Agent](https://cloud.tencent.com/document/product/457/79774)。
2. 在集群里的“扩展组件”页面，找到部署成功的 QoS Agent，单击右侧的**更新配置**。
3. 在修改 QoS Agent 的组件配置页面，勾选 **网络 QoS 增强**。
4. 单击**完成**。
5. 部署业务。
6. 部署关联该业务的 PodQOS 对象，选择需要作用的业务，示例如下：
```yaml
apiVersion: ensurance.crane.io/v1alpha1
kind: PodQOS
metadata:
  name: a
spec:
  labelselector:
    matchLabels:
      k8s-app: a	# 选择业务的 Label
  resourceQOS:
     netIOQOS:
       whitelistPorts: 	# 代表业务a的本地端口 5201 和远端端口 5205 被设置为白名单端口，不做限流。lports,rports 默认值都为0，取值范围为0 ~ 13位整数
         lports: 5201
         rports: 5205
```





