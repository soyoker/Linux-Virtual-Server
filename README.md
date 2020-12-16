# Linux-Virtual-Server
### 简介
LVS(Linux Virtual Server) 是Unix-like系统中的一个虚拟服务器，是国内贡献给开源组织的一个最优秀的项目之一，作者章文嵩博士。LVS在Unix-like系统中是作为一个前端(Director)存在的,又称为调度器，它本身不提供任何的服务，只是将通过互联网进来的请求接受后再转发给后台运行的真正的服务器(RealServer)进行处理，然后响应给客户端。
### 工作原理

<img src="images/main.jpg">

1、当用户向负载均衡调度器（Director Server）发起请求，调度器将请求发往至内核空间

2、PREROUTING链首先会接收到用户请求，判断目标IP确定是本机IP，将数据包发往INPUT链

3、IPVS是工作在INPUT链上的，当用户请求到达INPUT时，IPVS会将用户请求和自己已定义好的集群服务进行比对，如果用户请求的就是定义的集群服务，那么此时IPVS会强行修改数据包里的目标IP地址及端口，并将新的数据包发往POSTROUTING链

4、POSTROUTING链接收数据包后发现目标IP地址刚好是自己的后端服务器，那么此时通过选路，将数据包最终发送给后端的服务器

### LVS组件

IPVS（ip virtual server）：ipvs是LVS的核心组件，它本身只是一个框架，类似于iptables，工作于内核空间中,是真正生效实现调度的代码。

IPVSADM：ipvsadm 是用来定义LVS的转发规则的，工作于用户空间中,负责为ipvs内核框架编写规则，定义谁是集群服务，而谁是后端真实的服务器（Real Server）。

### LVS相关术语

-   DS：Direstor  Server 调度服务器，指LVS服务器
-   RS：Real Server 真正提供服务的服务器
-   VIP：virtual IP 调度服务器面对客户端外部所使用的虚拟 IP ,作为用户请求的目标的IP地址。
-   DIP：Director Server IP 主要用于和内部主机通讯的IP地址
-   RIP：Real Server IP 
-   CIP：Client IP 客户端IP

### LVS工作模式

-   lvs-nat  （Network Address Translation）建议小规模使用。

-   lvs-dr （Direct Routing 直接路由）建议大规模使用，也是现在较多使用场景的方法。

-   lvs-tun  （Tunneling 隧道）lvs-tun模型比较少用，因为他不能实现全局负载均衡，不能根据用户区域的距离来挑选最近的机房。这个最多为了实现异地容灾来实现的。比方说日本的机房地震了，而此时美国的机房仍然可使用，那么只要将指向到美国机房即可。而一般只有这种场景下才会用到隧道机制。

### 模型解析

####    LVS-NAT模型

重点：修改目标IP地址为挑选出的RS的IP地址。

<img src="images/nat.jpg">

1、当用户请求到达Director Server，此时请求的数据报文会先到内核空间的PREROUTING链。此时报文的源IP为CIP，目标IP为VIP。

2、PREROUTING检查发现数据包的目标IP是本机，将数据包送至INPUT链。

3、IPVS比对数据包请求的服务是否为集群服务，若是，修改数据包的目标IP地址为后端服务器IP，然后将数据包发至POSTROUTING链。此时报文的源IP为CIP，目标IP为RIP。

4、POSTROUTING链通过选路，将数据包发送给Real Server。

5、Real Server比对发现目标为自己的IP，开始构建响应报文发回给Director Server。此时报文的源IP为RIP，目标IP为CIP。

6、Director Server在响应客户端前，此时会将源IP地址修改为自己的VIP地址，然后响应给客户端。此时报文的源IP为VIP，目标IP为CIP。

####    LVS-NAT模型的特性：

-   RS应该使用私有地址，RS的网关必须指向DIP

-   DIP和RIP必须在同一个网段内

-   请求和响应报文都需要经过Director Server，高负载场景中，Director Server易成为性能瓶颈

-   支持端口映射

-   RS可以使用任意操作系统

缺陷：对Director Server压力会比较大，请求和响应都需经过director server

