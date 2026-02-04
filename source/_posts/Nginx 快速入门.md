---
title: Nginx 快速入门
date: 2026-02-04 23:01:00
tags: [Nginx, Web服务器]
categories: 中间件
---
# 介绍
> Nginx 是一个高性能的 Web 服务器，主要负责处理客户端（浏览器）和后端服务器（应用）之间的通信。  
>

# 用途
> nginx 最核心的三大用途
>
> + 反向代理
> + 负载均衡
> + 静态资源服务
>

## 反向代理
+ **原理**：用户访问网站时，其实是连接到 Nginx，由 Nginx 帮忙去后端获取数据再返回给用户。 
+ **价值**：
    - **安全**：隐藏了后端服务器的真实 IP，保护核心数据。  
    - **统一**：所有请求由 Nginx 统一接管，便于管理。  

## 负载均衡
+ **原理：**当访问量巨大时，Nginx 会按照规则（如轮询、权重）将请求平均分配给多台后端服务器。  
+ **价值：**
    - **稳定：** 防止某一台服务器因流量过大而累死（宕机)。
    - **高可用：** 如果一台坏了，Nginx 自动把请求转给其他正常的服务器。  

## 动静分离
+ **原理：** 将图片、CSS、JS、HTML 等不需要计算的静态文件，直接由 Nginx 快速返回，不经过后端应用服务器。
+ **价值**：
    - **提速：**Nginx 处理静态文件的速度远超 Tomcat/Python 等应用服务器。
    - **减负：**让昂贵的后端算力专心处理复杂的业务逻辑。

## 总结
 如果把一个 Web 系统比作一家**火爆的餐厅**，Nginx 扮演了以下角色：  

| **<font style="color:rgb(31, 31, 31);">功能</font>** | **<font style="color:rgb(31, 31, 31);">类比角色</font>** | **<font style="color:rgb(31, 31, 31);">解释</font>** |
| --- | --- | --- |
| **<font style="color:rgb(31, 31, 31);">反向代理</font>** | **<font style="color:rgb(31, 31, 31);">服务员</font>** | <font style="color:rgb(31, 31, 31);">顾客不需要进后厨找厨师，找服务员（Nginx）点菜即可。</font> |
| **<font style="color:rgb(31, 31, 31);">负载均衡</font>** | **<font style="color:rgb(31, 31, 31);">领班</font>** | <font style="color:rgb(31, 31, 31);">看到 1 号桌满了，领班（Nginx）带顾客去 2 号空桌，保证不拥堵。</font> |
| **<font style="color:rgb(31, 31, 31);">静态服务</font>** | **<font style="color:rgb(31, 31, 31);">自助水吧</font>** | <font style="color:rgb(31, 31, 31);">倒水这种简单事（静态资源），直接在水吧拿最快，不用麻烦大厨。</font> |


# 正向代理和反向代理
> 这里需要区分一下正向代理和反向代理的区别
>

⚔️ 正向代理 vs 反向代理

1. **正向代理 (Forward Proxy)**  
一句话概括：它是“客户端”的代理人。
+ **也就是谁在用它？** 用户（Client）主动使用的。
+ **谁被隐藏了？** **用户（Client）**。服务器只知道代理来访问了，不知道背后真正的用户是谁。
+ 生活通俗类比： 海外代购。
    - 你想买某品牌的限量包（目标服务器），但你去不了那个国家。
    - 你找了一个代购（正向代理）。
    - 代购去店里买了包发给你。
    - **结果：** 品牌店只知道是“代购”买了包，完全不知道其实是你买的。
+ **典型用途：**
    - **科学上网 (VPN)：**访问原本无法访问的网站。
    - **公司内网监控：**公司通过代理服务器上网，只允许员工访问特定网站。
2. **反向代理 (Reverse Proxy)**  
一句话概括：它是“服务端”的代言人。
+ **也就是谁在用它？** 网站管理员（Server）部署的。
+ **谁被隐藏了？** **服务器（Server）**。用户只知道访问了代理的地址，不知道背后具体是哪台机器在处理。
+ 生活通俗类比： 打客服电话。
    - 你有问题要咨询移动/联通（客户端请求）。
    - 你拨打 10086（反向代理）。
    - 10086 系统会自动把你转接到具体的“工号 8823”客服人员（后端服务器）。
    - **结果：** 你只知道你打给了 10086，根本不知道（也不需要知道）是哪位具体的客服在服务你。
+ **典型用途：**
    - **Nginx 负载均衡：**把流量分发给不同的服务器。
    - **安全防护：**作为防火墙，挡在核心数据库前面。

