# yum安装grafana7.3.4

```bash
#wget https://dl.grafana.com/oss/release/grafana-7.3.4-1.x86_64.rpm
yum localinstall grafana-7.3.4-1.x86_64.rpm
start grafana-server.service
systemctl enable grafana-server.service
```