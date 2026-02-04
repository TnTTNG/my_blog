---
title: 基于 Docker + Prometheus + Grafana 的全栈监控平台
date: 2026-01-31 20:31:00
tags:
    - 教程
    - Docker
categories: 系统搭建
---
## 项目背景与架构
+ **项目目标**：在单台云服务器上构建轻量级、高可用的运维监控体系，解决“服务器黑盒”问题，实现对主机硬件及 Docker 容器的实时监控。
+ **技术栈**：Docker, Docker Compose, Prometheus, Grafana, Node Exporter, cAdvisor。
+ **架构流向**：

> 数据源 (Node Exporter/cAdvisor) -> 拉取存储 (Prometheus) -> 数据展示 (Grafana)
>

## 环境准备
确保服务器已安装 `Docker` 和 `Docker Compose`。

```bash
# 1. 创建统一的项目目录，遵循运维规范
mkdir -p /opt/monitor/prometheus  # 存放 Prometheus 配置
mkdir -p /opt/monitor/grafana     # 存放 Grafana 持久化数据
cd /opt/monitor
```

## 配置文件编写
### 配置 Prometheus (`prometheus.yml`)
创建并编辑 `/opt/monitor/prometheus/prometheus.yml`，定义数据抓取任务。

```yaml
global:
  scrape_interval: 15s      # 全局抓取间隔
  evaluation_interval: 15s

scrape_configs:
  # 1. 监控 Prometheus 自身
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # 2. 监控宿主机硬件 (Node Exporter)
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  # 3. 监控 Docker 容器资源 (cAdvisor)
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

### 编写编排文件 (`docker-compose.yml`)
在 `/opt/monitor/` 下创建，一次性编排所有服务。

```yaml
version: '3.8'

services:
  # --- 核心数据库: Prometheus ---
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: always
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=15d' # 数据保留15天
    ports:
      - "9090:9090"
    networks:
      - monitor-net

  # --- 可视化面板: Grafana ---
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: always
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin # 初始密码
    ports:
      - "3000:3000"
    networks:
      - monitor-net

  # --- 宿主机采集器: Node Exporter ---
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: always
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
    networks:
      - monitor-net

  # --- 容器采集器: cAdvisor ---
  cadvisor:
    image: zcube/cadvisor:latest
    container_name: cadvisor
    restart: always
    privileged: true
    ports:
      - "8080:8080"
    devices:
      - /dev/kmsg
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    networks:
      - monitor-net

volumes:
  prometheus_data:
  grafana_data:

networks:
  monitor-net:
```

## 部署与启动
```bash
# 在 /opt/monitor 目录下执行
docker compose up -d

# 检查运行状态（确保所有 State 均为 Up）
docker compose ps
```

## Grafana 深度配置与可视化
访问地址：`http://<服务器IP>:3000` (账号/密码: admin/admin)

### 设置中文界面
1. 进入 **Profile (个人资料)** -> **Preferences**。
2. **Language** 选择 **简体中文** -> Save。

### 配置数据源
1. 左侧菜单 -> **连接** -> **数据源** -> **Add new data source**。
2. 选择 **Prometheus**。
3. **Connection URL** 填写：`http://prometheus:9090` (利用 Docker 内部 DNS 解析)。
4. 点击底部 **Save & Test**，出现绿色提示即成功。

### 导入核心仪表盘
左侧菜单 -> **仪表盘** -> **导入 (Import)**，输入以下 ID 并加载：

| 监控维度 | 推荐模板 ID | 关键指标 |
| --- | --- | --- |
| **宿主机监控** (Node Exporter) | **8919** | CPU 核数负载、内存明细、磁盘 I/O 读写、网络上下行流量 |
| **Docker 容器监控** (cAdvisor) | **14282** (或 193) | 单容器 CPU 使用率、单容器内存占用、网络流量排行 |


> **注意**：导入时在底部的 Data Source 选项中，务必选择刚刚创建的 Prometheus 数据源。
>

## 基于 Alertmanager 的自动化告警闭环
> 为了实现 7*24 小时无人值守监控，我们引入 Alertmanager 组件
>

流程：Prometheus (发现异常) -> Push -> Alertmanager (去重、分组、路由) -> Send -> Email (管理员)。

### 配置文件编写
#### 创建告警规则 (`rules.yml`)
在 /opt/monitor/prometheus/ 目录下创建 `rules.yml`，定义什么样的指标属于“故障”。

