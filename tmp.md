# Linux Virtual Server
系统扩展方式：

Scale UP：向上扩展，增强

Scale Out：向外扩展，增加设备，集群。---面临服务调度分配问题

Cluster：集群，为解决某个特定问题将多台计算机组合起来形成的单个系统

Linux Cluster类型：
-   LB：Load Balancing，负载均衡
-   HA：High Availiablity，高可用，
    -   SPOF（single Point Of failure）
    -   MTBF:Mean Time Between Failure 平均无故障时间
    -   MTTR:Mean Time To Restoration(repair）平均恢复前时间
    -   A=MTBF/(MTBE+MTTR)(0,1):99%,99.5%,99.9%,99.99%,99.999%
-   HPC：High-performance computing，高性能计算

分布式系统：
-   分布式存储：云盘
-   分布式计算：hadoop，Spark

LB Cluster的实现方式

硬件
-   F5.Big-IP
-   Citrix Netscaler
-   A10 A10

软件
-   LVS:Linux Virtual Server
-   nginx：支持七层调度
-   haproxy：支持七层调度
-   ats：apache traffic server,yahoo捐助
-   perlbal：Perl编写
-   pound

基于工作的协议层次划分：
-   传输层（通用）：DPORT1
    -   LVS:
    -   nginx:stream
    -   haproxy:mode tcp
-   应用层(专用)：针对特定协议，自定义的请求模型分类
    -   proxy server:
        -   http: nginx,httpd,haproxy(mode http),..
        -   fastcgi:nginx,httpd,..
        -   mysql:mysql-proxy,..

会话保持：负载均衡
-   session sticky：同一用户调度固定服务器
-   Source IP：LVS sh算法（对某一特定服务而言）[Cookie]
-   session replication：每台服务器拥有全部session[session multicast cluster]
-   session server：专门的session服务器[Memcached,Redis]

HA集群实现方案
-   keepalived:vrrp协议
-   ais:应用接口规范
heartbeat
cman+rgmanager(RHCS)
coresync_pacemaker
