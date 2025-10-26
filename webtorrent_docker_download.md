# Docker WebTorrent 多任务下载方案

## 方案概述

使用 Docker 官方 Node.js 镜像运行 webtorrent-cli，实现多任务并行下载。采用独立容器架构——**每个下载任务使用一个独立的 Docker 容器**，这样可以：

- 同时下载多个文件
- 随时添加新的下载任务
- 独立管理每个任务的启动、停止、删除
- 每个任务拥有独立的日志文件

## 实施步骤

### 1. 构建 WebTorrent 镜像

首先需要构建预装 webtorrent-cli 的 Docker 镜像。在项目目录中执行以下步骤：

**步骤 1：创建 Dockerfile**

```bash
cat > Dockerfile << 'EOF'
FROM node:lts

# 全局安装 webtorrent-cli
RUN npm install -g webtorrent-cli

# 设置工作目录
WORKDIR /downloads

# 设置入口点，保持容器运行
ENTRYPOINT ["webtorrent"]
EOF
```

**步骤 2：创建 .dockerignore**

```bash
cat > .dockerignore << 'EOF'
node_modules
npm-debug.log
.git
.gitignore
README.md
*.md
.vscode
.idea
*.log
.DS_Store
EOF
```

**步骤 3：创建镜像构建脚本**

```bash
cat > build-image.sh << 'EOF'
#!/bin/bash

# WebTorrent CLI 镜像构建脚本

set -e

IMAGE_NAME="webtorrent-cli"
IMAGE_TAG="latest"
FULL_IMAGE_NAME="${IMAGE_NAME}:${IMAGE_TAG}"

echo "开始构建 ${FULL_IMAGE_NAME} 镜像..."

# 构建镜像
docker build -t "${FULL_IMAGE_NAME}" .

echo ""
echo "镜像构建完成！"
echo ""
echo "镜像信息:"
docker images "${IMAGE_NAME}"

echo ""
echo "使用方法:"
echo "  启动下载任务:"
echo "  docker run -d --name webtorrent-task-1 --restart unless-stopped --network host \\"
echo "    -v ~/webtorrent/downloads:/downloads -v ~/webtorrent/logs:/logs \\"
echo "    ${FULL_IMAGE_NAME} 'magnet:?xt=urn:btih:YOUR_LINK' > /logs/task-1.log 2>&1"
EOF

chmod +x build-image.sh
```

**步骤 4：构建镜像**

```bash
# 使用构建脚本构建镜像
./build-image.sh

# 或者直接使用 docker build 命令
docker build -t webtorrent-cli:latest .
```

**镜像构建说明：**

- 基于 `node:lts` 官方镜像
- 预装 `webtorrent-cli`（避免每次启动时安装）
- 工作目录设置为 `/downloads`
- 启动速度更快，资源利用更高效

### 2. 准备宿主机目录

在宿主机上创建用于存储下载文件和日志的目录：

```bash
mkdir -p ~/webtorrent/downloads
mkdir -p ~/webtorrent/logs
```

### 3. 启动下载任务

使用以下命令模板启动第一个下载任务：

```bash
docker run -d \
  --name webtorrent-task-1 \
  --restart unless-stopped \
  --network host \
  -v ~/webtorrent/downloads:/downloads \
  -v ~/webtorrent/logs:/logs \
  webtorrent-cli:latest \
  bash -c "cd /downloads && \
           webtorrent --on-done 'kill 1' 'magnet:?xt=urn:btih:a733c037a81e6b14c4c20abdb963f2718b0aa1b7&dn=[javdb.com]JUR-572.mp4' \
           > /logs/task-1.log 2>&1"
```

**命令说明：**

- `-d`: 后台运行容器
- `--name webtorrent-task-1`: 容器名称，建议使用编号便于管理(如 `task-1`, `task-2` 等)
- `--restart unless-stopped`: 容器异常退出时自动重启，除非手动停止
- `--network host`: 使用宿主机网络，确保 BT 流量正常连接
- `-v ~/webtorrent/downloads:/downloads`: 挂载下载目录
- `-v ~/webtorrent/logs:/logs`: 挂载日志目录
- `webtorrent-cli:latest`: 使用预装 webtorrent-cli 的自定义镜像（启动速度更快）
- `bash -c "..."`: 执行一系列命令
- `> /logs/task-1.log 2>&1`: 将输出重定向到独立的日志文件，建议使用 `task-编号.log` 命名

### 4. 添加新的下载任务

当需要下载新文件时，只需要修改容器名称、磁力链接和日志文件名，再次执行命令即可：

**启动第二个任务：**

```bash
docker run -d \
  --name webtorrent-task-2 \
  --restart unless-stopped \
  --network host \
  -v ~/webtorrent/downloads:/downloads \
  -v ~/webtorrent/logs:/logs \
  webtorrent-cli:latest \
  bash -c "cd /downloads && \
           webtorrent --on-done 'kill 1' 'magnet:?xt=urn:btih:另一个磁力链接' \
           > /logs/task-2.log 2>&1"
```

