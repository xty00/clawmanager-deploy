# Armbian + NVIDIA Box 部署参考

## 硬件环境

| 项目 | 规格 |
|------|------|
| 设备 | Khadas VIM4 (或类似 ARM 开发板) |
| 系统 | Armbian OS 26.05.0 (Debian trixie) |
| 架构 | ARMv8 aarch64 (6-core CPU) |
| eMMC | 6.1 GB (系统盘) |
| SSD | 110 GB (数据盘，挂在 `/mnt/Storage1`) |
| 网络 | 双网口 (eth0 + eth1) |

## 网络配置

```
eth0: 192.168.99.150/24  (主网段)
eth1: 192.168.137.10/24  (备用网段)
```

**重要**: 设备在两个网段之间切换时，Docker 服务需要网络自适应。

## 存储布局

```
/mnt/Storage1/           (SSD, 110GB)
├── clawmanager/
│   ├── mysql/           MySQL 8.4.8 数据 (~208MB)
│   └── minio/           MinIO 数据
├── docker_data/         Docker 数据根目录
└── ...

/ (eMMC, 6.1GB)          系统盘
├── /var/lib/docker      → /mnt/Storage1/docker_data (软链接)
└── /var/lib/rancher    → /mnt/Storage1/k3s_data (软链接)
```

## Docker 安装

```bash
# 安装 Docker CE
curl -fsSL https://get.docker.com | sh

# 配置 Docker 数据目录到 SSD
sudo mkdir -p /mnt/Storage1/docker_data
sudo tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "data-root": "/mnt/Storage1/docker_data",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF

sudo systemctl restart docker
```

## 部署步骤

```bash
# 1. 克隆仓库
git clone https://github.com/xty00/clawmanager-deploy.git
cd clawmanager-deploy

# 2. 配置环境变量
cp .env.example .env
nano .env  # 修改 MYSQL_DATA_PATH 和 MINIO_DATA_PATH

# 3. 创建数据目录
sudo mkdir -p /mnt/Storage1/clawmanager/mysql
sudo mkdir -p /mnt/Storage1/clawmanager/minio
sudo chown -R 999:999 /mnt/Storage1/clawmanager/mysql/

# 4. 启动服务
docker-compose up -d

# 5. 验证
docker ps | grep clawmanager
curl http://localhost:9000/minio/health/live
```

## 端口映射

| 宿主机端口 | 容器端口 | 服务 |
|-----------|---------|------|
| `30443` | `9001` | ClawManager 主应用 |
| `3306` | `3306` | MySQL |
| `9000` | `9000` | MinIO API |
| `9001` | `9001` | MinIO Console |
| `8000` | `8000` | Skill Scanner |

## 访问地址

- ClawManager: `http://192.168.99.150:30443`
- MinIO Console: `http://192.168.99.150:9001`
- Skill Scanner API: `http://192.168.99.150:8000`

## 已知问题与解决

### K3s 不适合此硬件

K3s 在 ARM 开发板上重启后存在 kubelet 与 Flannel 竞态条件，导致节点 IP 识别失败，K3s 进入 crash loop。

**解决方案**: 使用 Docker Compose 部署，绕开 K3s。

### 双网段切换

设备在 eth0 和 eth1 之间切换时，Docker 容器网络不受影响，但宿主机上的服务端口绑定需要确认。

```bash
# 检查当前活跃网段
ip route get 1.1.1.1
```

### 数据目录权限

MySQL 容器使用 UID 999 (mysql 用户)，确保数据目录可写:

```bash
sudo chown -R 999:999 /mnt/Storage1/clawmanager/mysql/
```

## 备份策略

```bash
# MySQL 数据 (容器内导出)
docker exec clawmanager-mysql mysqldump -u root -p clawmanager > backup.sql

# MinIO 数据 (直接复制)
sudo rsync -av /mnt/Storage1/clawmanager/minio/ /backup/minio/

# 完整备份脚本
#!/bin/bash
DATE=$(date +%Y%m%d)
BACKUP_DIR=/mnt/Storage1/backups
mkdir -p $BACKUP_DIR
docker exec clawmanager-mysql mysqldump -u root -proot123 clawmanager > $BACKUP_DIR/mysql_$DATE.sql
rsync -av /mnt/Storage1/clawmanager/ $BACKUP_DIR/data_$DATE/
```
