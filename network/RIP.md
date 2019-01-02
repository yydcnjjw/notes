[TOC]

# RIP
## RIP概述
Route Information Protocol(路由信息协议)。RIP 是 Xerox 在 20 世纪 70 年代开发的,最初定义在 RFC1058(RIPv1) 中。使用 bellman-ford(贝尔曼-福特)算法。

## 使用的协议
RIP 使用 UDP 协议,源目端口皆为 520。
![](image/_1528887749_1644716281.png)

## RIP 的版本
- RIPv1 和 v2 的共同特点
    - 都是距离矢量路由协议
    - 使用跳数(hop count)作为度量值
    - 默认路由更新周期为30秒
    - 管理距离(AD)为120
        修改命令：`Router(config-router)#distance X(注意:只影响本地)`
    - 支持触发更新
    - 最大跳数为15跳(16为不可达)
    - 支持等价负载均衡(LB),默认为4条,最大16条
        修改命令:`Router(config-router)#maximum-paths X(跟IOS有关,15.x后支持到32条)`
    - 使用UDP 520端口进行路由更新
- RIPv1 和 v2 的区别
    |RIPv1|RIPv2|
    |---|---|
    |在路由更新的过程中不携带子网信息|在路由更新的过程中携带子网信息|
    |不提供认证|提供明文和 MD5 认证|
    |不支持 VLSM 和 CIDR|支持 VLSM 和 CIDR|
    |采用广播更新|采用组播(224.0.0.9)更新|
    |有类别(Classful)路由协议|无类别(Classful)路由协议|
    ![](image/_1528888545_921360520.png)

### RIPv1
#### 路由更新方式
使用广播(255.255.255.255)路由更新。

#### 包格式
![](image/_1528888630_1549043426.png)

- command(8bit):命令符,其实就是RIP的报文类型
    - request
    - response
    - traceon(追踪开启)
    - traceoff(追踪关闭)
    - reserved(保留,被SUN系统使用)
- version(8bit):版本
- address family identifier(16bit):地址族标识。IP协议,其协议簇号为2(还没有关于其他类型的地址,或许开发的当初设想是为以后做打算)
- must be zero(16bit):保留位(必须为0)
- IP address(32bit):能够通告的路由条目
    **注意**:最多一个消息可以有25个路由条目,因此消息的大小最多为4+25*20=504,再加上UDP的8个字节头,此时就有512字节。
- metric(32bit):度量值(也可以成为distance或hop)

#### 收发原则

##### 发送路由
同类发明细,异类发汇总
将要发出的路由条目和出接口的IP地址进行比较:

- 如果不是则以此路由的主类汇总网络发送
- 如果是同一个主类(A类看一个段,B类看两个段,C类看三个段),则比较接口的子网掩码
    - 如果相同则发送
    - 如果不同也可以发送,但会被置为16跳
- 32位主机路由可以发送

##### 接收路由
收到的路由条目和入接口的IP地址比较:

- 如果不是同类网络:
    - 路由表中无此路由,则按照此路由条目的主类网络处理
    - 路由表中有此路由条目的明细,则拒收。
- 如果是同类网络,和入接口的子网掩码比较,如果发现路由条目的主机位出现1,则将其作为主机路由(/32)放入路由表中。


### RIPv2
#### 路由更新方式
使用组播(224.0.0.9)路由更新
![](image/_1528888888_2056030880.png)

#### 包格式
![](image/_1528888918_427632616.png)

- command(8bit):命令符,其实就是RIP的报文类型
    - request
    - response
- version(8bit):版本
- must be zero(16bit):保留位(必须为0)
- address family identifier(16bit):地址族标识。IP协议,其协议簇号为2(还没有关于其他类型的地址,或许开发的当初设想是为以后做打算)
- Route Tag(16bit):路由标识
- IP address(32bit):能够通告的路由条目
    **注意**:最多一个消息可以有25个路由条目,因此消息的大小最多为4+25*20=504,再加上UDP的8个字节头,此时就有512字节。
- Subnet Mask(32bit):子网掩码
- Next Hop(32bit):下一跳,MA中会自动更改,也可以手工设置
- metric(32bit):度量值(也可以成为distance或hop)

##### 认证包
![](image/_1528888998_1626582515.png)
占用第一个路由条目的位置做认证信息,注意此时`Address Family Identifier`位置设置成了 0xFFFF,Authentication Type 设置成 0x0002。所以此时最大可以包含的路由条目是 24,如果是 MD5 认证,Authentication Type 设置成 0x0003。

### 自动汇总
虽然 v2 携带了掩码信息,但跨越不同网络边界时,还是会自动汇总成主类。
![](image/_1528889106_565935250.png)
使用指令:`no auto-summary`,关闭自动汇总

