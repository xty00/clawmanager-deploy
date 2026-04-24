# ClawManager Docker Compose 部署

## 概述

ClawManager 及其依赖组件已从 K3s 迁移至 Docker Compose 部署。

**访问地址**: `https://192.168.99.150:30443`

## 架构

```
┌─────────────────────────────────────────────────────────┐
│  clawmanager-net (bridge)  172.22.0.0/16              │
│                                                         │
│  ┌──────────────┐  ┌──────────┐  ┌───────────────┐   │
│  │  MySQL 8.4   │  │  MinIO   │  │ Skill Scanner │   │
│  │  3306        │  │ 9000/9001│  │    8000       │   │
│  └──────────────┘  └──────────┘  └───────────────┘   │
│                                                         │
│              ┌──────────────────┐                       │
│              │  ClawManager App  │  ← 30443 → 9001     │
│              └──────────────────┘                       │
└─────────────────────────────────────────────────────────┘
```

## 组件说明

| 组件 | 端口 | 用途 | 数据卷 |
|------|------|------|--------|
| MySQL 8.4 | 3306 | 数据库 | `/mnt/Storage1/clawmanager/mysql` |
| MinIO | 9000/9001 | 对象存储 | `/mnt/Storage1/clawmanager/minio` |
| Skill Scanner | 8000 | 技能目录扫描 | 无持久化 |
| ClawManager | 30443 | 主应用 | 无持久化 |

## 快速命令

```bash
# 启动全部
cd /home/scott/clawmanager-docker-compose && docker-compose up -d

# 查看状态
docker ps | grep clawmanager

# 查看日志
docker logs -f clawmanager-app
docker logs -f clawmanager-mysql
docker logs -f clawmanager-minio
docker logs -f clawmanager-skill-scanner

# 重启某个服务
docker restart clawmanager-app
docker restart clawmanager-mysql

# 停止全部
cd /home/scott/clawmanager-docker-compose && docker-compose down
```

## 配置文件

- Compose 配置: `/home/scott/clawmanager-docker-compose/docker-compose.yml`
- MySQL 数据: `/mnt/Storage1/clawmanager/mysql` (208M)
- MinIO 数据: `/mnt/Storage1/clawmanager/minio`
- K3s 原版配置备份: `/mnt/Storage1/clawmanager/clawmanager.yaml`

## 数据库凭证

| 项目 | 值 |
|------|-----|
| Root 密码 | `root123` |
| 用户名 | `clawreef` |
| 密码 | `clawreef123` |
| 数据库名 | `clawmanager` |
| 主机 | `mysql` (容器内部) |

## 对象存储凭证

| 项目 | 值 |
|------|-----|
| 用户名 | `minioadmin` |
| 密码 | `minioadmin` |
| Bucket | `clawmanager` |
| API 端口 | `9000` |
| Console 端口 | `9001` |

## 健康检查

```bash
# MinIO
curl http://localhost:9000/minio/health

# SkillScanner
curl http://localhost:8000/

# ClawManager (会返回 404，这是正常的 API 行为)
curl http://localhost:30443/api/v1/instances
```

## 常见问题

### MySQL 数据目录被锁
如果 MySQL 启动失败并报 `Unable to lock ./ibdata1`，说明有残留进程：
```bash
# 查看占用进程
sudo fuser /mnt/Storage1/clawmanager/mysql/ibdata1

# 杀掉残留 mysqld 进程
sudo kill <PID>
```

### ClawManager 连接数据库失败
确认 MySQL 已经 healthy：
```bash
docker inspect clawmanager-mysql --format '{{.State.Health.Status}}'
```
等待状态变为 `healthy` 后再启动 ClawManager。

### 端口被占用
```bash
# 查看端口占用
ss -tlnp | grep 30443

# 确认是 Docker 端口映射
docker port clawmanager-app
```

## 从 K3s 迁移记录

- **迁移日期**: 2026-04-25
- **原因**: K3s 在设备重启后网络初始化存在竞态条件，导致循环崩溃
- **K3s 状态**: 已禁用 (`systemctl disable k3s`)
- **数据迁移**: MySQL 和 MinIO 数据目录保留原路径（均位于 `/mnt/Storage1/`）
- **镜像版本**: ClawManager `v2026.4.24`，MySQL `8.4.8`
