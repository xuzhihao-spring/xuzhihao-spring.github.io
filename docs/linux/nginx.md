# Nginx配置

## nginx.conf

```xml
worker_processes 32;
error_log /var/log/nginx/nginx_error.log crit;
pid /var/run/nginx.pid;
worker_rlimit_nofile 102400;
events {
  use epoll;
  worker_connections 102400;
}

http {
  include mime.types;
  default_type application/octet-stream;
  server_names_hash_bucket_size 128;
  client_header_buffer_size 128k;
  large_client_header_buffers 4 128k;
  client_max_body_size 356m;
  log_format access '$host - $server_addr $remote_addr - $remote_user [$time_local] "$request" '
  '$status $body_bytes_sent "$http_referer" '
  '"$http_user_agent" "$remote_addr" "$http_x_forwarded_for" "$proxy_add_x_forwarded_for" "$http_x_real_ip" "$proxy_add_x_forwarded_for"';

  log_format post '$host - $remote_addr\t$remote_user\t[$time_local]\t"$request"\t$status\t$bytes_sent\t';

  log_format ip '$host - $remote_addr - $remote_user [$time_local] "$request" - [$upstream_addr] - [$upstream_status]';

  log_format rc escape=json '$remote_addr - $remote_user [$time_local] "$request" '
  '$status $body_bytes_sent "$http_referer" '
  '"$http_user_agent" "$http_x_forwarded_for" $request_body';

  #log_format fs '$domain  - $real';
  #log_format test '$real';

  access_log /var/log/nginx/ip_access.log post;
  sendfile on;
  tcp_nopush on;

  open_file_cache max=102400 inactive=20s;
  open_file_cache_valid 30s;
  open_file_cache_min_uses 1;
  keepalive_timeout 20s;
  server_tokens off;

  proxy_connect_timeout 30s;
  proxy_send_timeout 150s;
  proxy_read_timeout 150s;
  proxy_buffer_size 1024k;
  proxy_buffers 4 1024k;
  proxy_busy_buffers_size 1024k;
  proxy_temp_file_write_size 1024k;

  gzip on;
  gzip_vary on;
  gzip_min_length 1k;
  gzip_buffers 4 16k;
  gzip_http_version 1.1;
  gzip_comp_level 3;
  gzip_types text/xml application/xml application/atom+xml application/rss+xml application/xhtml+xml image/svg+xml text/javascript application/javascript 
application/x-javascript text/x-json application/json application/x-web-app-manifest+json text/css text/plain text/x-component font/opentype 
application/x-font-ttf application/vnd.ms-fontobject image/x-icon;

  include /usr/local/openresty/nginx/conf/blob.xuzhihao.com.cn_upstream.conf;
  #--------------------------------------------------------------------------------------
  include /usr/local/openresty/nginx/conf/blob.xuzhihao.com.cn.conf;
}
```

## blob.xuzhihao.com.cn_upstream.conf
```xml
upstream blob.xuzhihao.com.cn {
  ip_hash;
  server 192.168.3.200:7535;
}
```

## blob.xuzhihao.com.cn.conf
```xml
server {
  listen 480;
  server_name oaatt.vjsp.cn;
  location / {
    proxy_redirect off;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_pass http://oaatt.vjsp.cn;
  }
  access_log /var/log/nginx/oaatt_access.log;
}
server {
  listen 80;
  server_name oaatt.vjsp.cn;
  rewrite ^(.*)$ https://$host$1 permanent;
}
server {
  listen 443 ssl;
  server_name oaatt.vjsp.cn;
  charset utf-8;
  #ssl on;
  ssl_certificate /usr/local/openresty/nginx/conf/cer/oaatt.vjsp.cn/vjsp.cn.pem;
  ssl_certificate_key /usr/local/openresty/nginx/conf/cer/oaatt.vjsp.cn/vjsp.cn.key;
  ssl_session_timeout 5m;
  ssl_session_cache shared:SSL:20m;
  ssl_buffer_size 256k;
  ssl_session_tickets on;
  ssl_stapling on;
  ssl_stapling_verify on;
  resolver 223.5.5.5 114.114.114.114 180.76.76.76 valid=300s;
  resolver_timeout 10s;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_ciphers HIGH:!RC4:!MD5:!aNULL:!eNULL:!NULL:!DH:!EDH:!EXP:+MEDIUM;
  ssl_prefer_server_ciphers on;
  index index.html index.htm index.jsp index.do index.action;
  access_log /var/log/nginx/oaatt_access.log;
  location / {
    proxy_redirect off;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    add_header Access-Control-Allow-Origin *;
    add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
    add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,
Authorization';
    proxy_pass http://oaatt.vjsp.cn;
  }
}
```

## 泛域

```text
server
{ 
    listen 80;
	server_name  ~^(?<serno>.+).xuzhihao.com.cn$;
	server_name_in_redirect off; 
	location / {
		rewrite ^(.*)$ /$serno$1 break;
		#proxy_pass http://127.0.0.1:8080;
		root D:/workspace/;
		proxy_set_header   Host    $host;
		proxy_set_header   X-Real-IP   $remote_addr;
		proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
	}
	access_log logs/web.log;
}
```

## 2级路径

```text
server 
 {
	listen 80;
	server_name  www.xuzhihao.com.cn;
	charset utf-8; 
	index index.html;
	location / {
		root D:/webapp;
	}
    location /dashboard {
    	alias D:/dashboard;
    }
	location /blob/ {
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://192.168.3.200:8000/blob/;
    }
}
```