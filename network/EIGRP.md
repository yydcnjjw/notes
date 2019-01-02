[TOC]

# EIGRP
EIGRP(Enhanced Interior Gateway Routing Protocol)增强内部网关协议 EIGRP 是 Cisco 公司开发的高级的距离矢量路由协议,支持 IP、IPX 等多种网络层协议。IP协议号为88。
EIGRP是一个平衡混合型路由协议(Cisco公司创造的术语)

- 有传统的距离矢量协议的特点:
    - 路由信息依靠邻居路由器通告
    - 遵守路由水平分割和毒性逆转规则
    - 路由自动汇总
    - 配置简单
- 有传统的链路状态路由协议的特点:
    - 没有路由跳数的限制
    - 当路由信息发生变化时,采用增量更新的方式,保留对所有可能路由(网络的拓扑结构)的了解
    - 支持变长子网掩码
    - 路由手动汇总
- 该协议同时又具有自己独特的特点:
    - 支持非等成本路由上的负载均衡
    - 采用扩散更新算法(DUAL-Diffusing Update Algorithm,思科开发)在确保100%无路由环路的前提下,收敛迅速。因而适用于中大型网络。

## EIGRP协议的特点
- 快速收敛:EIGRP采用DUAL来实现快速收敛。运行EIGRP的路由器存储了邻居的路由表,因此能够快速适应网络中的变化。如果本地路由表中没用合适的路由且拓扑表中没用合适的备用路由,EIGRP将查询邻居以发现替代路由。查询将不断传播,直到找到替代路由或确定不存在替代路由。
- 部分更新(增量更新):EIGRP发送部分更新而不是周期更新,且仅在路由路径或者度量值发生变化时才发送。更新中只包含已变化的链路的信息,而不是整个路由表,可以减少带宽的占用。此外,还自动限制这些部分更新的传播,只将其传递给需要的路由器,因此EIGRP消耗的带宽比IGRP少很多。这种行为也不同于链路状态路由协议,后者将更新发送给区域内的所有路由器。
- 支持多种网络层协议:EIGRP使用协议无关模块来支持IPv4、IPv6、Apple Talk和IPX,以满足特定网络层需求。
- 使用组播和单播:EIGRP在路由器之间通信时使用组播和单播而不是广播,因此终端站不受路由更新和查询的影响。EIGRP使用的组播地址是224.0.0.10
- 支持变长子网掩码(VLSM):EIGRP是一种无类路由协议,它将通告每个目标网络的子网掩码,支持不连续子网和VLSM。
- 无缝连接数据链路层协议和拓扑结构

## EIGRP协议的4个组件
![](image/_1528974249_1939822641.png)

- 依赖于协议的模块:支持IPX/IP/Apple Talk
- 可靠传输协议(RTP-Reliable Transport Protocol):
    - RTP来管理EIGRP报文的发送和接收,发送的报文是有保障的而且报文是有序发送的。
    - DUAL算法提供了报文发送的可靠性。每一个接收可靠组播报文的邻居都会发送一个单播的确认报文。
- 邻居的发现和恢复模块
- 扩散更新算法(DUAL)

> 注意:
> EIGRP有跳数之说,默认为100,最大为255。
> 可以通过如下命令修改:
> `Router(config-router)#metric maximum-hops 255`

## AD
- 内部:90
- 外部:170
- 系统汇总:5

## 原理
### 包格式
![](image/_1528974356_683586918.png)

- Version(8bit):版本,目前皆为版本2
- OPcode(8bit):操作码
    |Opcode|type|
    |---|---|
    |1|更新(Update)|
    |3|查询(Query)|
    |4|更新(Reply)|
    |5|问候(Hello)|
    |6|IPX SAP|
- Checksum(16bit):校验和,整个EIGRP来计算
- Flags(32bit):标记
    - init位:0x00000001(新建立的邻居关系的开始)
    - 有条件接收位(conditional receive):0x00000002(使用DUAL)。
