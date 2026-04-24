# 常见问题排查

## 目录

- [健康检查](#健康检查)
- [MySQL 相关](#mysql-相关)
- [MinIO 相关](#minio-相关)
- [ClawManager 主应用](#clawmanager-主应用)
- [端口与网络](#端口与网络)
- [从 K3s 迁移后的遗留问题](#从-k3s-迁移后的遗留问题)

---

## 健康检查

```bash
# 查看所有容器状态
docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# 查看单个容器日志
docker logs clawmanager-mysql --tail 50
docker logs clawmanager-app --tail 50

# 检查容器内网络连通性
docker exec clawmanager-mysql mysqladmin ping -h localhost
docker exec clawmanager-app wget -O- http://localhost:9001/
```

---

## MySQL 相关

### MySQL 容器启动失败

**症状**: `docker ps` 显示 MySQL 状态不是 `healthy`

**排查步骤**:

```bash
# 1. 查看详细日志
docker logs clawmanager-mysql

# 2. 检查数据目录是否被其他进程占用
sudo fuser /mnt/Storage1/clawmanager/mysql/ibdata1
ps aux | grep mysqld

# 3. 清理残留进程
sudo kill -9 <PID>
```

**原因**: K3s 迁移后，残留的 mysqld 进程（由 K3s 的 containerd 管理）仍锁定了数据文件。

---

### 连接被拒绝 (Access Denied)

**症状**: `ERROR 1045 (28000): Access denied for user 'clawreef'@'%'`

```bash
# 进入 MySQL 容器手动排查
docker exec -it clawmanager-mysql mysql -u root -p

# 检查用户
SELECT user, host FROM mysql.user WHERE user = 'clawreef';

# 如果用户不存在，手动创建
CREATE USER 'clawreef'@'%' IDENTIFIED BY 'clawreef123';
GRANT ALL PRIVILEGES ON clawmanager.* TO 'clawreef'@'%';
FLUSH PRIVILEGES;
```

---

### 数据卷权限问题

**症状**: `Permission denied` in docker logs

```bash
# 检查目录权限
ls -la /mnt/Storage1/clawmanager/mysql/

# 修复权限
sudo chown -R 999:999 /mnt/Storage1/clawmanager/mysql/
```

---

## MinIO 相关

### MinIO Console 无法访问

```bash
# 检查 MinIO 是否正常运行
docker exec clawmanager-minio mc ready local

# 检查健康状态
curl http://localhost:9000/minio/health/live

# 访问 Console
# 浏览器打开 http://<host>:9001
# 默认凭证: minioadmin / minioadmin
```

### Bucket 不存在

```bash
# 进入 MinIO CLI
docker exec -it clawmanager-minio mc alias set local http://localhost:9000 minioadmin minioadmin

# 创建 bucket
docker exec clawmanager-minio mc mb local/clawmanager --ignore-existing

# 匿名访问策略
docker exec clawmanager-minio mc anonymous set download local/clawmanager
```

---

## ClawManager 主应用

### 容器状态 unhealthy

**症状**: `clawmanager-app` 容器一直 `unhealthy`

```bash
# 查看应用日志
docker logs clawmanager-app --tail 100

# 检查数据库连接
docker exec clawmanager-app wget -O- http://localhost:9001/api/v1/health 2>/dev/null

# 确认 MySQL 已就绪
docker inspect clawmanager-mysql --format '{{.State.Health.Status}}'
```

### 前端页面 404 或空白

```bash
# 检查端口映射
docker port clawmanager-app

# 确认 Nginx/反向代理配置正确
# ClawManager 容器内部监听 :9001，需通过 30443 访问
```

---

## 端口与网络

### 端口被占用

```bash
# 查看端口占用
ss -tlnp | grep -E '30443|3306|9000|9001|8000'

# 找到占用进程
lsof -i :30443

# 解决方案: 更改 .env 中的端口映射
```

### 容器之间无法通信

```bash
# 检查网络是否存在
docker network ls | grep clawmanager

# 检查 DNS 解析
docker exec clawmanager-app nslookup mysql
docker exec clawmanager-app nslookup minio

# 重建网络
docker network rm clawmanager-net
docker-compose up -d
```

---

## 从 K3s 迁移后的遗留问题

### K3s 进程仍然运行并锁定文件

```bash
# 检查是否有残留 K3s mysqld
ps aux | grep -E 'mysqld|k3s'

# 如果 K3s 已禁用但进程仍在
sudo systemctl stop k3s
sudo systemctl disable k3s

# 手动 kill 残留进程
sudo pkill -9 mysqld
sudo pkill -9 containerd-shim
```

### K3s 数据目录残留导致路径混淆

```
K3s MySQL 数据: /mnt/Storage1/clawmanager/mysql/
Docker MySQL 数据: /mnt/Storage1/clawmanager/mysql/

# 两者路径相同！确保 K3s 彻底停止后再启动 Docker MySQL
```

---

## 重置流程

如果问题无法排查，最彻底的解决方案:

```bash
# 1. 停止所有容器
cd /home/scott/clawmanager-docker-compose
docker-compose down

# 2. 确认无残留进程
ps aux | grep -E 'mysql|k3s|containerd' | grep -v grep

# 3. 备份数据
sudo rsync -av /mnt/Storage1/clawmanager/mysql/ /mnt/Storage1/clawmanager/mysql.backup/

# 4. 重新启动
docker-compose up -d

# 5. 验证
docker ps && echo "=== All healthy ===" && \
docker inspect clawmanager-mysql --format '{{.State.Health.Status}}' && \
docker inspect clawmanager-app --format '{{.State.Health.Status}}'
```
