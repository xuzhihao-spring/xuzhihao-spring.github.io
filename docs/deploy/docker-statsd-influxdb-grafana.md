# samuelebistoletti/docker-statsd-influxdb-grafana

## 1.Versions

- Docker Image: 2.3.0
- Ubuntu: 18.04
- InfluxDB: 1.7.10
- Telegraf (StatsD): 1.13.3-1
- Grafana: 6.6.2

## 2.Quick Start


```bash
docker run --ulimit nofile=66000:66000 \
  -d \
  --name docker-statsd-influxdb-grafana \
  -p 3003:3003 \
  -p 3004:8888 \
  -p 8086:8086 \
  -p 8125:8125/udp \
  samuelebistoletti/docker-statsd-influxdb-grafana:latest
```
## 3.Mapped Ports

```
Host        Container       Service
3003        3003            grafana
3004        8888            influxdb-admin (chronograf)
8086        8086            influxdb
8125        8125            statsd
```

## 4.Grafana
Open http://localhost:3003
```bash
Username: root
Password: root
```

Add data source on Grafana

```bash
Url: http://localhost:8086
Database:    telegraf
User: telegraf
Password:    telegraf
```

## 5.InfluxDB
Open [http://localhost:3004]

```
Username: root
Password: root
Port: 8086
```