- Sequence(32bit):序列号,用于RTP中
- ACK(32bit):确认序列号,单播确认
- 自治系统号(Autonomous System Number,32bit):EIGRP协议于的标识号
- TLV(32bit):Type/Length/Value
    ![](image/_1528974518_12407444.png)
    - 一般TLV(General TLV):包含复合度量值(K值)和Holdtime(这个时间是给邻居看的)时间。
        ![](image/_1528974546_456114395.png)
    - IP特有TLV(IP-Specific TLV):
        - The IP Internal Routes TLV
            ![](image/_1528974599_1425939891.png)
        - The IP External Routes TLV

#### 报文类型
- Hello(5):用于邻居的发现和恢复。使用组播的方式发送,并且是不可靠的。
    ![](image/_1528974661_1902003345.png)
- Update(1):用于传递路由更新信息。仅包含需要的路由条目。当为某一个指定的路由器发送路由更新时,使用单播。当为多台路由器发送路由更新时,使用组播。
    **更新报文是可靠的发送方式**
    ![](image/_1528974692_1554531242.png)
- Query(3):是DUAL有限状态机用来管理其扩散计算的。可以使用组播或单播发送。当找不到Feasible Successor时,发送组播的查询报文。采用可靠的发送方式。
    **注意**:若某台路由器发现路由发生变化时,先发Update,再发Query。
    ![](image/_1528974742_467257260.png)
- Reply(4):是DUAL有限状态机用来管理其扩散计算的。用于单播的形式来回应查询报文。采用可靠的发送方式。
    **注意**:若某台路由器收到Query后,会先发ACK,再发Reply。
    ![](image/_1528974774_957693343.png)
- 使用单播的方式来确认Update/Query/Reply报文的。
    **采用不可靠的发送方式**
    ![](image/_1528974812_1370148546.png)
    - SIA-QUERY:用于避免SIA超时导致邻居关系重置
    - SIA-REPLY:用于避免SIA超时导致邻居关系重置
    > 注:在路由丢失并且FS没有的前提下,quary和reply两种报文才会出现!!

### 三张表
#### EIGRP Neighbor Table
确保运行EIGRP的直连邻居可以互相通讯
![](image/_1528974910_1520457854.png)

#### EIGRP Topology Table
保存学习来自所有EIGRP邻居关于目的路由的条目
![](image/_1528974945_103999460.png)

#### EIGRP IP Routing Table
选取拓扑表中抵达最终目的地的最优路径放入路由表中
![](image/_1528974966_986362605.png)

#### 总结
![](image/_1528974992_1623802209.png)

### 邻居的建立
#### 邻居建立条件
- 必须是相同的Autonomous System Number
    ```
    Router(config)#router eigrp ?
        <1-65535> Autonomous system number
    ```
- K值必须相同
    ```
    R1#show ip protocols
        Routing Protocol is "eigrp 90"
        Outgoing update filter list for all interfaces is not set
        Incoming update filter list for all interfaces is not set
        Default networks flagged in outgoing updates
        Default networks accepted from incoming updates
        EIGRP metric weight K1=1, K2=0, K3=1, K4=0, K5=0
        EIGRP maximum hopcount 100
        EIGRP maximum metric variance 1
        Redistributing: eigrp 90
        EIGRP NSF-aware route hold timer is 240s
        Automatic network summarization is not in effect5
        Maximum path: 4
        Routing for Networks:
        1.1.1.0/24
        12.1.1.0/24
        Routing Information Sources:
        Gateway
         Distance
         Last Update
        Distance: internal 90 external 170
    ```
- 同处一条链路上的两台设备直连接口不属于相同网段,也建立不了邻居关系。

> K值
> - K1=bandwidth(带宽:源目间的最小带宽--默认为 1)
> - K2=loading(负载:源目间的最大负载--默认为 0)
> - K3=delay(延迟:源目间的延迟总和--默认为 1)
> - K4=reliability(可靠性:源目间的最低可靠性--默认为 0)
> - K5=MTU(最大传输单:源目间的最小MTU--默认为 0)

