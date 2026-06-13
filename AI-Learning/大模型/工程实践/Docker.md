---
aliases: [Docker, 容器]
tags: [AI/Learning, 工具, Docker, 容器]
created: 2026-06-13
---

# Docker 工具参考

> **一句话总结**：Docker 把应用和环境打包成轻量容器，实现"一次构建，到处运行"，解决"我电脑上能跑"的环境不一致难题。

---

## 一、Docker 是什么

开源容器引擎（Go 语言），核心理念：**Build, Ship and Run Any App, Anywhere**。

### Docker vs 虚拟机

| 对比项 | Docker 容器 | 虚拟机（VM） |
|--------|------------|-------------|
| 隔离级别 | 共享宿主机内核 | 完整 OS 虚拟化 |
| 启动速度 | 秒级 | 分钟级 |
| 资源占用 | MB 级 | GB 级 |
| 隔离性 | 较弱（命名空间+cgroup） | 强（独立内核） |
| 适用场景 | 微服务、CI/CD、快速部署 | 强隔离、多 OS 需求 |

> 类比：虚拟机是租一整栋楼（含地基），Docker 是租一间房（共享楼体）。

---

## 二、核心概念

| 概念 | 类比 | 说明 |
|------|------|------|
| **镜像（Image）** | 软件安装包 | 只读模板，包含运行环境全部文件；可创建多个容器 |
| **容器（Container）** | 已安装的软件 | 镜像的运行实例，可读写，独立隔离；删除后数据丢失 |
| **仓库（Repository）** | 应用商店 | 存放镜像的平台；官方仓库 [Docker Hub](https://hub.docker.com/) |
| **数据卷（Volume）** | D 盘 | 持久化存储，容器删除后数据不丢 |

**C/S 架构**：客户端通过 CLI 或 RESTful API 向 Docker 守护进程（dockerd）发送指令。

---

## 三、常用命令速查

### 3.1 镜像管理

```bash
docker pull <镜像名>:<标签>       # 拉取镜像，如 docker pull mysql:8.0.45
docker images                     # 查看本地所有镜像
docker rmi <镜像名>               # 删除镜像
docker save <镜像> -o <文件.tar>  # 导出镜像为 tar 文件
docker load -i <文件.tar>        # 从 tar 文件导入镜像
docker build -t <名称> .          # 根据 Dockerfile 构建镜像
docker tag <源> <新名:标签>       # 给镜像打标签
```

### 3.2 容器管理

```bash
# 创建并启动
docker run -d --name <名称> -p <主机端口>:<容器端口> -e KEY=VAL <镜像>
#  -d 后台运行  --name 命名  -p 端口映射  -e 环境变量

# 查看
docker ps                  # 查看运行中的容器
docker ps -a               # 查看所有容器（含已停止）
docker logs <容器名>       # 查看日志
docker exec -it <容器名> bash  # 进入容器终端

# 控制
docker start <容器名>      # 启动已停止的容器
docker stop <容器名>       # 停止容器
docker rm <容器名>         # 删除已停止的容器
docker rm -f <容器名>      # 强制删除（含运行中）

# 一键停止并删除所有容器
docker stop $(docker ps -aq) && docker rm $(docker ps -aq)
```

### 3.3 数据卷

```bash
# 运行时挂载
docker run -v <宿主路径>:<容器路径> <镜像>
# 如：docker run -v D:/mysql-data:/var/lib/mysql mysql:8.0.45

docker volume ls                    # 查看所有数据卷
docker volume create <卷名>         # 创建命名卷
docker volume rm <卷名>             # 删除数据卷
```

### 3.4 网络与清理

```bash
docker network ls                   # 查看网络
docker network create <网络名>      # 创建桥接网络
docker system prune -af             # 清理无用镜像/容器/缓存
```

---

## 四、Dockerfile

Dockerfile 是构建自定义镜像的脚本，无后缀名。

### 4.1 常用指令

| 指令 | 作用 | 示例 |
|------|------|------|
| `FROM` | 指定基础镜像 | `FROM python:3.12` |
| `WORKDIR` | 设置工作目录 | `WORKDIR /app` |
| `COPY` | 复制本地文件到镜像 | `COPY . /app` |
| `ADD` | 复制并自动解压 tar | `ADD app.tar.gz /app` |
| `RUN` | 构建时执行命令 | `RUN pip install pymysql` |
| `ENV` | 设置环境变量 | `ENV PYTHONUNBUFFERED=1` |
| `EXPOSE` | 声明端口（文档用） | `EXPOSE 8080` |
| `CMD` | 容器启动时默认命令 | `CMD ["python", "main.py"]` |
| `ENTRYPOINT` | 同 CMD 但不可被覆盖 | `ENTRYPOINT ["python"]` |
| `ARG` | 构建时参数（不保留到运行时） | `ARG VERSION=3.12` |

### 4.2 示例

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

### 4.3 多阶段构建

减小镜像体积，构建与运行环境分离：

```dockerfile
# 构建阶段
FROM python:3.12 AS builder
COPY . /src
RUN pip install --no-cache-dir -r /src/requirements.txt

# 运行阶段
FROM python:3.12-slim
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY . /app
WORKDIR /app
CMD ["python", "app.py"]
```

### 4.4 最佳实践

- 用 `.dockerignore` 排除无关文件（同 `.gitignore`）
- 合并 RUN 指令减少层数：`RUN apt update && apt install -y curl && rm -rf /var/lib/apt/lists/*`
- 优先用 `slim`/`alpine` 基础镜像减小体积
- 将变化少的层放前面（利用缓存），如先 COPY `requirements.txt` 再 COPY 源码
- 避免在镜像中存放密钥，运行时通过 `-e` 或 `--env-file` 注入

---

## 五、Docker Compose

多容器编排工具，通过 YAML 文件声明式管理。

### 5.1 核心语法

```yaml
services:                         # 顶层 key
  mysql-db:
    image: mysql:8.0.45          # 指定镜像
    restart: always              # 重启策略：always / on-failure / no
    ports:
      - "9999:3306"              # 主机端口:容器端口
    environment:                  # 环境变量
      MYSQL_ROOT_PASSWORD: "123456"
    volumes:
      - mysql-data:/var/lib/mysql # 命名卷挂载
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # 文件挂载
    networks:
      - app-network
    healthcheck:                  # 健康检查
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 3s
      timeout: 3s
      retries: 10

  python-app:
    build: .                     # 从 Dockerfile 构建
    depends_on:
      mysql-db:
        condition: service_healthy  # 等依赖服务健康后再启动
    environment:
      MYSQL_HOST: "mysql-db"     # 容器名即 DNS
    networks:
      - app-network

volumes:                         # 声明命名卷
  mysql-data:

networks:                        # 声明网络
  app-network:
    driver: bridge
```

### 5.2 常用命令

```bash
docker compose up -d              # 后台启动所有服务
docker compose up -d --build      # 重新构建后启动
docker compose down               # 停止并删除容器（保留卷）
docker compose down -v            # 停止并删除容器+数据卷（慎用）
docker compose ps                 # 查看服务状态
docker compose logs <服务名>      # 查看日志
docker compose restart <服务名>   # 重启指定服务
docker compose exec <服务名> bash # 进入容器
```

---

## 六、常见场景

| 场景 | 典型用法 |
|------|---------|
| **本地开发环境** | 一键拉取 MySQL/Redis/PostgreSQL，不污染本机 |
| **AI 模型训练** | 基于 `nvidia/cuda` 镜像构建 GPU 训练环境 |
| **模型部署服务** | 用 Dockerfile 封装推理代码 + 模型，搭配 FastAPI 对外服务 |
| **多服务编排** | Compose 同时启动数据库 + 应用 + Nginx + Redis |
| **CI/CD** | 构建阶段容器化编译测试，产出镜像直接部署 |
| **环境移植** | `docker save/load` 离线迁移，跨平台环境 100% 一致 |
| **微服务架构** | 每个服务独立容器，通过 Docker 网络互联 |

---

## 七、镜像源配置（国内加速）

编辑 Docker Desktop > Settings > Docker Engine，或 `/etc/docker/daemon.json`：

```json
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://docker.1ms.run",
    "https://func.ink"
  ]
}
```

修改后重启 Docker 服务生效。Linux 下需额外执行：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

---

## 八、常见问题

| 问题 | 解决 |
|------|------|
| Win11 启动失败提示 WSL2 未配置 | 确认 BIOS 虚拟化已开启，WSL2 内核已安装，重启 |
| `Connection refused` | 检查容器是否 UP：`docker compose ps`；Compose 中用 `healthcheck` + `depends_on.condition` |
| YAML 格式错误 | 缩进用空格不用 Tab，冒号后加空格，引号用英文 |
| Ubuntu 权限不足 | `sudo usermod -aG docker $USER` 后重新登录 |
| 容器内数据丢失 | 挂载 `-v` 数据卷做持久化 |
| 镜像拉取超时 | 配置国内镜像源（见第七节） |

---

## 相关笔记

- [[00. 大模型知识体系导航]]
- [[HuggingFace生态]]
- [[FastAPI_SQLAlchemy_笔记]]
