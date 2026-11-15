---
title: Docker 学习笔记
tags:
  - docker
  - 容器
  - 工具
status: maintained
updated: 2026-11-15
---

# Docker 学习笔记

> [!info] 命令约定
> 本文命令统一省略了 `sudo`。若当前用户未加入 `docker` 用户组，执行时请在命令前自行加上 `sudo`。

## 前置：命令中的 sudo

`sudo`（superuser do）：授权普通用户临时以其他用户身份（默认 root）执行单条命令，验证的是**自己账号的密码**，而不是 root 密码。

## 一、运行容器

以运行 nginx 为例：

```bash
docker run nginx
```

这样会在前台运行并占用当前终端窗口，所以更常见的启动方式是加 `-d`（detach，后台运行）：

```bash
docker run -d nginx
```

查看正在运行的容器（`ps` = process status）：

```bash
docker ps
```

### 常用 docker ps 选项

| 命令 | 作用 |
| --- | --- |
| `docker ps` | 只列出正在运行的容器（最常用） |
| `docker ps -a` | 列出所有容器：运行中 + 已停止 |
| `docker ps -q` | 只输出容器 ID（只看编号，无其他信息） |
| `docker ps --no-trunc` | 不截断输出，完整显示容器 ID、COMMAND 等长字段 |
| `docker ps --filter "status=running"` | 按条件过滤，只保留运行中的容器 |

### docker ps 输出字段含义

| 字段 | 说明 |
| --- | --- |
| CONTAINER ID | 容器短 ID |
| IMAGE | 使用的镜像名称 |
| COMMAND | 容器启动时执行的命令 |
| CREATED | 容器创建多久 |
| STATUS | 容器状态（Up = 正在运行；Exited = 已停止） |
| PORTS | 端口映射 |
| NAMES | 容器名称（`--name` 指定；未指定时由 Docker 随机生成） |

### 删除容器

```bash
docker rm <容器ID>
```

如果容器正在运行，需要加 `-f` 表示强制（force）删除：

```bash
docker rm -f <容器ID>
```

> [!note] 容器与镜像的删除命令不同
> 删除容器是 `rm`，删除镜像是 `rmi`。

### 端口映射 `-p`

- Docker 中容器与宿主机的网络是隔离的，不能从宿主机直接访问容器内的网络。
- 假设容器内启动了一个服务、端口是 80，浏览器无法直接访问容器的 80 端口。
- 这时需要加启动参数，把容器内端口和宿主机端口做映射：`-p <宿主机端口>:<容器端口>`。

```bash
docker run -d -p 80:80 nginx
```

> [!warning] 顺序不能反
> 冒号**前**是宿主机端口，**后**是容器端口。`-p 8080:80` 表示"访问宿主机的 8080 端口 → 转发到容器的 80 端口"。

映射后浏览器访问宿主机的 80 端口，即可正常访问容器内的服务。

## 二、挂载卷（数据持久化）

容器里的数据默认存在容器内部，容器一删除数据就跟着消失。挂载卷就是把宿主机的一块存储空间"接入"容器，实现数据持久化。

与 `-p`（绑定端口）类似的参数是 `-v`（volume）：

- `-p` 是把宿主机与容器的**端口**进行绑定；
- `-v` 是把宿主机与容器的**文件目录**进行绑定。

绑定后，宿主机目录和容器目录的内容会相互影响。

### 绑定挂载（bind mount）

直接把宿主机目录写在 `docker run` 命令中，这种方式叫绑定挂载：

```bash
docker run -d -p 80:80 -v <宿主机目录>:<容器目录> nginx
```

例如把宿主机的网站目录挂到 nginx 的网页根目录：

```bash
docker run -d -p 80:80 -v /data/nginx/html:/usr/share/nginx/html nginx
```

> [!warning] 宿主机一侧必须写绝对路径
> 如果写成相对路径（如 `./html:/usr/share/nginx/html`），Docker 会把它当成一个**命名卷的名字**，而不是宿主机目录。

### 命名卷挂载（named volume）

让 Docker 自动创建一块存储空间、并自己给它起名字，这种方式叫命名卷挂载。

先创建一个挂载卷：

```bash
docker volume create <卷名>
```

再运行 nginx 并挂载它：

```bash
docker run -d -p 80:80 -v <卷名>:<容器目录> nginx
```