## 功能
### 被动接口
- 第一种配置方式:将接口设为被动接口,只收不发。
    ```
    Router(config-router)#passive-interface FastEthernet 0/1
    ```
- 第二种配置方式:设备所有接口皆为被动接口,除了FastEthernet 0/1接口以外。
    ```
    Router(config-router)#passive-interface default
    Router(config-router)#no passive-interface FastEthernet 0/1
    ```
- 第三种配置方式: Passive-interface 是为了阻止该接口发送广播和组播的消息,但是可以通过手工指定 neighbor 来发送单播。正因为 neighbor 发送单播,所以在 neighbor X.X.X.X 可达的情况下(有路由)并且源检测通过的情况下(no validate-update-sourse),RIP 是可以跨越路由器更新路由的(update的TTL是2)
    ```
    Router(config)#router rip
    Router(config-router)#passive-interface default
    Router(config-router)#neighbor 12.1.1.2
    ```

### 汇总(v2)
- 本地只有明细路由,从做汇总的接口发出汇总路由。
- 直到明细的最后一条路由消失,汇总才会消失。
- 若有对相同网络的多个汇总条目,以最大掩码范围汇总。
- 强烈建议精确汇总
    注意:若 RIPv2 开启 auto-summary 之后,手工汇总失效(默认情况下,不管是 RIPv1 还是 RIPv2,都会
在主类网络边界自动汇总)
**RIPv2可以使用no auto-summary关闭边界自动汇总,对于RIPv1,此命令没有作用**。

命令:`Router(config-if)#ip summary-address rip 192.168.0.0 255.255.0.0`(在路由流向
的出接口使用)

### 偏移列表
用来调整路由的metric值(范围:0-16)

配置命令:
```
Router(config)#access-list 1 permit 2.2.2.0 0.0.0.255(匹配所需的路由条目)
Router(config-router)#offset-list 1 out 2 FastEther 0/2(调整Metric值(+3))
```

- out关键字:影响下游设备(对本设备无效)
- in关键字:既影响本地也影响下游设备

### 下放路由
- 在RIP进程中通告
    ```
    Router(config)#ip route 0.0.0.0 0.0.0.0 FastEther 0/1(只能写出接口)
    Router(config)#router rip
    Router(config-router)#network 0.0.0.0
    ```
- 重分布静态
    ```
    Router(config)#ip route 0.0.0.0 0.0.0.0 FastEther 0/1(随便写)
    Router(config)#router rip
    Router(config-router)#redistribute static metric X
    ```
- 在RIP进程中使用命令产生默认路由
    ```
    Router(config-router)#default-information originate ?
        route-map Route-map reference(注入默认路由的条件)
    ```

### 分发列表
路由过滤(控制)
配置命令:
```
access-list 1 permit 1.1.1.0 0.0.0.255
Router(config-router)#distribute-list 1 in fastEthernet 0/0
    只收邻居传递来的1.1.1.0/24的路由
Router(config-router)#distribute-list 1 out connected
    只能针对于其他协议重分发进来的路由做过滤,不能对邻居通告的路由做过滤。
```

### 路由源检查
使用命令:
```
Router(config-router)#validate-update-source(默认开启)
```
命令解释:

- 默认情况下RIP是要进行路由源检查的。也就是说,当一个接口收到路由更新时,会查看更新信息的源是否与该接口处于相同网段。如果不在就丢包,表明源检测失败。如果在同一网段,则源检测成功。
- 如果不需要此feature,可以使用命令`no validate-update-source`来关闭。关闭之后所有接口若
收到RIP更新,则不再进行源检查。
**注意**:关闭水平分割的接口是进行源检查的(关闭不了)。

### 计时器
- Update(更新):30秒,随机变量是更新周期的15%,即4.5秒(25.5秒-30秒)
- Invalid(失效):180秒,180秒后置为Possible Down状态并且立即启动hold Down计时器。
- Hold Down(抑制):180秒,实际只用60秒。
- Flush(刷新):240秒,240秒还没收到路由更新,将此路由删除。
- Sleep Time(休眠时间):(随机)触发更新计时器,单位毫秒 。触发更新不会引起接收路由器重置它们的更新计时器;因为如果这么做的话,网络拓扑的改变会造成触发更新“风暴”,还需要使用另外一个计时器,当一个触发更新传播时,这个计时器被随机的设置为1-5s之间的数值;在这个计时器超时前不能发送并发的触发更新。
- Output-delay(输出延迟):30毫秒,避免路由更新数据包发送速度太快导致老设备来不及处理。
![](image/_1528889872_995036014.png)
配置命令:
```
Router(config-router)#timers basic X Y Z
```

