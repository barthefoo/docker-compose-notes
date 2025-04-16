## 使用说明

* 在文件夹内 `foobar` 部新建 `docker-compose.yml` 文件
* 在  `foobar` 中执行 `docker compose up -d`
* 后续启动 `docker compose start` 和关闭 `docker compose stop`

* 查看日志 `docker compose logs 容器名`


# MySQL

## MySQL 8

```yaml
services:
  mysql:
    image: mysql:8.0.41
    container_name: mysql8
    environment:
      MYSQL_ROOT_PASSWORD: 124356
    ports:
      - "3306:3306"
    volumes:
      - ./data:/var/lib/mysql
      - ./conf.d:/etc/mysql/conf.d
      - ./init:/docker-entrypoint-initdb.d
    command:
      --default-authentication-plugin=mysql_native_password
```





# Redis

```yaml
services:
  redis:
    image: redis:7.2-alpine
    container_name: redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - ./data:/data
    command: >
      redis-server
      --port 6379
      --bind 0.0.0.0
      --requirepass 123456
      --appendonly yes
      --protected-mode no
```







# GitLab

> a5425d478c0b4405849ba67d028f502b.com 可以换成任意 `域名` 或者 `IP`

```yaml
services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    restart: unless-stopped
    hostname: 'gitlab.local'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://a5425d478c0b4405849ba67d028f502b.com:26001'
        gitlab_rails['gitlab_shell_ssh_port'] = 26002
        gitlab_rails['gitlab_ssh_host'] = 'a5425d478c0b4405849ba67d028f502b.com'
      GITLAB_ROOT_PASSWORD: "Foothebar!"
    ports:
      - '26001:26001'     # 网页访问
      - '26002:26002'     # SSH
    volumes:
      - './data/config:/etc/gitlab'
      - './data/logs:/var/log/gitlab'
      - './data/data:/var/opt/gitlab'
```



## Jenkins

```yaml
services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    restart: always
    environment:
      - JENKINS_USER=admin
      - JENKINS_PASS=123456
    ports:
      - "27001:8080"   # Jenkins UI
      - "27002:50000" # Agent 通信端口
    volumes:
      - ./data/jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock  # 让 Jenkins 容器内部能操作宿主机的 Docker（重要！）
    environment:
      - LANG=en_US.UTF-8

volumes:
  jenkins_home:
```





# Nacos

## 单机模式(Standalone)

> 需要先到 Nacos 的 GitHub 下载 Release https://github.com/alibaba/nacos
>
> 然后初始化 MySQL 数据
>
> up 之后访问 : `http://locahost:25001/nacos`

```yaml
services:
  nacos:
    image: nacos/nacos-server:v2.5.1
    container_name: nacos
    environment:
      - MODE=standalone
      - TZ=Asia/Shanghai
      - SPRING_DATASOURCE_PLATFORM=mysql
      - MYSQL_SERVICE_HOST=172.22.0.1 # MySQL IP
      - MYSQL_SERVICE_PORT=3306
      - MYSQL_SERVICE_USER=root # MySQL 账号
      - MYSQL_SERVICE_PASSWORD=123456 # MySQL 密码
      - MYSQL_SERVICE_DB_NAME=nacos_config # Nacos 数据库名
      - MYSQL_SERVICE_DB_PARAM=characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useSSL=false
      - NACOS_AUTH_ENABLE=true
      - NACOS_AUTH_IDENTITY_KEY=admin
      - NACOS_AUTH_IDENTITY_VALUE=admin
      - NACOS_AUTH_TOKEN=SecretKey012345678901234567890123456789012345678901234567890123456789
    ports:
      - "25001:8848" # 网页端口
      - "26001:9848" # gRpc 端口(必须是 网页端口+1000)
      - "26002:9849" # gRpc 端口(必须是 网页端口+1001)
```

## 集群模式(Cluster)

> 1. 和单机模式一样先要初始化 MySQL
> 2. 集群3个节点没有暴露端口，统一用的 nginx 转发
> 3. up 之后访问 : `http://locahost:8848/nacos`

### Nacos

#### `docker-compose.yml`