| **<font style="color:rgb(31, 31, 31);">特性</font>** | **<font style="color:rgb(31, 31, 31);">正向代理 (Forward Proxy)</font>** | **<font style="color:rgb(31, 31, 31);">反向代理 (Reverse Proxy)</font>** |
| --- | --- | --- |
| **<font style="color:rgb(31, 31, 31);">代理对象</font>** | **<font style="color:rgb(31, 31, 31);">客户端</font>**<font style="color:rgb(31, 31, 31);"> (Client)</font> | **<font style="color:rgb(31, 31, 31);">服务端</font>**<font style="color:rgb(31, 31, 31);"> (Server)</font> |
| **<font style="color:rgb(31, 31, 31);">架设方</font>** | <font style="color:rgb(31, 31, 31);">用户自己（或公司IT）设置</font> | <font style="color:rgb(31, 31, 31);">网站管理员设置</font> |
| **<font style="color:rgb(31, 31, 31);">谁被隐藏</font>** | **<font style="color:rgb(31, 31, 31);">用户 IP</font>**<font style="color:rgb(31, 31, 31);"> 被隐藏</font> | **<font style="color:rgb(31, 31, 31);">真实服务器 IP</font>**<font style="color:rgb(31, 31, 31);"> 被隐藏</font> |
| **<font style="color:rgb(31, 31, 31);">感知度</font>** | <font style="color:rgb(31, 31, 31);">用户</font>**<font style="color:rgb(31, 31, 31);">知道</font>**<font style="color:rgb(31, 31, 31);">自己在用代理</font> | <font style="color:rgb(31, 31, 31);">用户</font>**<font style="color:rgb(31, 31, 31);">不知道</font>**<font style="color:rgb(31, 31, 31);">自己在用代理</font> |
| **<font style="color:rgb(31, 31, 31);">典型代表</font>** | <font style="color:rgb(31, 31, 31);">VPN、梯子</font> | <font style="color:rgb(31, 31, 31);">Nginx、HAProxy</font> |
| **<font style="color:rgb(31, 31, 31);">方向</font>** | <font style="color:rgb(31, 31, 31);">出口 (Outbound)</font> | <font style="color:rgb(31, 31, 31);">入口 (Inbound)</font> |


# 负载均衡
> nginx 的负载均衡策略有两种
>
> + 内置策略
> + 扩展策略
>

## 1. 什么是负载均衡？(The Concept)
**一句话**：把所有的脏活累活（流量），平均分给那个“干活天团”（后端服务器集群），不让任何一个人累死。

+ **核心目的**：提升网站的吞吐量，增加稳定性（一台挂了还有别的）。
+ **Nginx 的角色**：它是那个**分发任务的组长**。

---

## 2. 核心组件：Upstream (服务器池)
在 Nginx 里，你不能直接把任务指派给某一台机器，你得先定义一个“组”。这个组在配置文件里叫 `upstream`。

+ **概念**：把一堆服务器打包成一个“资源池”。
+ **通俗理解**：这是你的**“后台工作小组”**名单。

---

## 3. 分配策略 (调度算法)
Nginx 这个“组长”怎么决定把下一个任务派给谁？它有几套不同的逻辑：

### A. 轮询 (Round Robin)
> **默认策略**
>

+ **逻辑**：排排坐，吃果果。第一个请求给 A，第二个给 B，第三个给 C，第四个又给 A……
+ **场景**：后端服务器配置都差不多，大家众生平等。

### B. 权重 (Weight)
> **能力越大，责任越大**
>

+ **逻辑**：给服务器打分。配置高的机器（比如 8核 CPU），权重设高点；配置低的，权重设低点。
+ **场景**：新旧服务器混用，新机器更能打，就多干点活。
    - _例子：Server A (权重3), Server B (权重1) -> 4个请求里，A 处理 3 个，B 处理 1 个。_

### C. IP Hash (IP哈希)
>  **专人专送**
>

+ **逻辑**：记仇（记人）。根据访客的 IP 地址计算。只要是你（同一个 IP）来的请求，永远固定发给同一台服务器。
+ **场景**：**为了保持登录状态**（Session）。
    - _如果不用于此：_ 你的请求一会儿飘到 A 服务器（有你的登录信息），一会儿飘到 B 服务器（没你的信息），你就会莫名其妙被踢下线。

### D. 最少连接 (Least Conn) 
> **谁闲给谁**
>

+ **逻辑**：Nginx 会看哪台服务器当前正在处理的连接数最少，就把新任务给它。
+ **场景**：有的请求处理很快，有的很慢。这个策略能保证不会有人在“摸鱼”，也不会有人被“累死”。

---

## 4. 容灾与状态 (Status & Failover)
除了分发，Nginx 还能管理组员的状态。

### A. Down (下线)
+ **含义**：告诉 Nginx 这台机器“请假了”。
+ **作用**：维护升级时，手动把它标记为 down，Nginx 就暂时不给它派活了。

### B. Backup (备胎)
+ **含义**：这台机器平时不干活，在旁边歇着。
+ **触发条件**：只有当**所有**正常工作的机器都挂了（忙不过来了），这个“备胎”才会紧急启动接客。

### C. 自动剔除 (Health Check)
+ **逻辑**：如果你派给 Server A 的活它没接（超时或报错），Nginx 会自动把它标记为“不可用”，并在接下来的一段时间内跳过它。
+ **通俗理解**：发现员工生病了，组长自动准假，让他休息一会儿再来看看好了没。