### 路由验证(v2)
- 验证的目的:为了防止攻击者向网络中注入非法的路由条目
- 验证方向:只在收路由的时候才进行验证

#### 配置步骤
> 配置注意点:
>
> - 别手欠,不能加空格!
> - 区分大小写
> - RIP中每一个路由更新包中最大可容纳25条路由,做了明文认证后只能有24条,做了MD5认证后只能有23条。

1. 设置密码串
    ```
    Router(config)#key chain QYT-Router(本地有效)
    Router(config-keychain)#key 1(建议两端一致)
        可以定义多个KEY值,并且按从小到大的顺序匹配
        发送KEY值时也是发送最小的一个,在配置KEY值的同时也可以配置密钥有效时间。
    Router(config-keychain-key)#key-string cisco
    ```
2. 接口下调用密码串
    ```
    Router(config-if)#ip rip authentication key-chain QYT-Router
    ```
3. 接口下指定认证方式
    ```
    Router(config-if)#ip rip authentication mode [md5密文|text明文](命令不显示)
    ```
4. 密码时间有效设置
    ```
    Router(config-keychain-key)#accept-lifetime 20:00:00 feb 2015 infinite(定时接收)
    Router(config-keychain-key)#send-lifetime 20:00:00 feb 2015 20:01:00 feb 2015(定时发送)
    Router(config-keychain-key)#send-lifetime 20:00:00 feb 2015 duration 300(有效期300秒)
    ```

#### 多个Key的匹配方式

- 多个key(明文):
    - 发:最小的key-id的密钥(只有key的内容,不包含Key-ID号码本身)。
    - 收:把收到的密钥跟本地key-chain中的密钥作比较,匹配通过,不匹配就验证失败。
- 多个key(密文):
    - 发:最小的key-id的密钥(包含key-id和key)
    - 收:收到密钥后,先匹配本地key-chain的key-id:
        - 如果存在这个key-id,那么就比较密码,相同就PASS,不同则验证失败 。
        - 如果不存在这个key-id,那么就匹配key-id+1的这个key。
        - 如果key-id+1的key存在,那么相同就PASS,不同则验证失败 。
        - 如果key-id+1的key不存在,那么继续匹配key-id+2,以此类推......

## 实施存在的问题和解决方案
### RIP 环路
- 链路出现问题
    ![](image/_1528890305_1473506898.png)
- C 询问 B，如何抵达网络 10.4.0.0
    ![](image/_1528891465_1760854969.png)
- 无线循环下去
    ![](image/_1528891441_1896174625.png)
- 产生环路
    ![](image/_1528891513_2046683500.png)
- 定义最大跳数
    ![](image/_1528891540_682569964.png)

#### 解决方案

##### Defining a Maximum
跳数范围为1-15,设置16为不可达。

##### Split Horizon(水平分割)
从一个接口收到的路由信息不再从此接口发出。
配置命令:
```
Router(config-if)#no ip split-horizon
```

- 简单水平分割:从本接口收到的路由条目不再从本接口发出
    ![](image/_1528891624_65074250.png)
- 毒性逆转的水平分割:是简单水平分割的加强版。从本接口收到的路由条目也会从本接口发出,但会标记为不可达。
    ![](image/_1528891648_1288646141.png)

> 注意:
>
> - 只有帧中继(FR)中的主接口(物理接口)是默认关闭水平分割的。
>     - FR:点到点子接口
> - 多点子接口---如果用在公司总部,建议手工关闭。
> - 点到点串口链路以及以太网链路均默认开启水平分割。

##### Route Poisoning(路由中毒)
将不可达的路由直接设成infinit(无限大)

![](image/_1528891715_1269589333.png)

##### Holddown Timers
所有邻居都将此路由“冻结”:

- 如在“冻结”期内该路由恢复,继续采纳该路由
- 如在“冻结”期收到更好的路由,将采纳更好的路由
- 如在“冻结”期收到更差的路由,不采纳该路由

##### Triggered Updates
避免周期性更新占用带宽,只有当拓扑变化时才发送更新。
> 注意:默认RIP的触发更新功能是关闭的。

![](image/_1528891754_1240795830.png)

配置命令:`Router(config-if)#ip rip triggered`

FR:

- 主接口:命令报错
- 点到点子接口:0K
- 多点子接口:报错
- 以太网接口:不支持

> 注意:12.4的IOS会自动生成Timers basic 30 180 0 240(15.x的IOS没有)一端配置了触发更新之后,对于每一个请求都会发送并设置一个轮询计时器Poll,每个的周期为5秒,5秒内没收到确认就再发一个,发完6个还没收到,则轮询超时等待下一个普通的更新时间。触发状态从down开始,经过init和loading状态,最后full。