检查卷在宿主机上的真实目录（看输出的 `Mountpoint` 字段）：

```bash
docker volume inspect <卷名>
```

列出所有创建过的卷：

```bash
docker volume list
```

删除指定卷：

```bash
docker volume rm <卷名>
```

删除所有没有任何容器正在使用的卷：

```bash
docker volume prune -a
```

> [!note]
> `-a`（`--all`）是较新版本 Docker 才有的参数；不加 `-a` 时，`prune` 只清理匿名卷。

## 三、docker run 其他常用参数

### `-e`：环境变量

作用：给容器设置环境变量。

环境变量 = 容器启动时注入的配置参数，容器内部程序读取这些变量来工作。一次可以传递多个环境变量。

示例，传递 MySQL 容器的环境变量：

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

### `--name`：容器名

给容器指定一个名字：

```bash
docker run -d --name <容器名> nginx
```

- 名字在宿主机上必须是唯一的。
- 容器名和容器 ID 是等价的：凡是能用 ID 的地方，都可以用容器名代替。

### `-it` 与 `--rm`

| 选项 | 全称 | 作用 |
| --- | --- | --- |
| `-i` | `--interactive` | 保持标准输入打开，持续接收键盘输入 |
| `-t` | `--tty` | 分配伪终端，模拟终端界面 |
| `-it` | `-i` + `-t` | 组合使用：交互式终端，可以进入容器内部操作 |
| `--rm` | automatically remove（Docker 未提供对应长参数） | 容器停止退出后，自动删除这个容器（不保留容器实例） |

- `-i` 和 `-t` 一般组合使用，简写为 `-it`；单独使用很少见。
- 适用场景：需要进入容器、在容器里面敲命令。
- `--rm`：容器一旦停止，自动删掉容器本身。
- 两者组合常用于一次性调试容器：

```bash
docker run -it --rm ubuntu:22.04 /bin/bash
```

> [!warning] `docker run` 后面跟的必须是镜像名
> 不是容器名。`docker run` 的语义是"从镜像创建新容器"；要操作已有容器请用 `docker start` / `docker exec`。

### `--restart`：重启策略

用来配置容器停止时的重启策略：

```bash
docker run -d --restart always nginx
```

| 策略 | 行为 |
| --- | --- |
| `no` | 默认值，不自动重启 |
| `always` | 只要容器停止就重启；宿主机重启、Docker 守护进程重启后也会自动拉起 |
| `unless-stopped` | 与 `always` 基本相同，但**被手动 stop 过的容器不会再被拉起** |
| `on-failure[:次数]` | 仅当容器以非 0 状态码退出时才重启，可限制重试次数 |

把 `always` 换成 `unless-stopped`，手动停止的容器就不会再尝试重启：

```bash
docker run -d --restart unless-stopped nginx
```

> [!note]
> 即使配置了 `always`，被 `docker stop` 手动停掉的容器也不会立刻重启；只有在 Docker 守护进程重启或宿主机重启后才会被拉起。`unless-stopped` 则连这些场景也不拉起。

## 四、容器的启停与调试

每次使用 `docker run` 命令都是创建一个**新**容器。如果想对原有容器进行启停：

```bash
docker start <容器ID或名字>
docker stop <容器ID或名字>
```

### 查看容器详细参数

```bash
docker inspect <容器ID>
```

输出内容很长（完整 JSON），直接看容易迷路：可以丢给 AI 分析，也可以用 `-f` 只取需要的字段：

```bash
docker inspect -f '{{.State.Status}}' <容器ID>
```

### 创建但不运行

`docker run` 会创建并运行；如果只想先创建、不运行，用 `create` 新建一个容器：

```bash
docker create <镜像名>
```

创建之后再 `docker start` 就可以运行了。

### 查看日志

```bash
docker logs <容器名或容器ID>
```

持续跟踪日志（类似 `tail -f`）：

```bash
docker logs -f <容器名或容器ID>
```

### 进入容器

在容器里执行一条命令——容器 ID 或容器名后面直接跟上你想执行的 Linux 命令：

```bash
docker exec <容器ID或容器名> ps -ef
```

进入容器并获得一个交互式的命令行环境：

```bash
docker exec -it <容器ID> /bin/sh
```

有些容器也提供 bash：

```bash
docker exec -it <容器ID> /bin/bash
```

