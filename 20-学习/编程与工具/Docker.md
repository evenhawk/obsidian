---
title: Docker 学习
tags:
  - docker
  - 容器
  - 工具
status: maintained
updated: 2026-11-15
---

# Docker 学习

`sudo` : superuser do

授权用户临时以其他用户身份（默认 root）执行单条命令，输入**自己账号密码**而非 root 密码。

*（本文命令省略了 `sudo`，如果你的用户不在 docker 组里，执行时自行加上。）*

## 运行容器

以运行 `nginx` 举例：

```bash
docker run nginx
```

但这样会 ==占用一个窗口==，所以更常见的启动方式是

```bash
docker run -d nginx
```

查看正在运行的容器：

`ps` : process status

```bash
docker ps
```

### 删除容器

```bash
docker rm <container id>
```

如果容器正在运行，需要加上 `-f` 表示**强制**（force）删除

```bash
docker rm -f <container id>
```

删除容器是 `rm`,删除镜像是 `rmi`

常用docker ps 选项表格:

| 命令 | 作用 |
| --- | --- |
| `docker ps` | 只列出正在运行的容器（最常用） |
| `docker ps -a` | 列出所有容器：运行中 + 已停止 |
| `docker ps -q` | 只输出容器 ID（只看编号，无其他信息） |
| `docker ps --no-trunc` | 完整显示容器 ID，不截断 |
| `docker ps --filter "status=running"` | 过滤，只保留运行状态容器 |

PS输出字段含义：

| 字段 | 说明 |
| --- | --- |
| CONTAINER ID | 容器短 ID |
| IMAGE | 使用的镜像名称 |
| COMMAND | 容器启动执行的命令 |
| CREATED | 容器创建多久 |
| STATUS | 容器状态（Up = 正在运行；Exited = 已停止） |
| PORTS | 端口映射 |
| NAMES | 容器名称（--name 指定） |

### 端口映射

端口映射参数 `-p`

在docker中，容器和宿主机的网络是隔离的，并不能从宿主机直接访问容器内网络

如果在容器内启动一个服务，假设端口是80，在浏览器并不能访问容器的80端口

这时候需要添加一个启动参数，把容器内的端口和主机的端口做一个端口映射,比如这样 `-p 80:80`

这时候浏览器就可以正常访问了

```bash
docker run -p 80:80 nginx
```

## 挂载卷

容器里的数据默认存在容器内部，容器删除，数据跟着消失；挂载卷就是把宿主机的一块存储空间 “接入” 容器，实现==数据持久化==。

与 `-p` 绑定端口类似的参数是 `-v` (volume)

`-p` 是把宿主机与容器的**端口**进行绑定，而 `-v` 是把宿主机与容器的**文件目录**进行绑定

绑定后宿主机的文件夹和容器的文件夹会相互影响

### 绑定挂载

直接把宿主机目录写在 `docker run` 命令中，这种方式叫做绑定挂载

```bash
docker run -p 80:80 -v <宿主机目录>:<容器目录> nginx
```

### 命名卷挂载

让docker自动创建一个存储空间，自己命名一个名字的挂载方式叫做命名卷挂载

先创建一个挂载卷

```bash
docker volume create <挂载卷名>
```

再运行nginx

```bash
docker run -p 80:80 -v <挂载卷名>:<容器目录> nginx
```

如果要检查宿主机的真实目录

```bash
docker volume inspect <挂载卷名>
```

列出所有创造过的卷

```bash
docker volume list
```

删除指定卷

```bash
docker volume rm <挂载卷名>
```

删除所有没有任何容器正在使用的卷

```bash
docker volume prune -a
```

## docker run 其他参数

### 参数 -e

作用：给容器设置环境变量

环境变量 = 容器启动时注入的配置参数，容器内部程序读取这个变量来工作。

一次可以传递多个环境变量

示例，传递MySQL容器的环境变量

```bash
docker run -d \
  --name mysql8 \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD="Root@123456" \
  -e MYSQL_DATABASE="test_db" \
  -e MYSQL_USER="appuser" \
  -e MYSQL_PASSWORD="app123456" \
  -e TZ=Asia/Shanghai \
  -v mysql_data:/var/lib/mysql \
  mysql:8.0
```

### 参数 --name

给容器一个指定的名字

示例：

```bash
docker run -d --name <NAMES> nginx
```

==名字在宿主机上必须是唯一的==

这里面的 **NAMES** 和 **CONTAINER ID** 是等价的，使用ID的地方都可以用 **NAMES** 来代替

### 参数 -it 和 --rm

| 选项 | 全称 | 作用 |
| --- | --- | --- |
| `-i` | `--interactive` | 保持标准输入打开，持续接收键盘输入 |
| `-t` | `--tty` | 分配伪终端，模拟终端界面 |
| `-it` | `-i -t` | 组合使用：交互式终端，可以进入容器内部操作 |
| `--rm` | （无） | 容器停止退出后，自动删除这个容器（不保留容器实例） |