修改K值的命令:(当然,不建议修改)
   `Router(config-router)#metric weights 0 1 0 1 0 0`
   此处的“0”表示为ToS,而且必须为0!

### 邻居建立过程
1. 步骤一：
    ![](image/_1528975172_888886946.png)
2. 步骤二：
    ![](image/_1528975202_651026818.png)
3. 步骤三：
    ![](image/_1528975218_145352194.png)
4. 步骤四：
    ![](image/_1528975245_83867473.png)
5. 步骤五：
    ![](image/_1528975265_990947259.png)

#### 总图
![](image/_1528975291_1066239025.png)

#### 关于goodbye消息
![](image/_1528975322_1532630191.png)

- goodbye消息是以Hello分组的方式发送的
- 在goodbye消息中,所有K值都被设置为255
    ![](image/_1528975334_251396036.png)
- 在使用命令`no router eigrp xx/no network`、或重启eigrp进程的时候会发送,如果是使用`no network`,则仅在该接口下发送。

### 度量值
#### 计算EIGRP Metric的5个参数
- 带宽(bandwidth)
- 延迟(delay)
- 可靠性(reliability):根据keepalive而定的源和目的之间最不可靠的可靠度的值
- 负载(loading):根据包速率和接口配置带宽而定的源和目的之间最不差的负载的值
- 最大传输单元(MTU):路径中最小的MTU,MTU包含在EIGRP的路由更新里,但是一般不参与EIGRP度量的运算

#### 计算公式
```
Metric = (10,000,000/BW + (DeLay sum)/10 )* 256
```

- 10的7次方除以源和目标之间最低的带宽(10的7次方除以以Kbit/s为单位的最小带宽),加上延迟之和除以10,最后乘于256
- 如果修改K值,使K5不等于0,则 Metric 计算式变成:256*[K1(10^7/带宽)+K2(10^7/带宽)/(256-负载)+K3(延迟)]*[K5 / (可靠性+K4)]
计算出的Metric值不是整数时自动取整,比如计算结果为8501.39 ,显示值将为8501。

> 友情提醒:
> 路由的入接口 = 数据包出接口(output interface)
>
> - BW:数据包去往目的地的方向上,所有路由器出接口中的最小带宽
>     - 路由条目:流向本路由器方向上,所有路由器入口中的最小带宽。
>     - 单位:Kbps
> - 数据包去往目的地的方向上,所有路由器出接口的延时之和。
>     - 路由条目:流向本路由器方向上,所有路由器入口的延时之和
>     - 单位:us(取路由来的方向的入接口的总和)

![](image/_1528976025_1515115497.png)

### DUAL算法
#### 应用DUAL算法的条件
- 一个节点需要在有限的时间内检测到一个新的邻居的存在或一个相连邻居的丢失。
- 在一个正在运行的链路上传递的所有消息应该在一个有限的时间内正确的收到,并且包含正确的序列号。
- 所有的消息,包括改变链路的开销、链路失效和发现新邻居的通告,都应该在一个有限的时间内一次一个地处理,并且应该被有序的检测到。
> EIGRP协议使用邻居发现/恢复和RTP来确定这些条件!

#### 术语
- 邻接(Adjacency):设备刚启动时,路由器使用hello报文发现它的邻居和标识自己并让邻居识别。当邻居被发现后,EIGRP协议将试图和它的邻居形成一个邻接的关系。邻接是指两个相互交换路由信息的邻居之间形成的一条虚链路。一旦邻接关系形成,路由器就可以从他们的邻居接收路由更新信息了。路由更新信息包含发送路由器所知道的路由和这些路由的度量值。对于每一条路由,路由器都将会基于它的邻居通告的距离(AD)和到它的邻居的链路开销计算出抵达最终目的地的一个距离(FD)。
- 可行距离(Feasible distance):抵达每一个目的地的最小度量将作为该路由器到最终目的网络的可行距离。
    (最终在路由表中体现)