进入容器后查看 Linux 发行版信息，可以据此判断容器里可用的下载/包管理工具（apt、apk、yum 等）：

```bash
cat /etc/os-release
```

## 五、Docker 技术原理

Docker 利用了 Linux 内核的两大原生功能来实现容器化：

1. **Cgroups**：用来限制和隔离进程的**资源使用**。可以为每个容器设定 CPU、内存、网络带宽等资源的使用上限，确保一个容器的资源消耗不会影响宿主机或其他正在运行的容器。
2. **Namespaces**：用来隔离进程的**资源视图**。让容器只能看见自己内部的进程 ID、网络资源和文件目录，而看不到宿主机的。

所以容器本质上还是一个特殊的进程；但进入容器内部后，容器内部看起来就像一个独立的操作系统——每个 Docker 容器都是独立的运行环境，都表现得像一个独立的 Linux 系统。

## 六、Dockerfile 构建镜像

要点：

- 文件名固定为大写的 `Dockerfile`，没有后缀。
- 每一行是一条指令，指令关键字用大写（`FROM`、`RUN`、`COPY` 等），这是约定俗成的写法。
- 分层构建：每条指令生成一层镜像缓存，构建时会复用缓存以加快速度。
- 构建命令：`docker build`。

用于构建镜像的 Dockerfile 示例：

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

# 容器启动默认执行 bash
CMD ["/bin/bash"]
```

构建镜像的命令：

```bash
# 末尾的 . 代表构建上下文：当前目录（Dockerfile 就在当前目录）
docker build -t my-demo:v1 .
```

### 常用指令

| 指令 | 作用 | 示例 |
| --- | --- | --- |
| `FROM` | 必须是第一条，指定基础镜像 | `FROM ubuntu:22.04` |
| `RUN` | 构建镜像时执行命令（安装软件） | `RUN apt update && apt install -y curl` |
| `COPY` | 把宿主机文件复制到镜像内 | `COPY app.py /opt/` |
| `ADD` | 复制，额外支持自动解压压缩包 | `ADD demo.tar.gz /opt/` |
| `WORKDIR` | 设置镜像内部工作目录（相当于 `cd`） | `WORKDIR /opt/myapp` |
| `ENV` | 设置镜像内置环境变量（构建时写入镜像、作为运行时默认值；可在运行时用 `-e` 覆盖） | `ENV TZ=Asia/Shanghai` |
| `EXPOSE` | 声明容器打算暴露哪个端口，**不会**自动做 `-p` 映射 | `EXPOSE 8080` |
| `CMD` | 容器启动时默认执行的命令（只能有一条；`docker run` 后跟命令会覆盖它） | `CMD ["python3","app.py"]` |
| `ENTRYPOINT` | 容器入口程序，常与 `CMD` 配合（`CMD` 作为默认参数） | `ENTRYPOINT ["/bin/bash"]` |

## 七、Docker 网络

### bridge 桥接模式（默认）

- Docker 网络默认是 bridge 桥接模式。
- 所有容器默认都连接到这个网络，每个容器分配一个内部 IP 地址。
- 在内部子网中，容器之间可以通过内部 IP 地址相互访问。
- 但容器网络与宿主机的网络是隔离的。
- 可以使用 `docker network create` 创建自定义子网。
- 可以指定容器加入不同的子网：同一子网可相互通信，跨子网不可通信。
- **自定义子网**中的容器可以用容器名互相访问（默认的 bridge 网络不支持按名字解析，只能用 IP）。

### 自动分配 IP 的子网

创建一个叫 `network1` 的子网，IP 地址自动分配：

```bash
docker network create network1
```

运行并加入 `network1` 的指令：

```bash
docker run -d --name mysql8 \
  --network network1 \
  -e MYSQL_ROOT_PASSWORD=Root@123456 \
  mysql:8.0
```

### 指定静态 IP 的子网

创建一个叫 `network1` 的子网，指定网段与网关：

```bash
docker network create \
  --subnet=172.21.0.0/24 \
  --gateway=172.21.0.1 \
  network1
```

运行并加入 `network1`，同时指定静态 IP（只有自定义子网才能用 `--ip`）：

```bash
docker run -d --name mysql8 \
  --network network1 \
  --ip=172.21.0.10 \
  -e MYSQL_ROOT_PASSWORD=Root@123456 \
  mysql:8.0