```bash
groups:
  - name: host_monitoring
    rules:
    - alert: HostCpuUsageTooHigh
      # 表达式含义：(1 - 空闲率) * 100 > 50%
      expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100) > 50
      for: 1m  # 持续 1 分钟才报警，避免抖动误报
      labels:
        severity: warning
      annotations:
        summary: "主机 {{ $labels.instance }} CPU 负载过高"
        description: "当前 CPU 使用率已超过 50% (当前值: {{ $value | printf \"%.2f\" }}%)"
```

#### 配置 Alertmanager (`config.yml`)
创建目录 `/opt/monitor/alertmanager `并新建`config.yml`。

> 注意：此处以 QQ 邮箱为例，密码必须使用 SMTP 授权码。
>

```bash
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.qq.com:465'      # SMTP 服务器
  smtp_from: '你的邮箱@qq.com'           # 发件人
  smtp_auth_username: '你的邮箱@qq.com'  # 账号
  smtp_auth_password: '你的授权码'       # 授权码 (非登录密码)
  smtp_require_tls: false                # 465端口通常为false

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h       # 报警未解决，1小时后重发
  receiver: 'email-receiver'

receivers:
  - name: 'email-receiver'
    email_configs:
      - to: '接收者邮箱@example.com'
        send_resolved: true # 故障恢复后发送通知
        headers:
          Subject: "【云服务器告警】 {{ .CommonAnnotations.summary }}"
```

#### 关联 Prometheus
修改 `/opt/monitor/prometheus/prometheus.yml`，加入规则文件和 Alertmanager 地址

```bash
# 在 global 块之后添加
rule_files:
  - "rules.yml"  # 引用刚刚写的规则文件

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
```

### 服务编排更新 (`docker-compose.yml`)
修改 `/opt/monitor/docker-compose.yml`，新增 Alertmanager 服务，并更新 Prometheus 的挂载。

```bash
services:
  # ... (其他服务保持不变) ...
  prometheus:
    # ... (image, ports 等保持不变) ...
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/rules.yml:/etc/prometheus/rules.yml  # <--- 新增：挂载规则文件
      - prometheus_data:/prometheus

  # --- 新增服务: 告警中心 ---
  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    restart: always
    volumes:
      # ⚠️ 关键点：将本地 config.yml 强制挂载为容器内的 alertmanager.yml 以覆盖默认配置
      - ./alertmanager/config.yml:/etc/alertmanager/alertmanager.yml
    ports:
      - "9093:9093"
    networks:
      - monitor-net
```

### 部署验证
#### 重启服务
由于修改了挂载卷配置，建议先删除旧容器再启动：

```bash
cd /opt/monitor
docker compose down
docker compose up -d
```

#### 压力测试（验证告警）
安装压力测试工具 stress，模拟 CPU 飙升场景

```bash
# 模拟 2 个核满载，持续 120 秒
stress -c 2 -t 120
```

#### 观察流程
+ **触发**：Prometheus 界面 (Status -> Alerts) 中，`HostCpuUsageTooHigh` 状态由 Green -> Yellow (Pending) -> Red (Firing)。
+ **发送**：Alertmanager 收到 Firing 信号，处理后投递邮件。
+ **恢复**：stress 结束后，收到标题带 [Resolved] 的恢复邮件。

### 常见问题排查
+ **问题 1：**Alertmanager 报错 connection refused 127.0.0.1:5001。
    - 原因：配置文件挂载路径错误，导致容器读取了镜像自带的默认配置（默认配置是指向本地 webhook 的）。
    - 解决：确保 docker-compose 中挂载目标路径为 `/etc/alertmanager/alertmanager.yml`。
+ **问题 2：**邮件发送失败 535 Authentication failed。
    - 原因：使用了邮箱登录密码，而非 SMTP 专用授权码。
+ **问题 3：**容器无法联网 i/o timeout。
    - 原因：Docker DNS 配置问题或云服务器安全组未放行。
+ **问题 4：**cAdvisor 报错 mountpoint for cpu not found 
    - 原因：ubuntu 24.04 默认启用 **Cgroup** **v2**，而 docler hub 上的 cadvisor 旧版本只支持 **Cgroup** **v1**
    - **解决：**通过更改 **zcube/cadvisor** 镜像并配置 **privileged** 模式解决版本兼容性问题

## 基于 PLG 的轻量级日志分析
> 为了解决“出现故障需要登录服务器一条条查 logs”的低效问题，引入 PLG 技术栈 (Promtail + Loki + Grafana)。
>

+ 优势：相比 ELK (Elasticsearch)，Loki 不建立全文索引，仅索引元数据（标签），内存占用极低，非常适合云服务器单机环境。
+ 流向：Promtail (自动发现容器) -> 采集日志 -> Loki (存储与索引) -> Grafana (统一查询)。

### 配置文件编写
首先创建日志组件的专用目录：

