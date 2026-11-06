# 实机验收记录

- 日期：2026-11-06（Asia/Shanghai）
- 环境：4 台 Rocky Linux 9，控制端使用 Ansible Core 2.14

## 服务部署

| 验收项 | 实测结果 |
| --- | --- |
| Ansible 连通性 | Web、数据库、监控节点均返回 `pong` |
| Web | 控制节点和 Windows 都收到演示页 HTTP 200 |
| 数据库 | `ops@localhost` 通过 Unix socket 查询，返回 Web、数据库、监控三条样例记录 |
| Prometheus | 四台节点均为 `up=1` |
| Grafana | Windows 健康接口 HTTP 200。认证后的仪表盘 API 返回 4 个面板，Grafana 的 Prometheus 代理接口返回 4 台在线节点 |

## 备份与恢复

- 已在 Web 和数据库节点启用定时器，每天上海时间 03:15 运行。
- 手动触发后，Web 归档 `nginx-site-20261106T114319900234815Z.tar.gz` 包含 `nginx.conf`，提取的页面与在用页面逐字节一致。
- 数据库归档 `mariadb-ops_lab-20261106T114319750015117Z.sql.gz` 通过压缩完整性校验，成功导入独立测试库。查询结果与原库一致：

```text
1  web-server
2  db-server
3  monitor-server
```

## 告警、日志和恢复演练

1. 先确认网页 HTTP 200，Nginx 与 Node Exporter 都在运行。
2. 设置三分钟后自动恢复，再暂停两个服务。控制节点无法访问网页，Prometheus 中 `web-server` 的 `LinuxNodeDown` 进入 `firing`。
3. 恢复两个服务。控制节点重新收到 HTTP 200，Prometheus 重新采集到 `up=1`，告警清除。
4. Nginx 服务日志记录 19:45:54 停止、19:46:45 启动，访问日志记录 19:46:47 返回 200。日志轮转配置通过 `logrotate -d` 核对，每日轮转并保留 10 份。

## 排障记录

- Node Exporter 安装包下载较慢，完成后 SHA-256 校验通过。解压失败定位到缺少 `tar`，补齐 `tar` 和 `gzip` 后复用缓存部署成功。
- Grafana 页面和健康接口正常，数据源查询曾返回 HTTP 502。服务日志显示连接 9090 被拒，SELinux 审计确认是 `grafana_t` 的连接权限。启用专用 `grafana_can_tcp_connect_prometheus_port` 开关后取数恢复，SELinux 保持 Enforcing。

## 完整部署复跑

在现有四节点环境再次运行 site.yml，四台均为 changed=0、unreachable=0、failed=0。

```text
db-server                  : ok=24   changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
monitor-server             : ok=32   changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
ops-server                 : ok=11   changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
web-server                 : ok=25   changed=0    unreachable=0    failed=0    skipped=4    rescued=0    ignored=0
online: 4 series, four nodes
cpu: 4 series, four nodes
memory: 4 series, four nodes
root_disk: 4 series, four nodes
active alerts: 0
```