- 通告距离(Advertise distance):邻居设备抵达最终目的网络的开销。
- 可行性条件(Feasibility condition):本地路由器的一个邻居路由器所通告的到达一个目的网络的距离(AD)是否小于本地路由器到达相同目的网络的可行距离(FD)
    **注意**:满足“FC=AD<FD”这个条件的路由条目将会放入到EIGRP的拓扑表中。
- 后继路由器(Successor):对于在拓扑表中列出的每一个目的网络,将选用拥有最小度量值的路由条目并放入路由表中,那么通告这条路由的邻居就成为一个后继路由器(或者是到达目的网络数据包的下一跳路由器)。
- 可行后继路由器(Feasible successor):满足FC条件中AD的那个路由器就成为去往目的网络的一个可行后继路由器。

## 功能
### 负载
#### 等价负载均衡
EIGRP默认支持4条等价负载均衡链路,最多支持16条负载均衡链路。(看IOS,15.x支持32条)
```
R7200(config-router)#maximum-paths ?
<1-32> Number of paths
Router#sh ip protocols
    Routing Protocol is "eigrp 90"
    Outgoing update filter list for all interfaces is not set
    Incoming update filter list for all interfaces is not set
    Default networks flagged in outgoing updates
    Default networks accepted from incoming updates
    EIGRP metric weight K1=1, K2=0, K3=1, K4=0, K5=0
    EIGRP maximum hopcount 100
    EIGRP maximum metric variance 1
    Redistributing: eigrp 90
    EIGRP NSF-aware route hold timer is 240s
    Automatic network summarization is not in effect
    Maximum path: 4
    Routing for Networks:
    1.1.1.0/24
    12.1.1.0/24
    Passive Interface(s):
    FastEthernet0/0
    Routing Information Sources:
    Gateway
     Distance
     Last Update
    Distance: internal 90 external 170
```

##### 通过命令修改最大负载
```
Router(config-router)#maximum-paths ?
    <1-16> Number of paths
```

##### 配置等价负载均衡
- 修改接口的带宽和延迟来达到改动路由Metric值的目的,从而实现等价路由
    - 如果修改Metric则需要在路由流向入向修改
    - 如果修改延时
        ```
        Router(config-if)#delay ?
            <1-16777215> Throughput delay (tens of microseconds)
        ```
- 通过使用偏移列表来改动Metric值,实现等价负载均衡
    ```
    access-list 1 permit 172.16.1.0 0.0.0.255
    Router(config-router)#offset-list 1 in xxxx fastethernet 0/0
    ```
    在原有Metric的基础上增加xxxx

#### 非等价负载均衡
默认只支持等价的负载均衡
如下图,若想完成非等价的负载均衡,使用参数“variance”(变量)
```
Router(config-router)#variance ?
    <1-128> Metric variance multiplier
```

满足条件:

- Router E chooses router C to get to network Z, because it has lowest FD of 20。
- With a variance of 2, router E chooses router B to get to network Z (20 + 10 = 30) < [2 * (FD) = 40]---必须小于不能等于
路由通过FS设备的FD<variance's value *FD
- Router D is never considered to get to network Z (because 25 > 20).

![](image/_1528976415_1863692482.png)

##### 相关命令
```
Router(config-router)#traffic-share ?
    balanced Share inversely proportional to metric
    min      All traffic shared among min metric paths
```
> “balanced”:默认配置,路由器在参与负载的链路上对传输流量的分配跟Metric成反比

```
Router(config-router)#traffic-share min ?
    across-interfaces Use different interfaces for equal-cost paths
```
路由器在参与负载的链路上对传输流量的分配按照最小开销

### 被动接口
```
Router(config-router)#passive-interface FastEther 0/1
```
**处于passive状态的接口将“不收发”Hello包**。
建立不起EIGRP的邻居关系!!
```
Router(config-router)#neighbor 12.1.1.2 fastEthernet 0/0---必须跟上接口
```
就算是单播指邻居了,EIGRP的邻居关系也建立不起来。
**EIGRP的hello包,要么都是组播,要么都是单播,发送方式不一致不行**。