```

### 运行中的容器加入 / 脱离子网

```bash
docker network connect network1 <容器ID>
docker network disconnect network1 <容器ID>
```

检验是否加入成功：

```bash
docker network inspect network1
```

看输出的 `Containers` 里有没有对应的容器名。

### host 模式

- 容器直接共享宿主机的网络，容器直接使用主机的 IP 地址。
- 无需 `-p` 进行端口映射，容器内的服务直接运行在宿主机的端口上。
- 通过宿主机的 IP 和端口就能访问到容器。

创建 host 模式的容器：

```bash
docker run -d --network host nginx
```

> [!note]
> host 模式下 `-p` 参数会被忽略（Docker 会给出警告）。

### none 模式

不联网，容器内只有 `lo` 回环接口：

```bash
docker run -d --network none nginx
```

### 网络管理

展示所有的 Docker 网络：

```bash
docker network list
```

删除子网：

```bash
docker network rm <网络ID>
```

进入容器后查看 IP：

```bash
ip addr show
```

## 八、Docker Compose

用一个 YAML 文件一次性编排多个容器，一键启停一组服务，不用敲很长一串 `docker run` 命令。

可以简单地理解为：把一条或多条 `docker run` 命令，按照特定格式列进一个文件里。

### 对照：不用 compose 的普通创建方式

```bash
# 1. 先创建自定义网络
docker network create --subnet=172.21.0.0/24 --gateway=172.21.0.1 network1

# 2. 启动 mysql 容器
docker run -d \
  --name mysql8 \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=Root@123456 \
  -e TZ=Asia/Shanghai \
  -v mysql_data:/var/lib/mysql \
  --network network1 \
  mysql:8.0

# 3. 启动 ubuntu 测试容器
docker run -d \
  --name ubuntu1 \
  -t \
  --network network1 \
  ubuntu:22.04
```

### 用 compose 改写

在 compose 中不需要手动创建子网：同一个 compose 文件中的所有容器会自动放在一个子网里。

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

### 使用流程

1. 创建并编辑 compose 文件（`compose.yaml` / `docker-compose.yaml` 均可）：

```bash
vi docker-compose.yaml
```

2. 写入上面的内容。
3. 启动文件中定义的所有容器：

```bash
docker compose up -d
```

### 常用命令

| 命令 | 作用 |
| --- | --- |
| `docker compose up -d` | 创建并后台启动文件中定义的所有服务 |
| `docker compose down` | 停止并删除容器、网络（加 `-v` 连卷一起删） |
| `docker compose stop` | 只停止容器，不删除 |
| `docker compose start` | 启动已存在的容器 |
| `docker compose -f <文件名> up -d` | 指定 compose 文件 |
| `docker compose -p <项目名> up -d` | 指定项目名（默认取所在目录名） |

> [!warning] 两个易错点
> 1. `docker compose down` **没有 `-d` 参数**。`down` 是同步执行的，删除完成后才返回，不需要后台标志。
> 2. `-f` 是 `--file` 的缩写，用于**指定 compose 文件路径**，不是给命令起别名。

### 核心对比

| 项目 | 多条 docker run | docker compose yaml |
| --- | --- | --- |
| 启动 | 复制多条长命令手动执行 | 一条 `up -d` 一键整套启动 |
| 网络 / 卷 | 需要手动提前 `network create`、手动管理 | yml 内部声明，自动创建网络、命名卷 |
| 服务访问 | 要记住容器名字互相访问 | 直接用服务名（`mysql`）访问，不用记容器名 |
| 修改配置 | 改完必须手动删容器，再复制整段 run 重建 | 修改 yml 后 `compose up -d` 自动重建变更的服务 |
| 停止清理 | 要分别 stop、rm 容器、rm 网络、rm 卷 | 一条 `down` 统一清理整套资源 |
| 适用场景 | 临时测试单个容器 | 多容器项目、开发环境、整套服务编排 |

### 重要等价映射回顾

| docker run 参数 | compose yaml 配置项 |
| --- | --- |
| `--name mysql8` | `container_name: mysql8` |
| `-p 3306:3306` | `ports: ["3306:3306"]` |
| `-e KEY=VAL` | `environment:` |
| `-v xxx:/path` | `volumes:` |
| `--network network1` | `networks: [network1]` |
| `-t` | `tty: true` |