```yaml
mkdir -p /opt/monitor/loki
mkdir -p /opt/monitor/promtail
```

#### 配置 Loki 存储端 (`loki-config.yaml`)
创建`/opt/monitor/loki/loki-config.yaml`，定义日志如何存储。采用本地文件系统存储，简单高效。

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h
```

#### 配置 Promtail 采集端 (`promtail-config.yaml`)
创建 `/opt/monitor/promtail/promtail-config.yaml`。、

> 核心逻辑：通过挂载 Docker Socket，自动发现当前运行的所有容器，并利用 relabel 机制将 Docker 内部名称格式化为易读的标签。
>

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml # 记录采集游标，防止重启后日志重复或丢失

clients:
  - url: http://loki:3100/loki/api/v1/push # 推送给 Loki

scrape_configs:
  - job_name: system
    docker_sd_configs:
      - host: unix:///var/run/docker.sock # 监听 Docker 事件
        refresh_interval: 5s
    relabel_configs:
      # 只保留有名称的容器
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container' # 重命名标签为 container，方便检索
```

### 服务编排更新 (`docker-compose.yml`)
编辑 `/opt/monitor/docker-compose.yml`，新增 Loki 和 Promtail 服务，并挂载必要的数据卷。

```yaml
services:
  # ... (之前的 Prometheus, Grafana, Alertmanager 等保持不变) ...

  # --- 新增: 日志存储 Loki ---
  loki:
    image: grafana/loki:latest
    container_name: loki
    restart: always
    volumes:
      - ./loki/loki-config.yaml:/etc/loki/local-config.yaml
      - loki_data:/loki
    ports:
      - "3100:3100"
    networks:
      - monitor-net

  # --- 新增: 日志采集 Promtail ---
  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    restart: always
    volumes:
      - ./promtail/promtail-config.yaml:/etc/promtail/config.yml
      # ⚠️ 关键挂载 1: 允许 Promtail 访问 Docker 守护进程以发现容器
      - /var/run/docker.sock:/var/run/docker.sock
      # ⚠️ 关键挂载 2: 允许 Promtail 读取实际的日志文件 (只读模式)
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    networks:
      - monitor-net

volumes:
  prometheus_data:
  grafana_data:
  loki_data: # <--- 别忘了新增这个卷用于持久化日志

networks:
  monitor-net:
```

### 部署与 Grafana 配置
#### 重启服务
```yaml
cd /opt/monitor
docker compose up -d
# 检查 loki 和 promtail 状态是否为 Up
docker compose ps
```

#### 在 Grafana 接入数据源
+ 登录 Grafana -> Connections -> Data sources -> Add new data source。
+ 选择 Loki。
+ URL 填写：[http://loki:3100。](http://loki:3100。)
+ 点击 Save & Test，出现绿色提示 "Data source connected" 即成功。

### 如何进行日志排查
+ 进入 Grafana 左侧菜单 Explore (探索)。
+ 顶部数据源选择 Loki。
+ Label filters 选择 container = 你的应用容器名 (例如 cadvisor)。
+ 点击 Run query。
+ 效果：可以看到实时的应用日志流。配合 Live 按钮，可以像在终端使用 `tail -f` 一样监控线上日志。

---

## 简历上的项目描述
**项目名称：基于 Docker 的云服务器监控平台**

**项目描述：**  
针对单体云服务器缺乏可视化监控、故障排查困难的问题，基于 Docker 搭建了一套标准化的监控告警系统。实现了从硬件层到应用容器层的全方位可观测性。

**操作系统**：Ubuntu 24.04

**技术栈**：Docker + Prometheus + Grafana + Alertmanager + cAdvisor + Loki + Promtail 

**核心技术：**

+ **容器化基础设施**：使用 **Docker Compose** 统一编排 **Prometheus、Grafana、Loki** 等 6 个服务组件，实现了“一键拉起”的标准化部署环境。
+ **多维度监控**：通过 **Node Exporter** 采集宿主机 CPU 使用率、磁盘 IO 等核心指标；引入 **cAdvisor** 解决了 Docker 容器资源隔离不可见的问题，实现了对单容器内存泄漏的实时监测。
+ **可视化监控**：基于 **Grafana** 自定义监控仪表盘，配置了 **Prometheus 数据源**，实现了服务器资源使用率的秒级刷新展示。
+ **自动化告警**：编写 PromQL 规则（如 CPU > 90%），结合 **Alertmanager **实现了邮件实时告警，经**压测**验证，平均故障响应时间缩短至 2 分钟以内
+ **日志聚合分析：**引入**Promtail + Loki + Grafana **技术栈，实现 docker 容器日志的自动化采集和聚合，与传统 ELK 方案相比，资源占用更低

---