```yaml
services:
  nacos1:
    hostname: nacos1
    container_name: nacos1
    image: nacos/nacos-server:v2.5.1
    volumes:
      - ./cluster-logs/nacos1:/home/nacos/logs
      - /etc/localtime:/etc/localtime:ro  # 时间同步
    env_file:
      - ./env/nacos-hostname.env
    restart: always
    networks:
      - nacos_net

  nacos2:
    hostname: nacos2
    container_name: nacos2
    image: nacos/nacos-server:v2.5.1
    volumes:
      - ./cluster-logs/nacos2:/home/nacos/logs
      - /etc/localtime:/etc/localtime:ro  # 时间同步
    env_file:
      - ./env/nacos-hostname.env
    restart: always
    networks:
      - nacos_net

  nacos3:
    hostname: nacos3
    container_name: nacos3
    image: nacos/nacos-server:v2.5.1
    volumes:
      - ./cluster-logs/nacos3:/home/nacos/logs
      - /etc/localtime:/etc/localtime:ro  # 时间同步
    env_file:
      - ./env/nacos-hostname.env
    restart: always
    networks:
      - nacos_net

networks:
  nacos_net:
    external: true
```



#### `nacos-hostname.env`

```shell
PREFER_HOST_MODE=hostname
MODE=cluster
NACOS_SERVERS=nacos1:8848 nacos2:8848 nacos3:8848
SPRING_DATASOURCE_PLATFORM=mysql

MYSQL_SERVICE_HOST=mysql
MYSQL_SERVICE_DB_NAME=nacos
MYSQL_SERVICE_PORT=3306
MYSQL_SERVICE_USER=root
MYSQL_SERVICE_PASSWORD=123456
MYSQL_SERVICE_DB_PARAM=characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useSSL=false&allowPublicKeyRetrieval=true

NACOS_AUTH_ENABLE=true
NACOS_AUTH_IDENTITY_KEY=admin
NACOS_AUTH_IDENTITY_VALUE=admin
NACOS_AUTH_TOKEN=SecretKey012345678901234567890123456789012345678901234567890123456789
```



### Nginx(转发 Naocs)

#### `nginx/docker-compose.yml`

```yaml
services:
  nginx:
    image: nginx:latest
    container_name: nginx
    restart: always
    ports:
      - "8848:8848"
      - "9848:9848"
    environment:
      TZ: Asia/Shanghai
    volumes:
      - ./data/nginx.conf:/etc/nginx/nginx.conf
      - ./data/conf.d:/etc/nginx/conf.d
      - ./data/stream.d:/etc/nginx/stream.d
      - ./data/logs:/var/log/nginx
      - ./data/cert:/etc/nginx/cert
      - ./data/html:/usr/share/nginx/html
```



#### `nginx/data/nginx.conf`

```shell
user  nginx;
worker_processes  1;

events {
    worker_connections  1024;
}

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    # 全局 gzip 设置
    gzip on;
    gzip_min_length 1k;
    gzip_buffers 4 16k;
    gzip_http_version 1.1;
    gzip_comp_level 2;
    gzip_types text/plain application/x-javascript text/css application/xml application/javascript;
    gzip_proxied any;

    include /etc/nginx/conf.d/*.conf;  # 加载所有子配置
}

stream {
    include /etc/nginx/stream.d/*.conf;  # 从 conf.d 中加载 TCP 代理
}
```



#### `nginx/data/conf.d/nacos.conf`

```shell
# -------------------- 日志格式 --------------------
log_format nacos_log '$remote_addr - $remote_user [$time_local] '
                     '"$request" $status $body_bytes_sent '
                     '"$http_referer" "$http_user_agent" '
                     '-> $upstream_addr';

access_log /var/log/nginx/nacos_access.log nacos_log;

# -------------------- HTTP 接口代理（Spring Cloud Discovery） --------------------
upstream nacos_http {
    server nacos1:8848;
    server nacos2:8848;
    server nacos3:8848;
}

server {
    listen 8848;

    location /nacos/ {
        proxy_pass http://nacos_http/nacos/;

        proxy_http_version 1.1;
        proxy_set_header Connection "";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        send_timeout 60s;
    }
}
```



#### `nginx/data/stream.d/nacos-grpc.conf`