必须==组合使用== `-i -t`，简写 `-it`，单独 `-i` 或者单独 `-t` 基本很少用。

适用场景：需要进入容器、在容器里面敲命令。

`--rm`：容器一旦停止，自动删掉容器本身。

组合使用用来调试容器

```bash
docker run -it --rm <镜像名>
```

### 参数 --restart

用来配置容器在停止时的**重启策略**

常用的有 `--restart always`

只要容器停止了就立刻重启，包含内部崩溃和外部宿主机断电等场景

```bash
docker run -d --restart always nginx
```

如果把 `always` 换成 `unless-stopped`

则手动停止的容器就不会尝试重启

```bash
docker run -d --restart unless-stopped nginx
```

## 调试容器

### 容器的启停与检查

每次使用 `docker run` 命令都是创建一个==新==容器

如果想对原有容器进行一个启停，可以使用下面的两个命令

```bash
docker start <容器ID>/<名字>
docker stop <容器ID>/<名字>
```

查看容器的参数使用 `inspect`

```bash
docker inspect <容器ID>
```

输出内容很长，*建议丢给AI分析*

### 创建容器

使用 `run` 会创建并运行，如果要先创建不运行的话使用 `create` 新建一个容器

```bash
docker create <镜像名>
```

创建之后再 `start` 就可以运行了

### 日志

要查看日志可以使用 `logs`

```bash
docker logs <容器名>/<容器ID>
```

## Docker 技术原理

Docker 利用了Linux内核的两大原生功能实现容器化

1. **Cgroups** 用来限制和隔离进程的**资源使用**，可以为每个容器设定CPU,内存,网络带宽等资源的使用上限，确保一个容器的资源消耗不会影响宿主机或其他正在运行的容器。
2. **Namespaces** 用来隔离进程的**资源视图**，让容器只能看见自己内部的进程ID，网络资源和文件目录，而看不到宿主机的。

所以容器本质上还是一个==特殊的进程==，不过当进入容器内部后容器内部看起来就像是一个独立的操作系统，每个docker 容器都是一个独立的运行环境，都表现的像一个独立的Linux系统

## 进入容器

如果需要进入容器查看进程

容器ID或容器名后面加你想要执行的Linux命令

```bash
docker exec <容器ID>/<容器名> ps -ef
```

进入容器并获得一个交互式的命令行环境

```bash
docker exec -it <容器ID> /bin/sh
```

有些容器也可以使用

```bash
docker exec -it <容器ID> /bin/bash
```

进入容器后查看Linux版本信息，根据版本信息可以查询使用的下载工具是什么。

```bash
cat /etc/os-release
```

## Dockerfile 构造镜像

1. Dockerfile 文件名字**固定大写** Dockerfile，没有后缀
2. 每一行是一条指令，**大写关键字**，参数小写
3. **分层构建**：每条指令生成一层镜像缓存，构建时会复用缓存加快速度
4. 构建命令：`docker build`

构造镜像的Dockerfile如下

```dockerfile
# 基础镜像
FROM ubuntu:22.04

# 设置环境变量
ENV TZ=Asia/Shanghai

# 设置工作目录
WORKDIR /opt/demo

# 构建时安装工具
RUN apt update && apt install -y curl

# 复制宿主机文件到镜像
COPY test.txt ./

# 容器启动默认执行bash
CMD ["/bin/bash"]
```

构建镜像的命令

```bash
# . 代表上下文：当前目录，Dockerfile就在当前目录
docker build -t my-demo:v1 .
```

| 指令 | 作用 | 示例 |
| --- | --- | --- |
| `FROM` | 必须第一条，指定基础镜像 | `FROM ubuntu:22.04` |
| `RUN` | 构建镜像时执行命令（安装软件） | `RUN apt update && apt install -y curl` |
| `COPY` | 把宿主机文件复制到镜像内 | `COPY app.py /opt/` |
| `ADD` | 复制，额外支持自动解压压缩包 | `ADD demo.tar.gz /opt/` |
| `WORKDIR` | 设置镜像内部工作目录（相当于 cd） | `WORKDIR /opt/myapp` |
| `ENV` | 设置镜像内置环境变量（等价 -e） | `ENV TZ=Asia/Shanghai` |
| `EXPOSE` | 声明容器打算暴露哪个端口，不会自动 -p 映射 | `EXPOSE 8080` |
| `CMD` | 容器启动时默认执行命令（只能一条） | `CMD ["python3","app.py"]` |
| `ENTRYPOINT` | 容器入口程序，和 CMD 配合 | `ENTRYPOINT ["/bin/bash"]` |

## docker 网络

### 桥接模式

Docker 网络默认是 **bridge 桥接模式**

所有的容器默认都连接到这个网络，每个容器都分配了一个内部IP地址