### 汇总
默认情况下,EIGRP在穿越不同的网络边界时将按照主类网络对路由进行汇总,并且在路由表中增加一条指向null0的汇总路由(为了防环,让一个没有明细路由条目的数据包可以快速丢弃,并且不用查找完整个路由表)。
**注意**:仅汇总本地路由,对收到的路由不进行汇总。

![](image/_1528976601_1572073631.png)

```
Router#sh ip route 1.1.0.0
    Routing entry for 1.0.0.0/8
    Known via "eigrp 90", distance 5, metric 128256, type internal
    Redistributing via eigrp 90
    Routing Descriptor Blocks:
    * directly connected, via Null0
        Route metric is 128256, traffic share count is 1
        Total delay is 5000 microseconds, minimum bandwidth is 10000000 Kbit
        Reliability 255/255, minimum MTU 1514 bytes
        Loading 1/255, Hops 0
```

#### 手动汇总
一般情况下要关闭自动汇总(`no auto-summary`),这样做是为了防止丢包(可能会造成路由黑洞或下一跳不明确)。
```
Router(config-if)#ip summary-address eigrp 90 1.1.0.0 255.255.0.0 ?
    <1-255>   Administrative distance ---修改默认的管理距离
    leak-map  Allow dynamic prefixes based on the leak-map ---做路由泄露,除发汇总外,也发匹配的明细路由
```
依然会产生一条去往null0的汇总路由:
```
1.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
D  1.1.0.0/16 is a summary, 00:00:02, Null0
```

##### 手动汇总的原则
- 本地必须有明细路由,才会从做汇总的接口发出汇总路由。
- 直到明细的最后一条路由消失,汇总才会消失。
- 汇总路由的metric值会取最小的metric值 (接收方)。
**注意**:仅汇总本地路由,对收到的路由不进行汇总。不能对外部路由进行汇总。
```
interface Ethernet1/0
    ip address 12.1.1.1 255.255.255.0
    ip summary-address eigrp 90 1.1.0.0 255.255.0.0 5 ---应该是介于5和6之间
```

### 下方路由
- 在EIGRP的AS中通告:
    ```
    Router(config)#ip route 0.0.0.0 0.0.0.0 FastEther 0/1(只能写出接口)
    Router(config)#router eigrp 90
    Router(config-router)#network 0.0.0.0
    ```
- 重分布静态:
    ```
    Router(config)#ip route 0.0.0.0 0.0.0.0 FastEther 0/1(随便写)
    Router(config)#router eigrp 90
    Router(config-router)#redistribute static metric 1000 100 255 1 1500
    ```
- 接口下汇总:
    ```
    Router(config-if)# ip summary-address eigrp 90 0.0.0.0 0.0.0.0
    ```
**注意**:
```
Router(config-router)#no default-information ?
    allowed Allow default information
    in      Accept default routing information
    out     Output default routing information
```

### 末节(Stub)区域
一般把直接接入终端设备网络的路由器配置成Stub路由器。

- 当某台路由器配置为stub路由器后,该路由器会向所有邻居发送特殊信息,邻居设备将不会向其发送查询。
- 某一台路由器配置成了stub路由器,则该路由器默认只会通告它的直连路由和本地所做的汇总路由。
```
Router(config)#router eigrp 90
    Router(config-router)#eigrp stub
    Router#show running-config | s router eig
    router eigrp 90
        network 12.1.1.0 0.0.0.255
        no auto-summary
        eigrp stub connected summary
```

#### 配置
```
Router(config-router)#eigrp stub ?
    connected     Do advertise connected routes
    leak-map      Allow dynamic prefixes based on the leak-map
    receive-only  Set IP-EIGRP as receive only neighbor
    redistributed Do advertise redistributed routes
    static        Do advertise static routes
    summary       Do advertise summary routes
```

### 水平分割
EIGRP默认开启水平分割
```
R2(config-if)#no ip split-horizon eigrp 90 ---关闭EIGRP的水平分割
R2(config-if)#no ip split-horizon ---关闭RIP的水平分割
```