**快速添加任务技巧：**

```bash
# 将以下内容保存为函数，添加到 ~/.bashrc 或 ~/.zshrc 中
add_download() {
  local task_name=$1
  local magnet_link=$2
  
  docker run -d \
    --name "webtorrent-${task_name}" \
    --restart unless-stopped \
    --network host \
    -v ~/webtorrent/downloads:/downloads \
    -v ~/webtorrent/logs:/logs \
    webtorrent-cli:latest \
    bash -c "cd /downloads && \
                 webtorrent --on-done 'kill 1' '${magnet_link}' \
                 > /logs/${task_name}.log 2>&1"
  
  echo "任务 ${task_name} 已启动，日志: ~/webtorrent/logs/${task_name}.log"
}

# 使用方式：
# add_download "movie-1" "magnet:?xt=urn:btih:..."
```

### 5. 管理下载任务

**查看所有下载任务：**

```bash
# 查看正在运行的任务
docker ps | grep webtorrent-task

# 查看所有任务（包括已停止的）
docker ps -a | grep webtorrent-task
```

**查看特定任务的进度和日志：**

```bash
# 查看任务 1 的日志
tail -f ~/webtorrent/logs/task-1.log

# 查看任务 2 的日志
tail -f ~/webtorrent/logs/task-2.log

# 同时查看所有任务日志
tail -f ~/webtorrent/logs/*.log
```

**停止特定任务：**

```bash
docker stop webtorrent-task-1
```

**重启特定任务：**

```bash
docker restart webtorrent-task-1
```

**删除特定任务：**

```bash
# 停止并删除容器
docker stop webtorrent-task-1 && docker rm webtorrent-task-1

# 或者直接删除（如果容器已停止）
docker rm webtorrent-task-1
```

**批量管理操作：**

```bash
# 停止所有下载任务
docker ps -q --filter "name=webtorrent-task" | xargs -r docker stop

# 重启所有下载任务
docker ps -aq --filter "name=webtorrent-task" | xargs -r docker restart

# 删除所有已停止的任务
  docker ps -aq --filter "name=webtorrent-task" --filter "status=exited" | xargs -r docker rm
```

### 6. 下载完成后

所有下载完成的文件将保存在 `~/webtorrent/downloads` 目录中，每个任务的日志保存在对应的 `~/webtorrent/logs/task-N.log` 文件中。

当一个任务下载完成后，通过 `--on-done 'kill 1'` 参数，对应的容器会自动退出。

**查看已完成的任务：**

```bash
# 查看所有已退出的容器
docker ps -a --filter "name=webtorrent-task" --filter "status=exited"

# 查看退出时间和状态码
docker ps -a | grep webtorrent-task
```

**清理已完成的容器：**

```bash
# 删除所有已完成的容器
docker ps -aq --filter "name=webtorrent-task" --filter "status=exited" | xargs -r docker rm
```

## 注意事项

- 每个容器会在下载完成后自动退出：通过 `--on-done 'kill 1'` 参数实现，容器正常退出（exit code 0）时不会触发 `--restart unless-stopped` 的重启策略
- 只有容器异常退出（非 0 状态码）时才会自动重启
- 可以随时添加新的下载任务，只需要使用不同的容器名称和日志文件名
- 日志文件会持续增长，建议定期清理：`> ~/webtorrent/logs/task-N.log`（保留日志但清空内容）
- 下载同一个磁力链接时，使用不同的容器名称可以创建多个下载实例
- 建议使用任务编号或描述性名称（如 `movie-123`、`album-456`）来命名容器，便于管理

## 高级用法

### 批量添加下载任务

创建一个包含磁力链接列表的文件 `torrents.txt`：

```bash
cat > ~/webtorrent/torrents.txt << EOF
magnet:?xt=urn:btih:a733c037a81e6b14c4c20abdb963f2718b0aa1b7
magnet:?xt=urn:btih:另一个磁力链接
magnet:?xt=urn:btih:第三个磁力链接
EOF
```

然后使用脚本批量启动任务：

```bash
#!/bin/bash
task_num=1

while IFS= read -r magnet_link; do
  docker run -d \
    --name "webtorrent-task-${task_num}" \
    --restart unless-stopped \
    --network host \
    -v ~/webtorrent/downloads:/downloads \
    -v ~/webtorrent/logs:/logs \
    webtorrent-cli:latest \
    bash -c "cd /downloads && \
             webtorrent --on-done 'kill 1' '${magnet_link}' \
             > /logs/task-${task_num}.log 2>&1"
  
  echo "任务 ${task_num} 已启动"
  ((task_num++))
  sleep 2  # 避免并发过多
done < ~/webtorrent/torrents.txt
```