在内部子网中，容器可以通过内部IP地址相互访问

但容器网络与宿主机的网络是隔离的

可以使用 `docker network create` 创建出子网

可以指定容器加入不同的子网，同一个子网可以相互通信，跨子网不可以通信

同一个子网的容器可以==使用名字互相访问==

### 自动分配IP的子网

创建一个叫 `network1` 的子网,自动分配IP地址

```bash
docker network create network1
```

运行并加入 `network1` 的指令

```bash
docker run -d --name mysql8 \
  --network network1 \
  -e MYSQL_ROOT_PASSWORD=Root@123456 \
  mysql:8.0
```

### 指定静态IP的子网

创建一个叫 `network1` 的子网，指定静态IP地址

```bash
docker network create \
  --subnet=172.21.0.0/24 \
  --gateway=172.21.0.1 \
  network1
```

运行并加入 `network1`

```bash
docker run -d --name mysql8 \
  --network network1 \
  --ip=172.21.0.10 \
  -e MYSQL_ROOT_PASSWORD=Root@123456 \
  mysql:8.0
```

在正在运行的容器中加入子网

```bash
docker network connect network1 <容器ID>
```

脱离子网

```bash
docker network disconnect network1 <容器ID>
```

检验是否加入成功

```bash
docker network inspect network1
```

看输出有没有对应容器名

### Host模式

Docker容器直接共享宿主机的网络，容器直接使用主机的IP地址

而且==无需 `-p` 进行端口映射==，容器内的服务直接运行在宿主机的端口上

通过宿主机的IP和端口就能访问到容器

创建Host模式的容器

```bash
docker run -d --network host nginx
```

### None模式

不联网

展示出所有的Docker网络

```bash
docker network list
```

删除子网

```bash
docker network rm <网络ID>
```

在调试模式exec进入容器后查看ip

```bash
ip addr show
```

## Docker Compose

用一个 yaml 文件一次性编排多个容器，一键启停一组服务，不用敲很长一串 `docker run` 命令

可以简单的把docker compose文件理解成一个或多个的Docker run 命令，按照特定的格式列到了一个文件里

普通创建

```bash
# 1.先创建自定义网络
docker network create --subnet=172.21.0.0/24 --gateway=172.21.0.1 network1

# 启动mysql容器
docker run -d \
  --name mysql8 \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=Root@123456 \
  -e TZ=Asia/Shanghai \
  -v mysql_data:/var/lib/mysql \
  --network network1 \
  mysql:8.0

# 启动ubuntu测试容器
docker run -d \
  --name ubuntu1 \
  -t \
  --network network1 \
  ubuntu:22.04
```

在compose中==不需要创建子网==，在同一个compose文件中的所有容器会自动放在一个子网中。

compose创建

```yaml
services:
  mysql:
    image: mysql:8.0
    container_name: mysql8
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: Root@123456
      TZ: Asia/Shanghai
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - network1

  ubuntu-test:
    image: ubuntu:22.04
    container_name: ubuntu1
    tty: true
    networks:
      - network1

networks:
  network1:
    driver: bridge
    ipam:
      config:
        - subnet: 172.21.0.0/24
          gateway: 172.21.0.1

volumes:
  mysql_data:
```

创建流程

创建并写入 `docker-compose.yaml`

```bash
vi docker-compose.yaml
```

然后写入内容

启动文件定义所有的容器

```bash
docker compose up -d
```

停止并且删除容器

```bash
docker compose down
```

想停止但不想删除容器

```bash
docker compose stop
```

启动容器

```bash
docker compose start
```

指定compose文件

```bash
docker compose -f <文件名> up -d
```

核心对比表格

| 项目 | 多条 docker run | docker compose yaml |
| --- | --- | --- |
| 启动 | 复制多条长命令手动执行 | 一条 `up -d` 一键整套启动 |
| 网络 / 卷 | 需要手动提前 `network create`、手动管理 | yml 内部声明，自动创建网络、命名卷 |
| 服务访问 | 记住容器名字互相访问 | 直接用服务名（`mysql`）访问，不用记容器名 |
| 修改配置 | 改完必须手动删容器，再复制整段 run 重建 | 修改 yml 后，`compose up -d` 自动重建变更服务 |
| 停止清理 | 要分别 stop、rm 容器、rm 网络、rm 卷 | 一条 `down` 统一清理整套资源 |
| 适用场景 | 临时测试单个容器 | 多容器项目、开发环境、整套服务编排 |

重要等价映射回顾

| docker run 参数 | compose yaml 配置项 |
| --- | --- |
| `--name mysql8` | `container_name: mysql8` |
| `-p 3306:3306` | `ports: ["3306:3306"]` |
| `-e KEY=VAL` | `environment:` |
| `-v xxx:/path` | `volumes:` |
| `--network network1` | `networks: [network1]` |
| `-t` | `tty: true` |
