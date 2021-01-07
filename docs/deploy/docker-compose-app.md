## App部署

```bash
version: '3'
services:
  springboot-aop:
    image: webapp/springboot-aop:0.0.1-SNAPSHOT
    container_name: springboot-aop
    ports:
      - 8080:8080
    volumes:
      - /mydata/springboot_test:/var/logs
    environment:
      - 'TZ="Asia/Shanghai"'
```