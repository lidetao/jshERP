# jshERP 部署指南

全栈 docker-compose 一键部署。数据库使用阿里云 RDS MySQL。

```
┌─────────────────────────────────────────────┐
│                    ECS                        │
│                                              │
│  docker-compose                              │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐ │
│  │ frontend │  │ backend  │  │   redis   │ │
│  │ nginx:80 │──▶ :9999    │──▶   :6379   │ │
│  └──────────┘  └────┬─────┘  └───────────┘ │
│                     │                       │
│              .env ← 注入密码                 │
└─────────────────────┼───────────────────────┘
                      │
             ┌────────┴────────┐
             │ 阿里云 RDS MySQL │
             └─────────────────┘
```

---

## 一、前提条件

| 组件 | 说明 |
|------|------|
| **阿里云 RDS MySQL 8.0** | 已创建数据库 `jsh_erp`（字符集 utf8mb4），与 ECS 同 VPC |
| **阿里云 ECS** | 已安装 Docker + docker-compose |

### ECS 安装 Docker

```bash
# CentOS
yum install -y yum-utils
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
systemctl enable docker --now

# Ubuntu
apt update && apt install -y docker.io docker-compose-v2
systemctl enable docker --now
```

---

## 二、配置（安全设计）

```
敏感信息流向：
.env ──docker-compose──▶ 环境变量 ──Spring Boot──▶ 应用
                        ${DB_PASSWORD}    ${DB_PASSWORD:root}
```

- **`.env`**（不提交 Git）：唯一存密码的地方，`chmod 600`
- **`application.yml`**（可提交 Git）：只用 `${VAR}` 占位符，零明文
- **`docker-compose.yml`**：从 `.env` 读取，注入容器环境变量

### 2.1 创建 `.env` 并设置权限

```bash
cp deploy/.env.example deploy/.env
chmod 600 deploy/.env
```

编辑 `deploy/.env`，填入真实值：

```bash
DB_HOST=rm-xxx.mysql.rds.aliyuncs.com
DB_USERNAME=jsh_erp
DB_PASSWORD=你的RDS密码
REDIS_PASSWORD=你的Redis密码
```

### 2.2 `application.yml`（无需修改）

已使用 `${DB_URL}`、`${DB_PASSWORD}`、`${REDIS_PASSWORD}` 等环境变量占位符，默认可直接使用。

如需修改文件存储方式，编辑 `deploy/config/application.yml`：
- `uploadType: 1` → 本地存储（Docker volume 持久化）
- `uploadType: 2` → 阿里云 OSS（需在 `jsh_platform_config` 表配置 OSS 参数）

---

## 三、部署

### 3.1 上传项目到服务器

```bash
rsync -av --exclude 'node_modules' --exclude '.git' \
  /path/to/jshERP/ root@<ECS_IP>:/opt/jshERP/
```

> `.env` 不会被 Git 跟踪，需额外手动上传或直接在 ECS 上创建。

### 3.2 构建并启动

```bash
# ECS 上执行
cd /opt/jshERP

# 构建后端 JAR（需要 JDK 8 + Maven，首次装一下）
cd jshERP-boot && mvn clean package -DskipTests && cd ..

# 一键启动
docker-compose -f deploy/docker-compose.yml up -d --build
```

---

## 四、验证

```bash
# 容器状态
docker-compose -f deploy/docker-compose.yml ps

# Redis
docker exec jsherp-redis redis-cli -a $REDIS_PASSWORD ping   # → PONG

# MySQL
mysql -h $DB_HOST -u $DB_USERNAME -p -e "SHOW DATABASES;"    # → 有 jsh_erp

# 后端 API
curl http://localhost:9999/jshERP-boot/doc.html

# 前端 → http://<ECS_IP>
```

登录：租户 `jsh`，用户 `admin`，密码 `123456`

---

## 五、日常运维

```bash
cd /opt/jshERP

# 启动 / 停止
docker-compose -f deploy/docker-compose.yml up -d
docker-compose -f deploy/docker-compose.yml down

# 查看日志
docker-compose -f deploy/docker-compose.yml logs -f backend

# 更新后端
cd jshERP-boot && mvn clean package -DskipTests && cd ..
docker-compose -f deploy/docker-compose.yml up -d --build backend

# 更新前端
docker-compose -f deploy/docker-compose.yml up -d --build frontend
```

---

## 六、端口与安全

| 端口 | 服务 | 暴露范围 |
|------|------|----------|
| 80 | 前端 Nginx | 公网 |
| 9999 | 后端 API | 仅 127.0.0.1 |
| 6379 | Redis | 仅 127.0.0.1 |
| 3306 | RDS MySQL | 仅 VPC 内网 |
