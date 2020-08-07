## docker-compose

docker-compose官网 https://docs.docker.com/compose/compose-file



#### 1. 安装dockercompose

```
[root@node2 ~]# curl -L \
https://get.daocloud.io/docker/compose/releases/download/1.25.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose && chmod +x /usr/local/bin/docker-compose

```

#### 1. docker镜像加速

```
[root@node2 ~]# mkdir /etc/docker
[root@node2 ~]# tee /etc/docker/daemon.json <<EOF
{
  "insecure-registries" : ["repo.flkj.local"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": [
     "https://w9tu3nny.mirror.aliyuncs.com",
     "http://hub-mirror.c.163.com/",
     "https://dockerhub.azk8s.cn",
     "https://docker.mirrors.ustc.edu.cn/",
     "https://0648c427d18026450f2dc01eb3f5fa00.mirror.swr.myhuaweicloud.com",
     "https://registry.docker-cn.com"     
     ],
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

配置网络(相当于kubernetes的命名空间)

```
[root@node2 ~]# docker network list
NETWORK ID          NAME                DRIVER              SCOPE
5716a9d46a09        bridge              bridge              local
8ca69d8e631f        host                host                local
1d94c8e08b2f        none                null                local
[root@node2 ~]# docker network create frontend
23ce6842b20dd8c91ba90703035c71b7297fefa4aa80b0656854924fc93838d4
[root@node2 ~]# docker network list
NETWORK ID          NAME                DRIVER              SCOPE
5716a9d46a09        bridge              bridge              local
23ce6842b20d        frontend            bridge              local
8ca69d8e631f        host                host                local
1d94c8e08b2f        none                null                local

#应用命名空间
 docker run -itd --name c3 --net backend centos
```



#### 3. docker-compose

`build`指定在使用Dockerfile构建镜像时的参数

```
version: "3.8"
services:
  webapp:
    build:
      context: ./dir
      dockerfile: Dockerfile-alternate
      args:
        buildno: 1
```

context : 构建镜像的工作目录
dockerfile: Dockerfile的文件名
args: 构建时传入的传输

***

`image` 指定容器的基础镜像, 格式为:

```
version: "3.8"
services:
  redis:
    image: redis:latest
```

***

`deploy`容器部署的个数

```
version: "3.8"
services:
  redis:
    image: redis:latest
    deploy:
      replicas: 2
```

容器复制

```
version: "3.8"
services:
  redis:
    image: redis:latest
    deploy:
      mode: replicated
      replicas: 2
```

资源限制  
limits: 最小限制
reservations: 保留资源

```
version: "3.8"
services:
  redis:
    image: redis:alpine
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 50M
        reservations:
          cpus: '0.25'
          memory: 20M
```

重启策略 restart_policy
condition: 重启的条件 ,选项为 none, on-failure, 默认 any
delay: 重新启动尝试之间等待的[时间](https://docs.docker.com/compose/compose-file/#specifying-durations)，指定为 [持续时间](https://docs.docker.com/compose/compose-file/#specifying-durations)（默认值：0）
max_attempts: 放弃尝试重新启动容器的次数, 默认 `一直重启` 
window: 决定重新启动是否成功之前要等待的[时间](https://docs.docker.com/compose/compose-file/#specifying-durations)，指定为[持续时间](https://docs.docker.com/compose/compose-file/#specifying-durations)（默认值：立即决定）

```
version: "3.8"`
services:
  redis:
    image: redis:alpine
    deploy:
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
```



****

`configs` 容器的配置文件

```
version: "3.8"
services:
  redis:
    image: redis:latest
    deploy:
      replicas: 1
    configs:
      - source: my_config
        target: /redis_config
        uid: '103'
        gid: '103'
        mode: 0440
configs:
  my_config:
    file: ./my_config.txt
  my_other_config:
    external: true
```

***

`container_name` 指定容器的名称, 如果指定了名称,则replicas不能超过1

```
container_name: my-web-container
```

***

`volumes` 卷挂载

```
version: "3.8"
services:
  mysql:
    image: mysql
    volumes:
       - db-data:/var/lib/mysql/data
```

`networks` 指定网络的命名空间, 示例中树妖提前创建frontend网络命名空间

```
version: "3.8"
services:
  mysql:
    image: mysql
    volumes:
       - db-data:/var/lib/mysql/data
    networks:
      - frontend    
```

`DNS` 自定义DNS服务器, 可以是单个值或者列表.
`dns: 114.114.114.114`

```
dns:
  - 114.114.114.114
  - 8.8.8.8
```

`entrypoint` 覆盖容器的entrypoint

```
entrypoint: /code/entrypoint.sh
```

```
entrypoint: ["php", "-d", "memory_limit=-1", "vendor/bin/phpunit"]
```

***

`env_file` 环境变量配置文件
使用指定了Compose文件`docker-compose -f FILE`，则in `env_file`的路径 相对于该文件所在的目录

```
env_file: .env
```

```
env_file:
  - ./common.env
  - ./apps/web.env
  - /opt/runtime_opts.env
```

***

环境变量 
可以使用数组或字典。任何布尔值（true，false，yes，no）都需要用引号引起来，以确保YML解析器不会将其转换为True或False

```
environment:
  RACK_ENV: development
  SHOW: 'true'
  SESSION_SECRET:
```

```
environment:
  - RACK_ENV=development
  - SHOW=true
  - SESSION_SECRET
```

***

`expose` 
公开端口而不将其发布到主机上-只有链接的服务才能访问它们。只能指定内部端口

```
expose:
  - "3000"
  - "8000"
```

***

主机名映射

```
extra_hosts:
  - "somehost:162.242.195.82"
  - "otherhost:50.31.209.229"
```

