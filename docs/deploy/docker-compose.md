## 1. Fastdfs
docker-compose-fastdfs.yml
```bash
version: '2'
services:
    fastdfs-tracker:
        hostname: fastdfs-tracker
        container_name: fastdfs-tracker
        image: season/fastdfs:1.2
        network_mode: "host"
        command: tracker
        volumes:
          - ./tracker_data:/fastdfs/tracker/data
    fastdfs-storage:
        hostname: fastdfs-storage
        container_name: fastdfs-storage
        image: season/fastdfs:1.2
        network_mode: "host"
        volumes:
          - ./storage_data:/fastdfs/storage/data
          - ./store_path:/fastdfs/store_path
        environment:
          - TRACKER_SERVER=172.17.17.80:22122
        command: storage
        depends_on:
          - fastdfs-tracker
    fastdfs-nginx:
        hostname: fastdfs-nginx
        container_name: fastdfs-nginx
        image: season/fastdfs:1.2
        network_mode: "host"
        volumes:
          - ./nginx.conf:/etc/nginx/conf/nginx.conf
          - ./store_path:/fastdfs/store_path
        environment:
          - TRACKER_SERVER=172.17.17.80:22122
        command: nginx
```
## 2. Zookeeper

```bash
docker run -d -p 2181:2181 -v /mysoft/zookeeper/data/:/data/ --name=zookeeper  --privileged zookeeper  #启动zk
```

## 3. Portainer

```bash
docker run -p 9000:9000 -p 8000:8000 --name portainer \
--restart=always \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /mydata/portainer/data:/data \
-d portainer/portainer
```

## 4. MinIO
默认Access Key和Secret都是minioadmin

```bash
docker run -p 9090:9000 --name minio \
  -v /mydata/minio/data:/data \
  -v /mydata/minio/config:/root/.minio \
  -d minio/minio server /data
```

## 4. nginx-fancyindex

```bash
docker run -d \
  -p 8083:80 \
  -p 8084:443 \
  -e HTTP_AUTH="off" \
  -e HTTP_USERNAME="admin" \
  -e HTTP_PASSWD="admin" \
  -v /home/my-files:/app/public \
  --restart unless-stopped \
  --mount type=tmpfs,destination=/tmp \
  80x86/nginx-fancyindex
```