####  LVS-DR模型  

重点：将请求报文的目标MAC地址设定为挑选出的RS的MAC地址

<img src="images/dr.jpg">

1、当用户请求到达Director Server，此时请求的数据报文会先到内核空间的PREROUTING链。此时报文的源IP为CIP，目标IP为VIP。

2、PREROUTING检查发现数据包的目标IP是本机，将数据包送至INPUT链

3、IPVS比对数据包请求的服务是否为集群服务，若是，将请求报文中的源MAC地址修改为DIP的MAC地址，将目标MAC地址修改RIP的MAC地址，然后将数据包发至POSTROUTING链。此时的源IP和目的IP均未修改，仅修改了源MAC地址为DIP的MAC地址，目标MAC地址为RIP的MAC地址

4、由于DS和RS在同一个网络中，所以是通过二层来传输。POSTROUTING链检查目标MAC地址为RIP的MAC地址，那么此时数据包将会发至Real Server。

5、RS发现请求报文的MAC地址是自己的MAC地址，就接收此报文。处理完成之后，将响应报文通过lo接口传送给eth0网卡然后向外发出。此时的源IP地址为VIP，目标IP为CIP

6、响应报文最终送达至客户端

####    LVS-DR模型的特性

-   保证前端路由将目标地址为VIP报文统统发给Director Server，而不是RS
解决方案：

    -   在前端路由器做静态地址路由绑定，将对于VIP的地址仅路由到Director Server
存在问题：用户未必有路由操作权限，因为有可能是运营商提供的，所以这个方法未必实用

    -   arptables：在arp的层次上实现在ARP解析时做防火墙规则，过滤RS响应ARP请求。这是由iptables提供的

    -   修改RS上内核参数（arp_ignore和arp_announce）将RS上的VIP配置在lo接口的别名上，并限制其不能响应对VIP地址解析请求。（最容易实现）

-   RS可以使用私有地址；也可以是公网地址，如果使用公网地址，此时可以通过互联网对RIP进行直接访问

-   RS跟Director Server必须在同一个物理网络中

-   所有的请求报文经由Director Server，但响应报文必须不能进过Director Server

-   不支持地址转换，也不支持端口映射

-   RS可以是大多数常见的操作系统

-   RS的网关绝不允许指向DIP(因为我们不允许他经过director)

-   RS上的lo接口配置VIP的IP地址

缺陷：RS和DS必须在同一机房中

####    LVS-Tun模型

重点：在原有的IP报文外再次封装多一层IP首部，内部IP首部（源地址为CIP，目标IIP为VIP），外层IP首部（源地址为DIP，目标IP为RIP）

<img src="images/tun.jpg">

1、当用户请求到达Director Server，此时请求的数据报文会先到内核空间的PREROUTING链。此时报文的源IP为CIP，目标IP为VIP。

2、PREROUTING检查发现数据包的目标IP是本机，将数据包送至INPUT链

3、IPVS比对数据包请求的服务是否为集群服务，若是，在请求报文的首部再次封装一层IP报文，封装源IP为为DIP，目标IP为RIP。然后发至POSTROUTING链。此时源IP为DIP，目标IP为RIP

4、POSTROUTING链根据最新封装的IP报文，将数据包发至RS（因为在外层封装多了一层IP首部，所以可以理解为此时通过隧道传输）。此时源IP为DIP，目标IP为RIP

5、RS接收到报文后发现是自己的IP地址，就将报文接收下来，拆除掉最外层的IP后，会发现里面还有一层IP首部，而且目标是自己的lo接口VIP，那么此时RS开始处理此请求，处理完成之后，通过lo接口送给eth0网卡，然后向外传递。此时的源IP地址为VIP，目标IP为CIP

6、响应报文最终送达至客户端

####    LVS-Tun模型特性

-   RIP、VIP、DIP全是公网地址
-   RS的网关不会也不可能指向DIP
-   所有的请求报文经由Director Server，但响应报文必须不能进过Director Server
-   不支持端口映射
-   RS的系统必须支持隧道

### LVS算法调度