需要在/etc/hosts文件中增加主机名解析

***

`healthcheck`
配置运行的检查以确定该服务的容器是否“健康”

```
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"]
  interval: 1m30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

***

`init` 
初始化, 在容器内运行一个初始化程序，以转发信号并获取进程。设置此选项可以`true`为服务启用此功能

```
version: "3.8"
services:
  web:
    image: alpine:latest
    init: true
```

***

`links`
链接容器

```
web:
  links:
    - "db"
    - "db:database"
    - "redis"
```

***

`日志`

```
logging:
  driver: syslog
  options:
    syslog-address: "tcp://192.168.0.42:123"
```

driver: 默认值为json-file
`driver: "json-file"`
默认驱动程序[json-file](https://docs.docker.com/config/containers/logging/json-file/)，具有用于限制所存储日志数量的选项。为此，请使用键值对以获取最大存储大小和最大文件数：

```
version: "3.8"
services:
  some-service:
    image: some-service
    logging:
      driver: "json-file"
      options:
        max-size: "200k"
        max-file: "10"
```

`network_mode`
支持类型

```
network_mode: "bridge"
network_mode: "host"
network_mode: "none"
network_mode: "service:[service name]"
network_mode: "container:[container name/id]"
```

***

`网络`
定义容器要加入的网络

```
services:
  some-service:
    networks:
     - some-network
     - other-network
```

定义容器要加入的别名

```
services:
  some-service:
    networks:
      some-network:
        aliases:
          - alias1
          - alias3
      other-network:
        aliases:
          - alias2
```

示例:

```
version: "3.8"
services:
  web:
    image: "nginx:alpine"
    networks:
      - new

  worker:
    image: "my-worker-image:latest"
    networks:
      - legacy

  db:
    image: mysql
    networks:
      new:
        aliases:
          - database
      legacy:
        aliases:
          - mysql

networks:
  new:
  legacy:
```

***

`ports`
暴露端口, 端口映射模式network_mode: host , 映射出的端口需要大于60

```
ports:
  - "3000"
  - "3000-3005"
  - "8000:8000"
```

长语法:

```
ports:
  - target: 80
    published: 8080
    protocol: tcp
    mode: host
```

***

`restart`

默认的重启策略是no , 当指定always时,容器总是启动,当 容器服务是on-failure时,重启容器
unless-stopped 总是重启容器,除非容器停止

```
restart: "no"
restart: always
restart: on-failure
restart: unless-stopped
```

***

`sysctl`
容器中设置内核参数

```
sysctls:
  - net.core.somaxconn=1024
  - net.ipv4.tcp_syncookies=0
```

***

`tmpfs`
临时文件,可以是单个值或者是列表

```
tmpfs: /run
```

```
tmpfs:
  - /run
  - /tmp
```

***

`volume`

```
version: "3.8"
services:
  web:
    image: nginx:alpine
    volumes:
      - type: volume
        source: mydata
        target: /data
        volume:
          nocopy: true
      - type: bind
        source: ./static
        target: /opt/app/static

  db:
    image: postgres:latest
    volumes:
      - "/var/run/postgres/postgres.sock:/var/run/postgres/postgres.sock"
      - "dbdata:/var/lib/postgresql/data"

volumes:
  mydata:
  dbdata:
```

短语法

```
volumes:
  # Just specify a path and let the Engine create a volume
  - /var/lib/mysql

  # Specify an absolute path mapping
  - /opt/data:/var/lib/mysql

  # Path on the host, relative to the Compose file
  - ./cache:/tmp/cache

  # Named volume
  - datavolume:/var/lib/mysql
```

长语法

```
version: "3.8"
services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - type: volume
        source: mydata
        target: /data
        volume:
          nocopy: true
      - type: bind
        source: ./static
        target: /opt/app/static

networks:
  webnet:

volumes:
  mydata:
```

***

`domainname, hostname, ipc, mac_address, privileged, read_only, shm_size, stdin_open, tty, user, working_dir`
其中每个都是一个值，类似于其 [docker run](https://docs.docker.com/engine/reference/run/)对应项。请注意，这`mac_address`是一个旧选项。

```

user: postgresql
working_dir: /code

domainname: foo.com
hostname: foo
ipc: host
mac_address: 02:42:ac:11:65:43

privileged: true


read_only: true
shm_size: 64M
stdin_open: true
tty: true
```

卷名

```
version: "3.8"
volumes:
  data:
    name: my-app-data
```

***

变量替换

```
db:
  image: "postgres:${POSTGRES_VERSION}"
```





## 配置示例

```

version: "3.8"
services:

  redis:
    image: redis:alpine
    ports:
      - "6379"
    networks:
      - frontend
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure

  db:
    image: postgres:9.4
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend
    deploy:
      placement:
        max_replicas_per_node: 1
        constraints:
          - "node.role==manager"

  vote:
    image: dockersamples/examplevotingapp_vote:before
    ports:
      - "5000:80"
    networks:
      - frontend
    depends_on:
      - redis
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
      restart_policy:
        condition: on-failure

  result:
    image: dockersamples/examplevotingapp_result:before
    ports:
      - "5001:80"
    networks:
      - backend
    depends_on:
      - db
    deploy:
      replicas: 1
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure

  worker:
    image: dockersamples/examplevotingapp_worker
    networks:
      - frontend
      - backend
    deploy:
      mode: replicated
      replicas: 1
      labels: [APP=VOTING]
      restart_policy:
        condition: on-failure
        delay: 10s
        max_attempts: 3
        window: 120s
      placement:
        constraints:
          - "node.role==manager"

  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    stop_grace_period: 1m30s
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints:
          - "node.role==manager"

networks:
  frontend:
  backend:

volumes:
  db-data:
```

























