# Prometheus+Grafana

## 1. 下载

Prometheus是由SoundCloud开发的开源监控报警系统和时序列数据库(TSDB)。Prometheus使用Go语言开发，是Google BorgMon监控系统的开源版本

https://grafana.com/grafana/dashboards?dataSource=prometheus

- Prometheus 下载地址：https://prometheus.io/download/
- Grafana 下载地址：https://grafana.com/grafana/download
- node_exporter 下载地址：https://prometheus.io/download/#node_exporter

## 2. 安装

### 2.1 Prometheus

```bash
tar zxvf prometheus-2.27.0.linux-amd64.tar.gz
cd prometheus-2.27.0.linux-amd64/
./prometheus --config.file="/usr/local/prometheus/prometheus.yml" &
curl localhost:9090
```

### 2.2 Grafana

```bash
wget https://dl.grafana.com/oss/release/grafana-7.3.7-1.x86_64.rpm
yum localinstall grafana-7.3.7-1.x86_64.rpm -y
systemctl start grafana-server.service
systemctl enable grafana-server.service
curl localhost:3000
```

### 2.3 node_exporter

```bash
curl -OL https://github.com/prometheus/node_exporter/releases/download/v1.1.2/node_exporter-1.1.2.linux-amd64.tar.gz
tar -xzf node_exporter-1.1.2.linux-amd64.tar.gz
cp node_exporter-1.1.2.linux-amd64/node_exporter /usr/local/bin/
node_exporter
lsof -i:9100
```

https://grafana.com/grafana/dashboards/8919

### 2.4 mysqld_exporter

```sql
CREATE USER 'mysqld_exporter'@'localhost' IDENTIFIED BY '123456' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'mysqld_exporter'@'localhost';
flush privileges;
```

```bash
docker pull prom/mysqld-exporter
docker run -d -p 9104:9104 --network mydata_default --link=mysql:mysql -e DATA_SOURCE_NAME="mysqld_exporter:123456@(mysql:3306)/mysql" prom/mysqld-exporter
```

https://grafana.com/grafana/dashboards/7362

### 2.5 容器监控cAdvisor

docker-compose-cadvisor.yml
```yml
version: '2'
services:
  cadvisor:
    image: "google/cadvisor:v0.32.0"
    hostname: cadvisor
    container_name: cadvisor
    ports:
      - '18080:8080'
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    restart: always
```

```bash
docker-compose -f docker-compose-cadvisor.yml up -d  
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
        labels:
          instance: prometheus
        
  - job_name: linux
    static_configs:
      - targets: ['172.17.17.201:9100','172.17.17.200:9100']
        labels:
          instance: Linux

  - job_name: docker
    static_configs:
      - targets: ['172.17.17.80:18080']
        labels:
          instance: docker18
      
  - job_name: mysql
    static_configs:
      - targets: ['172.17.17.80:9104']
        labels:
          instance: mysql8
```