### 路由认证
动态路由协议支持的路由认证

- 只支持明文认证:IS-IS
- 只支持MD5认证:EIGRP/BGP
- 俩都支持:RIPv2/OSPF
**EIGRP只支持密文(MD5)的认证**

#### 配置
```
Router(config)#key chain QYT
Router(config-keychain)#key 1
Router(config-keychain-key)#key-string cisco
Router(config-if)#ip authentication key-chain eigrp 90 QYT
Router(config-if)#ip authentication mode eigrp 90 md5
```

#### 认证失败现象
![](image/_1528977038_1403521454.png)

## 实施存在的问题和解决方案
### EIGRP 的计时器
- Hello Timer:大于等于T1线路,5s(组播);小于T1线路,60s(单播)。
    帧中继的主接口和多点子接口为60秒,HDLC/PPP/以太网链路/帧中继点到点子接口为5秒。
    ![](image/_1528977130_197501674.png)
- Hold Timer(保持时间):3倍的Hello时间
- SRTT(Smooth Round-Trip Time,平均往返时间):从发送EIGRP的3种可靠报文,到对方回应ACK的时间,单位是毫秒。
- RTO(Retransmission TimeOut,重传超时):重传超时计时器(Queue count,队列数,还在排队等候发送的报文数)
- 组播流定时器(multicast flow):指定了从组播切换到单播之前,等待ACK分组的时间。
- Active time:仅为SIA服务,默认3分钟。

> 注意
>
> - 如果Hello时间和保持时间不一致的情况下依然不影响EIGRP邻居的建立,但可能会出现问题。两个时间的修改,互相不影响。
> - 针对EIGRP的3种可靠报文的最大重传次数为16次。如果16次还没有收到ACK,则重置邻居关系。一般情况下,16次超时的时间会持续50到80秒。
> - RTO和组播流定时器是通过SRTT推算出来的。

#### 相关配置
可以通过命令来修改Hello和Hold Timer的时间:
```
Router(config-if)#ip hello-interval eigrp AS yy (1-65535)
Router(config-if)#ip hold-time eigrp AS yy (1-65535)
```
修改Active time时间:
```
Router(config-router)#timers active-time ?
    <1-65535> EIGRP active-state time limit in minutes
    disabled  disable EIGRP time limit for active state
```

### SIA(stuck in active)
当本地路由器的一条路由变为活跃状态并且向它的邻居路由器发送查询报文的时候,在本地路由器收到每一个查询报文的答复报文之前,这条路由将一直保持活跃状态。但是如果一个邻居失效了或其他无效的情况下没有办法做出答复,那么,这条路由将永久的停留在活跃状态。

#### 活动计时器(默认3分钟)
当发送一个查询报文时,活动计时器就被设置了,如果在收到查询报文的答复报文之前,活动计时器超时了,这条路由就被宣告“卡”在活动状态。那么这个邻居也就被推断为失效了,并从邻居表中删除。SIA路由和任何其他经过这个邻居的路由也都会从路由选择表中删除。DUAL算法将会认为这个邻居已经答复了一个含有无穷大度量的报文。

值得注意的是,查询将会蔓延到网络的整个边界,如果扩散计算的直径增大到足够大的时候,活动计时器将可能在收到所有的答复前超时,结果会从邻居表中刷新掉一个合法的邻居,这样将造成网络的不稳定。

![](image/_1528977292_1093780308.png)

#### 造成SIA的原因
- 路由器因为系统资源的问题回复Reply不及时
- 网络的链路存在问题,可能会造成Query/Reply报文丢失
- Query报文传播直径过大

#### 解决方案
- 思科针对于SIA新开发了两种报文(SIA-QUERY/SIA-REPLY)---IOS 12.2以后
    ![](image/_1528977329_1100936448.png)
- 手工汇总:路由表中有明细路由时,才进行再次查询,否则回复网络不可达。
- stup区域
