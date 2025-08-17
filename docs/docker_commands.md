# Docker 常用命令整理

## 1. 映像檔操作 (Image)

```bash
# 搜尋映像檔
docker search <image>

# 下載映像檔
docker pull <image>[:tag]

# 列出本地映像檔
docker images

# 刪除映像檔
docker rmi <image_id>

# 檢視映像檔詳細資訊
docker inspect <image_id>

# 為映像檔加上標籤
docker tag <image_id> <repo>:<tag>

# 匯出映像檔為 tar
docker save -o <file>.tar <image>

# 匯入映像檔
docker load -i <file>.tar
```

## 2. 容器操作 (Container)

```bash
# 建立並執行容器 (互動式)
docker run -it <image> /bin/bash

# 建立並執行容器 (背景執行)
docker run -d --name <container_name> <image>

# 列出正在執行的容器
docker ps

# 列出所有容器 (包含停止)
docker ps -a

# 停止容器
docker stop <container_id>

# 啟動容器
docker start <container_id>

# 重啟容器
docker restart <container_id>

# 刪除容器
docker rm <container_id>

# 檢視容器詳細資訊
docker inspect <container_id>

# 檢視容器資源使用
docker stats
```

## 3. 容器內操作

```bash
# 進入容器
docker exec -it <container_id> /bin/bash

# 查看容器日誌
docker logs <container_id>

# 實時查看容器日誌
docker logs -f <container_id>

# 複製檔案：容器 -> 主機
docker cp <container_id>:/path/in/container /host/path

# 複製檔案：主機 -> 容器
docker cp /host/path <container_id>:/path/in/container

# 檢視容器內程序
docker top <container_id>
```

## 4. 網路與埠口

```bash
# 建立容器並映射埠口
docker run -d -p 8080:80 <image>

# 檢視網路
docker network ls

# 建立自訂網路
docker network create <network_name>

# 加入容器到網路
docker network connect <network_name> <container_id>

# 從網路移除容器
docker network disconnect <network_name> <container_id>

# 檢視網路詳細資訊
docker network inspect <network_name>
```

## 5. Volume (資料卷)

```bash
# 建立 Volume
docker volume create <volume_name>

# 檢視 Volume 列表
docker volume ls

# 檢視 Volume 詳細資訊
docker volume inspect <volume_name>

# 刪除 Volume
docker volume rm <volume_name>

# 建立容器並掛載 Volume
docker run -d -v <volume_name>:/path/in/container <image>
```

## 6. Docker Compose

```bash
# 啟動
docker-compose up -d

# 停止
docker-compose down

# 查看服務狀態
docker-compose ps

# 查看日誌
docker-compose logs -f

# 重新啟動
docker-compose restart
```

## 7. 系統管理

```bash
# 檢視 Docker 系統資訊
docker info

# 清理未使用的資源
docker system prune -a

# 檢視磁碟使用情況
docker system df
```