```shell
upstream nacos-grpc9848 {
    server nacos-standalone:9848 weight=1 max_fails=2 fail_timeout=5s;

    server nacos1:9848 weight=1 max_fails=2 fail_timeout=5s;
    server nacos2:9848 weight=1 max_fails=2 fail_timeout=5s;
    server nacos3:9848 weight=1 max_fails=2 fail_timeout=5s;
}

server {
    listen 9848;
    listen [::]:9848;
    proxy_connect_timeout 1s;
    proxy_timeout 3s;
    proxy_pass nacos-grpc9848;
}
```






# Nginx

```yaml
services:
  nginx:
    image: nginx:latest
    container_name: nginx
    restart: always
    # 需要代理多少个端口就写多少个
    ports:
      - "8080:80"
      - "8443:443"
      - "10001:10001"
      - "10002:10002"
    environment:
      TZ: Asia/Shanghai
    volumes:
      - ./data/nginx.conf:/etc/nginx/nginx.conf
      - ./data/conf.d:/etc/nginx/conf.d
      - ./data/stream.d:/etc/nginx/stream.d
      - ./data/logs:/var/log/nginx
      - ./data/cert:/etc/nginx/cert
      - ./data/html:/usr/share/nginx/html
```



# RabbitMQ

```yaml
services:
  rabbitmq:
    image: rabbitmq:4.0.8-management
    container_name: rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: root
      RABBITMQ_DEFAULT_PASS: 123456
    volumes:
      - ./data:/var/lib/rabbitmq
    restart: unless-stopped

volumes:
  rabbitmq-data:
```





# Anki-Sync-Server

> https://github.com/LuckyTurtleDev/docker-images/tree/main/dockerfiles/anki
>
> `mkdir ./data & chown -R 1000:1000 ./data`

```yaml
version: '3.3'
services:
    anki:
        image: ghcr.io/luckyturtledev/anki
        container_name: anki
        environment:
            - SYNC_USER1=username1:password1
            - SYNC_USER2=foobar:123456
            - RUST_LOG=info
        ports:
         - 0.0.0.0:10080:8080
        volumes:
            - './data:/data'
        user: 1000:1000
        restart: unless-stopped
```








# 附录



## 常用命令

------

### 启动 & 停止

| 命令                     | 说明                               |
| ------------------------ | ---------------------------------- |
| `docker compose up`      | 启动服务（前台运行）               |
| `docker compose up -d`   | 启动服务（后台运行）               |
| `docker compose down`    | 停止并移除容器、网络（保留数据卷） |
| `docker compose restart` | 重启服务（常用于配置修改后）       |
| `docker compose stop`    | 暂停容器（不删除）                 |
| `docker compose start`   | 启动已暂停的容器                   |

------

### 构建相关

| 命令                              | 说明                             |
| --------------------------------- | -------------------------------- |
| `docker compose build`            | 构建服务（针对 `build:` 的服务） |
| `docker compose build --no-cache` | 不使用缓存重新构建               |
| `docker compose pull`             | 拉取最新镜像                     |

------

### 查看运行状态

| 命令                                 | 说明                              |
| ------------------------------------ | --------------------------------- |
| `docker compose ps`                  | 查看容器状态                      |
| `docker compose top`                 | 查看容器内进程                    |
| `docker compose logs`                | 查看日志（默认所有服务）          |
| `docker compose logs -f`             | 实时查看日志（跟 `tail -f` 一样） |
| `docker compose logs -f servicename` | 实时查看指定服务日志              |

------

### 进入容器 / 执行命令

| 命令                                        | 说明                                     |
| ------------------------------------------- | ---------------------------------------- |
| `docker compose exec servicename bash`      | 进入某个容器的交互终端（前提容器运行中） |
| `docker compose run servicename bash`       | 临时运行容器执行命令（即使容器没在跑）   |
| `docker compose exec servicename <command>` | 在容器中执行指定命令                     |

------

### 清理资源

| 命令                     | 说明                           |
| ------------------------ | ------------------------------ |
| `docker compose down -v` | 停止容器并删除挂载的卷（谨慎） |
| `docker compose rm`      | 删除已停止的服务容器           |

------

### 调试辅助

| 命令                               | 说明                               |
| ---------------------------------- | ---------------------------------- |
| `docker compose config`            | 查看合并后的配置（含 `.env` 解析） |
| `docker compose config --services` | 查看所有服务名                     |
| `docker compose config --volumes`  | 查看所有定义的卷名                 |

------









