# Nginx高并发原理

```text
events {
  use epoll;
  worker_connections 102400;
}
```