# 动静分离
> 为什么要做动静分离
>
> + **🚀**** 速度飞快**：Nginx 处理静态文件的能力比 Tomcat/Django 强 10 倍以上，并发能力极强。
> + **🔋**** 给后端减负**：让昂贵的后端服务器专注于处理复杂的业务逻辑，不再被图片加载这种琐事占用内存和 CPU。
> + **📂**** 维护方便**：静态文件可以直接由前端工程师更新到 Nginx 目录下，不需要重启后端的应用服务。
>

## 1. 核心概念：什么是“动”和“静”？
在网站的世界里，资源分为两类：

+ **🧱****静态资源 (Static)**：
    - **特点**：谁来访问都一样，不需要计算，拿走就能用。
    - **例子**：图片 (`.jpg`)、样式表 (`.css`)、脚本文件 (`.js`)、普通的 HTML 页面。
+ **⚙️****动态资源 (Dynamic)**：
    - **特点**：因人而异，需要经过服务器计算、查数据库才能生成。
    - **例子**：购物车数据、个人中心、搜索结果、API 接口。

## 2. 什么是“动静分离”？
**一句话解释**：把“搬运死文件”的粗活交给 **Nginx**，把“计算逻辑”的细活交给 **后端应用服务器**（如 Tomcat、Go、Python）。

+ **不分离的情况（大锅饭）**： 所有的请求（无论是看一张图，还是查账单）都丢给后端的 Tomcat 处理。Tomcat 既要算账又要搬图，累得半死，效率极低。
+ **分离的情况（各司其职）**：
    - **Nginx**：站在最前面。如果是图片/CSS，自己直接从硬盘读取返回（速度极快）。
    - **后端**：如果是复杂的业务逻辑，Nginx 再转发给 Tomcat/Java 处理。

# 从三个角度感受 Nginx
##  拯救混乱
**没有 Nginx 的世界**

> 所有用户直接拥挤到一台应用服务器上，服务器既要拿图片，又要算数据，还要抗并发，最终不堪重负。  
>
![](https://my-static-website-bucket.oss-cn-guangzhou.aliyuncs.com/1.svg)

 **有了 Nginx 的世界**

> Nginx 挡在最前面抗住压力，后端服务器只需专心干好自己的活，整个系统轻松扩容。  
>
![](https://my-static-website-bucket.oss-cn-guangzhou.aliyuncs.com/2.svg)

##  极速分流
> Nginx **动静分离**的魅力：它能瞬间识别请求类型，让简单的请求走“高速公路”，复杂的请求走“普通车道”。  
>
![](https://my-static-website-bucket.oss-cn-guangzhou.aliyuncs.com/3.svg)
##  安全护盾  
> Nginx **反向代理**的魅力：它隐藏了后端服务器的真实身份，把危险挡在门外。外界只知道 Nginx 的存在，不知道核心数据到底在哪。  
>
![](https://my-static-website-bucket.oss-cn-guangzhou.aliyuncs.com/4.svg)
# 查看 nginx 进程
`ps -ef | grep nginx`

```shell
root@1769126960986:~# ps -ef | grep nginx
root     1335626 1335605  0 Feb01 ?        00:00:00 nginx: master process nginx -g daemon off;
message+ 1341281 1335626  0 Feb01 ?        00:00:00 nginx: worker process
message+ 1341282 1335626  0 Feb01 ?        00:00:00 nginx: worker process
message+ 1341283 1335626  0 Feb01 ?        00:00:00 nginx: worker process
message+ 1341284 1335626  0 Feb01 ?        00:00:00 nginx: worker process
root     1538084 1536452  0 00:01 pts/1    00:00:00 grep --color=auto nginx
```

# nginx 进程模型
**master**：nginx 的主进程，负责读取和验证配置文件，管理 worker 进程

**worker**：nginx 的工作进程，负责处理实际的请求

`master`只有一个，`worker`可以有很多个，他们之间的关系就像老板和员工

`worker`进程的数量可以通过配置文件来调整

# nginx 启停
`nginx -s signal`

```shell
nginx -s signal
quit:	优雅停止
stop:	立即停止
reload: 重载配置文件
reopen: 重新打开日志文件
```

# nginx 快速部署静态文件
通过`nginx -V` 和 `nginx -t`来查看 nginx 配置文件路径

```shell
1.nginx -V
--http-client-body-temp-path=/var/lib/nginx/body 
--http-fastcgi-temp-path=/var/lib/nginx/fastcgi 
--http-proxy-temp-path=/var/lib/nginx/proxy 
--http-scgi-temp-path=/var/lib/nginx/scgi 
--http-uwsgi-temp-path=/var/lib/nginx/uwsgi 
--with-compat --with-debug --with-pcre-jit --with-http_ssl_module --with-http_stub_status_module --with-http_realip_module --with-http_auth_request_module --with-http_v2_module --with-http_dav_module --with-http_slice_module --with-threads --with-http_addition_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_secure_link_module --with-http_sub_module --with-mail_ssl_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-stream_realip_module --with-http_geoip_module=dynamic --with-http_image_filter_module=dynamic --with-http_perl_module=dynamic --with-http_xslt_module=dynamic --with-mail=dynamic --with-stream=dynamic --with-stream_geoip_module=dynamic=

2.nginx -t
root@1769126960986:~# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

```

