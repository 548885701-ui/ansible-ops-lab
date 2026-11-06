# 基于 Ansible 的企业 Linux 服务器自动化运维与监控平台

用四台 Rocky Linux 9 虚拟机模拟小型企业运维环境，覆盖批量巡检、服务部署、监控告警、定时备份、恢复演练和日志排查。

## 节点和端口

| 节点 | 实验网 IP | 职责 |
| --- | --- | --- |
| ops-server | 192.168.175.10 | Ansible 控制节点 |
| web-server | 192.168.175.11 | Nginx 和演示网页 |
| db-server | 192.168.175.12 | MariaDB 实验库 |
| monitor-server | 192.168.175.13 | Prometheus 和 Grafana |

- 网页 `http://192.168.175.11/ansible-lab.html`，TCP 80 的项目规则允许来源为 `192.168.175.0/24`。
- MariaDB 监听 `127.0.0.1:3306`。本项目用 Unix socket 查询和备份。
- 监控节点的 Node Exporter 监听 `127.0.0.1:9100`。其他三台监听各自的实验网 IP，项目防火墙规则仅放行监控节点 `192.168.175.13/32` 访问 TCP 9100。
- Prometheus 监听监控节点本机的 `127.0.0.1:9090`，每 15 秒抓取一次指标，保留 3 天。
- Grafana 监听 `192.168.175.13:3000`，允许实验网段访问。Grafana 通过本机的 Prometheus API 取数。

## 从头部署

前置条件为四台 x86_64 Rocky Linux 9 虚拟机，实验网卡名为 `ens192`，IP 与上表一致。启用 firewalld。各节点准备有 sudo 权限的 Linux 用户 `ops`，控制节点能够通过 SSH 公钥登录另外三台，并已核对它们的主机密钥。控制节点安装 `ansible-core`，节点可访问 Rocky 软件源，监控节点还需访问 GitHub 官方 Node Exporter 发布页和 EPEL 国内镜像。

在 **ops-server** 上解压并进入项目目录：

```bash
ansible-inventory -i inventory.ini --graph
ansible -i inventory.ini managed -m ping
ansible-playbook -i inventory.ini site.yml --syntax-check
ansible-playbook -i inventory.ini site.yml --ask-become-pass
```

`--ask-become-pass` 适用于各目标节点使用同一 sudo 密码的实验配置。各节点密码不同时，通过本机的 Ansible Vault 主机变量分别提供 `ansible_become_password`，保管库文件应独立于本项目。Node Exporter 的官方包由监控节点下载并校验，后续由控制节点缓存和分发。

| 文件 | 作用 |
| --- | --- |
| `inventory.ini` / `site.yml` | 主机清单 / 完整部署入口 |
| `health-check.yml` | 查询三台受管节点的主机名、负载和根分区 |
| `deploy-nginx.yml` / `publish-demo.yml` | 安装 Nginx / 发布网页、放行并跨机验证 HTTP |
| `deploy-mariadb.yml` | 部署数据库，建立 `ops_lab.assets` 和 Unix socket 只读账号 |
| `deploy-node-exporter.yml` / `expand-node-exporters.yml` | 采集器部署 / 安装包分发、定向防火墙和跨机验证 |
| `deploy-prometheus.yml` / `deploy-grafana.yml` | 四节点采集、离线告警 / 数据源和四面板仪表盘 |
| `deploy-backups.yml` | Web 归档和数据库转储，每日上海时间 03:15 运行 |
| `verify-lab.yml` / `drill-lab.yml` | 验证备份、恢复和仪表盘 / 演练停服、告警和恢复 |

## 查看配置和验收

在 Web 节点执行 `ss -ltn 'sport = :80'`，在控制节点请求演示 URL。用 `sudo firewall-cmd --get-active-zones` 查实验网卡所在区域，再用 `sudo firewall-cmd --zone=区域名 --list-rich-rules` 看当前规则，加 `--permanent` 查看保存的规则。用 `systemctl cat node_exporter` 查看实际启动命令，再用 `ss -ltn 'sport = :9100'` 检查监听。

在 **db-server** 以 Linux 用户 `ops` 查询样例记录：

```bash
mariadb --protocol=socket -u ops ops_lab -e 'SELECT id, hostname FROM assets ORDER BY id;'
```

在 **monitor-server** 查看采集状态与告警：

```bash
curl -fsS 'http://127.0.0.1:9090/api/v1/query?query=up'
curl -fsS http://127.0.0.1:9090/api/v1/alerts
getenforce
getsebool grafana_can_tcp_connect_prometheus_port
```

在浏览器打开 `http://192.168.175.13:3000/d/ops-linux-lab`，用 `admin` 登录。初始密码可在 **monitor-server 的本机终端** 用 `sudo cat /etc/grafana/ops-lab-admin.env` 查看，文件仅 root 可读。Grafana 用专用 SELinux 开关连接 Prometheus 9090。

在 **ops-server** 运行验收和演练：

```bash
ansible-playbook -i inventory.ini verify-lab.yml --ask-become-pass
ansible-playbook -i inventory.ini drill-lab.yml --ask-become-pass
```

验收会触发一份新备份，解包检查网页，将数据库备份导入独立临时库并对比三条记录，然后清理测试库。Grafana 验收使用部署时生成的初始管理员密码。停服演练会短暂停止实验 Web 节点的 Nginx 和 Node Exporter，等待 `LinuxNodeDown` 进入 firing，再启动服务并验证恢复。演练前会设置三分钟自动恢复兜底。

备份目录是 Web 和数据库节点各自的 `/var/backups/ops-lab/`，保留约 7 天。备份保存在原节点本地。告警状态在 Prometheus 内查看。Nginx 的访问日志和服务日志分别通过 `/var/log/nginx/access.log` 和 `journalctl -u nginx` 查阅。

实机证据见 [验收记录](docs/acceptance.md)。
