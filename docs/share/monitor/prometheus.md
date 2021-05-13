
# Prometheus+Grafana

## 1. 下载

Prometheus是由SoundCloud开发的开源监控报警系统和时序列数据库(TSDB)。Prometheus使用Go语言开发，是Google BorgMon监控系统的开源版本

https://grafana.com/grafana/dashboards?dataSource=prometheus

Prometheus 下载地址：https://prometheus.io/download/

Grafana 下载地址：https://grafana.com/grafana/download

node_exporter 下载地址：https://prometheus.io/download/#node_exporter

## 2. 安装

### 2.1 安装Prometheus

```bash
tar zxvf prometheus-2.27.0.linux-amd64.tar.gz
cd prometheus-2.27.0.linux-amd64/
curl localhost:9090
```

### 2.2 安装Grafana

```bash
wget https://dl.grafana.com/oss/release/grafana-7.3.7-1.x86_64.rpm
yum localinstall grafana-7.3.7-1.x86_64.rpm -y
systemctl start grafana-server.service
systemctl enable grafana-server.service
curl localhost:3000
```

### 2.3 安装node_exporter

```bash
curl -OL https://github.com/prometheus/node_exporter/releases/download/v1.1.2/node_exporter-1.1.2.linux-amd64.tar.gz
tar -xzf node_exporter-1.1.2.linux-amd64.tar.gz
cd node_exporter-1.1.2.linux-amd64
cp node_exporter-1.1.2.linux-amd64/node_exporter /usr/local/bin/
node_exporter
```

## 3. prometheus.yml配置文件

```yaml
# 全局配置（如果有内部单独设定，会覆盖这个参数）
global:
  scrape_interval: 15s      # 默认15s 全局每次数据收集的间隔
  evaluation_interval: 15s  # 规则扫描时间间隔是15秒，默认不填写是 1分钟
  scrape_timeout: 5s        # 超时时间 默认10s
  external_labels: lable    # 用于外部系统标签的，不是用于metrics(度量)数据

# 告警插件定义。这里会设定alertmanager这个报警插件
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# 告警规则。 按照设定参数进行扫描加载，用于自定义报警规则，其报警媒介和route路由由alertmanager插件实现
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# 采集配置。配置数据源，包含分组job_name以及具体target。又分为静态配置和服务发现
scrape_configs:
  - job_name: 'prometheus'
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
    - targets: ['localhost:9090']

  - job_name: 'Linux'
    static_configs:
    - targets: ['192.168.3.201:9100']
      labels:
        instance: Linux
```

## node_exporter安装

```
#创建目录
mkdir -p /opt/exporter
cd /opt/exporter
#下载安装包
wget https://github.com/prometheus/node_exporter/releases/download/v0.14.0/node_exporter-0.14.0.linux-amd64.tar.gz
wget https://github.com/prometheus/node_exporter/releases/download/v0.18.1/node_exporter-0.18.1.linux-arm64.tar.gz
#解压
tar -xvzf  node_exporter-0.14.0.linux-amd64.tar.gz
#修改名称
mv node_exporter-0.14.0.linux-amd64 node_exportercd /opt/exporter/node_exporter
#修改权限
chmod 777 node_exporter
#启动服务
nohup /opt/exporter/node_exporter/node_exporter &
#访问
curl http://IP:9100/metrics
```
```