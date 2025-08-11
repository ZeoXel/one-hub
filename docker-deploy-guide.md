# One Hub Docker 部署指南

## 快速部署（推荐方式）

### 1. 使用 docker-compose 部署

项目已提供完整的 `docker-compose.yml` 配置，一键部署包含：
- One Hub 主服务（端口 3000）
- MySQL 数据库（端口 3306）  
- Redis 缓存

**部署步骤：**

```bash
# 1. 创建数据目录
mkdir -p ./data/mysql

# 2. 启动所有服务
docker-compose up -d

# 3. 查看运行状态
docker-compose ps

# 4. 查看日志
docker-compose logs -f one-hub
```

**访问地址：** http://localhost:3000

### 2. 环境变量配置

编辑 `docker-compose.yml` 中的环境变量：

```yaml
environment:
  - SQL_DSN=oneapi:123456@tcp(db:3306)/one-api
  - REDIS_CONN_STRING=redis://redis
  - SESSION_SECRET=your_random_string_here          # 修改为随机字符串
  - USER_TOKEN_SECRET=your_32_char_random_string    # 修改为32位随机字符串
  - TZ=Asia/Shanghai
  # - HASHIDS_SALT=random_string                    # 可选，建议设置
```

## 高级部署选项

### 仅使用 Docker 镜像（外部数据库）

```bash
docker run -d \
  --name one-hub \
  --restart always \
  -p 3000:3000 \
  -v ./data:/data \
  -v ./config.yaml:/data/config.yaml \
  -e SQL_DSN="mysql://user:password@host:3306/database" \
  -e REDIS_CONN_STRING="redis://redis:6379" \
  -e SESSION_SECRET="your_session_secret" \
  -e USER_TOKEN_SECRET="your_32_char_token_secret" \
  -e TZ=Asia/Shanghai \
  martialbe/one-api:latest
```

### 自定义构建

```bash
# 方式1：使用 Task（推荐）
task docker

# 方式2：手动构建
docker build -t one-hub:custom .

# 运行自定义镜像
docker run -d \
  --name one-hub \
  -p 3000:3000 \
  -v ./data:/data \
  one-hub:custom
```

## 配置说明

### 数据库配置

支持三种数据库：

1. **MySQL**（推荐生产环境）
```bash
SQL_DSN=username:password@tcp(host:port)/database
```

2. **PostgreSQL**
```bash
SQL_DSN=postgres://username:password@host:port/database?sslmode=disable
```

3. **SQLite**（默认，开发环境）
```bash
# 不设置 SQL_DSN 即使用 SQLite
```

### Redis 配置

```bash
# 基础配置
REDIS_CONN_STRING=redis://localhost:6379

# 带认证
REDIS_CONN_STRING=redis://username:password@localhost:6379

# 指定数据库
REDIS_CONN_STRING=redis://localhost:6379/1
```

### 多节点部署

**主节点配置：**
```yaml
environment:
  - NODE_TYPE=master
  - SQL_DSN=mysql://user:pass@host:3306/db
  - REDIS_CONN_STRING=redis://redis:6379
```

**从节点配置：**
```yaml
environment:
  - NODE_TYPE=slave
  - FRONTEND_BASE_URL=https://your-main-domain.com
  - SYNC_FREQUENCY=60
  - SQL_DSN=mysql://user:pass@host:3306/db
  - REDIS_CONN_STRING=redis://redis:6379
```

## 监控和维护

### 健康检查

docker-compose 已配置健康检查：
```bash
# 查看健康状态
docker-compose ps

# 手动健康检查
curl http://localhost:3000/api/status
```

### 日志管理

```bash
# 查看实时日志
docker-compose logs -f one-hub

# 查看特定服务日志
docker-compose logs mysql
docker-compose logs redis

# 限制日志输出行数
docker-compose logs --tail=100 one-hub
```

### 备份和恢复

**数据备份：**
```bash
# 备份 MySQL 数据
docker exec mysql mysqldump -u root -p one-api > backup_$(date +%Y%m%d).sql

# 备份持久化数据
tar -czf data_backup_$(date +%Y%m%d).tar.gz ./data
```

**数据恢复：**
```bash
# 恢复 MySQL 数据
docker exec -i mysql mysql -u root -p one-api < backup.sql

# 恢复持久化数据
tar -xzf data_backup.tar.gz
```

## 常见问题

### 端口冲突
如果 3000 端口被占用，修改 docker-compose.yml：
```yaml
ports:
  - "8080:3000"  # 外部访问 8080，容器内仍使用 3000
```

### 数据库连接问题
1. 检查数据库服务是否启动：`docker-compose ps`
2. 检查连接字符串格式
3. 确认数据库用户权限

### 性能优化
1. 启用 Redis 缓存
2. 配置内存缓存：`MEMORY_CACHE_ENABLED=true`
3. 调整数据库连接池大小

### SSL/HTTPS 配置
推荐使用 Nginx 或 Traefik 作为反向代理处理 SSL：

```yaml
# 添加到 docker-compose.yml
nginx:
  image: nginx:alpine
  ports:
    - "443:443"
    - "80:80"
  volumes:
    - ./nginx.conf:/etc/nginx/nginx.conf
    - ./ssl:/etc/nginx/ssl
  depends_on:
    - one-hub
```

## 升级指南

```bash
# 拉取最新镜像
docker-compose pull

# 重启服务
docker-compose up -d

# 查看版本
curl http://localhost:3000/api/status | jq .version
```