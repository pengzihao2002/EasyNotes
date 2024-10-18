## 镜像（image）
---
标准写法：`IMAGE:TAG` 略写：`IMAGE`

检索：`docker search`
下载：`docker pull`
列表：`docker images`
删除：`docker rmi IMAGE:TAG(VERSION)`

提交：`docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]`
  Options：
      `-a`， `--author` string 作者
      `-c`， `--change` list 应用Dockerfile文件的指令创建镜像
      `-m`， `--message` string 提交信息类似Git

保存：`docker save [-o] IMAGE` 打包成一个文件（一般为压缩包） => `-o` 后加文件名
加载：`docker load`

登录：`docker login`
命名：`docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]` `TARGET_IMAGE` 完整写法 => `UserName/ImageName`
推送：`docker push IMAGE`


## 容器（container）
---
创建容器先思考以下问题：
- 网络：是否需要暴露一些端口让外界访问，使用`-d`或`--network`
- 存储：是否需要挂载配置文件到外部，进行数据持久化，使用`-v`进行目录挂载或卷映射
- 环境变量：是否需要在容器启动时传入环境变量进行初始配置，使用`-e`
- 具体需要参考[DockerHub官网](https://hub.docker.com/)



运行：`docker run [OPTIONS] IMAGE`
  Options:
    `-d` detach 分离 即后台运行
    `--name 容器名称` 如果不指定名称，系统自动命名
    `-p 服务器主机的端口：容器的端口` port 端口 访问服务器主机的端口等于访问容器的端口，即端口映射
    `--network 自定义网络` 需提前创建自定义网络
    `--restart always` 服务器开机，容器自动启动
    `-e` 环境变量，具体参考[DockerHub官网](https://hub.docker.com/)
    `-v` 存储挂载
    
查看：`docker ps [-a]` 查看所有 => `-a`
停止：`docker stop ID/NAMES`
启动：`docker start ID/NAMES`
重启:`docker restart ID/NAMES`
状态：`docker stats ID/NAMES`
日志：`docker logs ID/NAMES`
进入：`docker exec -it ID/NAMES /bin/bash` 交互模式 => `-it`， 进入控制台 => `/bin/bash`可简写为`bash`
删除：`docker rm [-f] ID/NAMES` 强制删除 => `-f`
检查：`docker [container] inspect ID/NAMES` 用于查看容器状况 


## 存储（volume）
---
目录挂载：`docker run -v 外部目录:容器目录` 以nginx为例： `-v /app/nginx_html:/usr/share/nginx/html`
卷映射：`docker run -v 卷名:容器目录` 以nginx为例： `-v nginx:/etc/nginx` 卷名不以`./`或`/`开头
！可同时进行两种存储方式，二者都可实现数据同步

区别：
- 目录挂载：直接将主机的某个目录映射到容器内，数据实时同步，外部修改，内部同步修改，使用方法简单。
  适用于需要访问特定文件、特定数据共享的场景。
- 卷映射：卷由Docker管理，不依赖于主机文件系统，数据可在多个容器间共享，并且删除容器时卷中数据不会自动删除。
  适合在容器间共享数据、数据持久化、数据需要复用的场景
- 目录挂载 => 外挂内，外部同步到内部，如果此时外部目录文件为空，则内部文件也为空，而且该文件是容器启动所必须的，那么会报错。
- 卷映射 => 内挂外，内部同步到外部，此时哪怕外部目录为空，也不影响容器启动。

查看卷：`docker volume ls`
创建卷：`docker volume create VOLUME NAME`


## 网络（network）
---
容器IP访问方法：
- 进入容器控制台：`docker exec -it ID/NAMES bash` 
- 访问某个容器：`curl http://容器IP:容器端口`
- 缺点：容器IP地址可能由于容器删除、迁移、重启等原因发生变化，属于不太稳定的访问方式

自定义网络访问方法：
- 创建自定义网络：`docker network create NETWORK NAME` 
- 使用自定义网络：`docker run --network NETWORK NAME`
- 进入容器控制台：`docker exec -it ID/NAMES bash`
- 访问某个容器：`curl http://容器名或ID` 
- 优点：使用容器名访问容器，更加方便，同时docker会自动解析容器IP地址，属于稳定的访问方式

使用Redis主从集群：
- 参考文档：[bitnami/redis](https://hub.docker.com/r/bitnami/redis)
- 主服务器redis1 => 进行数据的写入、修改、删除
- 从服务器redis2 => 同步、备份主服务器的数据，进行数据的读取
- 优点：负载均衡，数据备份，主备服务切换
- 实战演示：
```
# 创建主服务器redis1
docker run -d -p 6379:6379 
-v /app/red1:/bitnami/redis/data 
-e REDIS_REPLICATION_MODE=master 
-e REDIS_PASSWORD=123456 
-- network mynet --name redis1 
bitnami/redis


# 此时会报错，提示无权限写入文件，/app/red1只有root用户是可读可写，其余用户只能读
cd /app
ls    # 显示目录列表
ll    # ls-l 的别名，以较长格式列出信息
chmod -R 777 red1    # 所有用户可读可写
mkdir red2    # 建议创建服务器前，先创建目录，再修改权限
chmod -R 777 red2


# 创建从服务器redis2
docker run -d -p 6380:6379 
-v /app/red2:/bitnami/redis/data 
-e REDIS_REPLICATION_MODE=slave 
-e REDIS_MASTER_HOST=redis1 
-e REDIS_MASTER_PORT_NUMBER=6379 
-e REDIS_MASTER_PASSWORD=123456 
-e REDIS_PASSWORD=123456 
--network mynet --name redis2 
bitnami/redis
```


## [Docker Compose](https://docs.docker.com/reference/compose-file/)
---
Docker Compose => 用于批量管理容器的工具，可以快速启动指定容器并完成匹配
文件名：`compose.yaml`

上线：`docker compose up -d` 用于第一次创建并启动应用
下线：`docker compose down`
启动：`docker compose start x1 x2 x3` x1、x2、x3指的是compose.yaml文件里已经启动过的容器
停止：`docker compose stop x1 x2` 
扩容：`docker compose scale x1=3` 创建3个x1容器

`compose.yaml`文件
包含顶级元素（top-level elements）
- `name` ：名字
- `services` ：服务
- `networks` ：网络
- `volumes` ：卷
- `configs` ：配置
- `secrets` ：密钥
```
# compose.yaml
name: myblog
services:
  mysql:
    container_name: mysql
    image: mysql:8.0
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=123456
      - MYSQL_DATABASE=wordpress
    volumes:
      - mysql-data:/var/lib/mysql
      - /app/myconf:/etc/mysql/conf.d
    restart: always
    networks:
      - blog

  wordpress:
    image: wordpress
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: mysql
      WORDPRESS_DB_USER: root
      WORDPRESS_DB_PASSWORD: 123456
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wordpress:/var/www/html
    restart: always
    networks:
      - blog
    depends_on:
      - mysql

volumes:
  mysql-data:
  wordpress:

networks:
  blog:
```