# Alcali 生产环境 Docker 部署

## 前置要求

### 1. 系统要求
- Docker 20.10+
- Docker Compose 2.0+
- 至少2GB内存
- 10GB可用磁盘空间

### 2. 数据库准备

#### MySQL/MariaDB 配置
```sql
CREATE DATABASE salt DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE USER 'alcali'@'%' IDENTIFIED BY 'your_secure_password';
GRANT ALL PRIVILEGES ON salt.* TO 'alcali'@'%';

USE salt;

-- 创建必要的表
CREATE TABLE jids (
  jid varchar(255) NOT NULL,
  load mediumtext NOT NULL,
  UNIQUE KEY jid (jid)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE salt_returns (
  fun varchar(50) NOT NULL,
  jid varchar(255) NOT NULL,
  return mediumtext NOT NULL,
  id varchar(255) NOT NULL,
  success varchar(10) NOT NULL,
  full_ret mediumtext NOT NULL,
  alter_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  KEY id (id),
  KEY jid (jid),
  KEY fun (fun)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE salt_events (
  id BIGINT NOT NULL AUTO_INCREMENT,
  tag varchar(255) NOT NULL,
  data mediumtext NOT NULL,
  alter_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  master_id varchar(255) NOT NULL,
  PRIMARY KEY (id),
  KEY tag (tag)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

#### PostgreSQL 配置
```sql
CREATE ROLE alcali WITH PASSWORD 'your_secure_password' LOGIN;
CREATE DATABASE salt WITH OWNER alcali;

\c salt;

CREATE TABLE jids (
  jid varchar(20) PRIMARY KEY,
  load text NOT NULL
);

CREATE TABLE salt_returns (
  fun varchar(50) NOT NULL,
  jid varchar(255) NOT NULL,
  return text NOT NULL,
  full_ret text,
  id varchar(255) NOT NULL,
  success varchar(10) NOT NULL,
  alter_time TIMESTAMP WITH TIME ZONE DEFAULT now()
);

CREATE INDEX idx_salt_returns_id ON salt_returns (id);
CREATE INDEX idx_salt_returns_jid ON salt_returns (jid);
CREATE INDEX idx_salt_returns_fun ON salt_returns (fun);

CREATE SEQUENCE seq_salt_events_id;
CREATE TABLE salt_events (
    id BIGINT NOT NULL UNIQUE DEFAULT nextval('seq_salt_events_id'),
    tag varchar(255) NOT NULL,
    data text NOT NULL,
    alter_time TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    master_id varchar(255) NOT NULL
);
CREATE INDEX idx_salt_events_tag on salt_events (tag);
```

## 3. Salt Master 配置

### 安装Salt Master和Salt-API
```bash
# Debian/Ubuntu
apt-get update
apt-get install -y salt-master salt-api python3-openssl

# CentOS/RHEL
yum install -y salt-master salt-api python3-openssl
```

### 配置Salt Master (/etc/salt/master)
```yaml
# 数据库配置 (MySQL示例)
event_return: [mysql]
master_job_cache: mysql
mysql.host: 'your_db_host'
mysql.user: 'alcali'
mysql.pass: 'your_secure_password'
mysql.db: 'salt'
mysql.port: 3306

# 作业缓存设置
keep_jobs: 0  # 禁用自动清理，由数据库管理

# Salt-API配置
rest_cherrypy:
  port: 8080
  host: 0.0.0.0
  debug: False
  ssl_crt: /etc/pki/tls/certs/localhost.crt
  ssl_key: /etc/pki/tls/certs/localhost.key

# 外部认证配置
external_auth:
  rest:
    ^url: http://your_alcali_host:8000/api/token/verify/
    admin:
      - .*
      - '@runner'
      - '@wheel'
```

### 生成SSL证书
```bash
salt-call --local tls.create_self_signed_cert cacert_path='/etc/pki'
```

### 启动服务
```bash
systemctl enable salt-master salt-api
systemctl start salt-master salt-api
```

## 4. Alcali 容器部署

### 创建环境配置文件
```bash
# 创建 .env 文件
cat > .env << EOF
# 数据库配置
DB_BACKEND=mysql  # 或 postgresql
DB_NAME=salt
DB_USER=alcali
DB_PASS=your_secure_password
DB_HOST=your_db_host
DB_PORT=3306  # MySQL: 3306, PostgreSQL: 5432

# Django配置
SECRET_KEY=$(openssl rand -base64 32)
ALLOWED_HOSTS=your_domain.com,your_ip_address
DEBUG=False

# Salt配置
MASTER_MINION_ID=master
SALT_URL=https://your_salt_master:8080
SALT_AUTH=rest

# 可选: LDAP配置
# AUTH_BACKEND=ldap
# AUTH_LDAP_SERVER_URI=ldap://your_ldap_server
# AUTH_LDAP_BIND_DN=cn=admin,dc=example,dc=org
# AUTH_LDAP_BIND_PASSWORD=ldap_password
# AUTH_LDAP_USER_BASE_CN=dc=example,dc=org

# 可选: Google OAuth2配置
# AUTH_BACKEND=social
# SOCIAL_AUTH_GOOGLE_OAUTH2_KEY=your_google_client_id
# SOCIAL_AUTH_GOOGLE_OAUTH2_SECRET=your_google_client_secret
# SOCIAL_AUTH_REDIRECT_URI=https://your_domain.com
EOF
```

### 创建Docker Compose文件
```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  alcali:
    image: latenighttales/alcali:latest
    restart: unless-stopped
    ports:
      - "8000:8000"
    env_file:
      - .env
    volumes:
      - alcali_static:/opt/alcali/static
      - alcali_media:/opt/alcali/media
    command: >
      bash -c "
        alcali migrate &&
        alcali collectstatic --noinput &&
        gunicorn config.wsgi:application
        --bind 0.0.0.0:8000
        --workers 4
        --timeout 120
        --max-requests 1000
        --max-requests-jitter 100
      "
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/api/stats/"]
      interval: 30s
      timeout: 10s
      retries: 3

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - alcali_static:/var/www/static:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - alcali

volumes:
  alcali_static:
  alcali_media:
```

### 创建Nginx配置
```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream alcali {
        server alcali:8000;
    }

    server {
        listen 80;
        server_name your_domain.com;
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name your_domain.com;

        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;

        client_max_body_size 100M;

        location /static/ {
            alias /var/www/static/;
            expires 30d;
            add_header Cache-Control "public, immutable";
        }

        location / {
            proxy_pass http://alcali;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }
    }
}
```

### 部署步骤

1. **创建超级用户**
```bash
# 首次部署需要创建管理员用户
docker compose -f docker-compose.prod.yml run --rm alcali alcali createsuperuser
```

2. **启动服务**
```bash
docker compose -f docker-compose.prod.yml up -d
```

3. **验证部署**
```bash
# 检查服务状态
docker compose -f docker-compose.prod.yml ps

# 查看日志
docker compose -f docker-compose.prod.yml logs -f alcali

# 测试API
curl -f http://localhost:8000/api/stats/
```

### 维护操作

#### 更新Alcali
```bash
# 拉取最新镜像
docker compose -f docker-compose.prod.yml pull

# 重启服务
docker compose -f docker-compose.prod.yml up -d

# 运行数据库迁移（如果需要）
docker compose -f docker-compose.prod.yml exec alcali alcali migrate
```

#### 备份数据
```bash
# 备份数据库（MySQL示例）
mysqldump -h your_db_host -u alcali -p salt > alcali_backup_$(date +%Y%m%d).sql

# 备份用户上传文件
docker run --rm -v alcali_media:/data -v $(pwd):/backup alpine tar czf /backup/alcali_media_$(date +%Y%m%d).tar.gz -C /data .
```

#### 监控和日志
```bash
# 实时查看日志
docker compose -f docker-compose.prod.yml logs -f

# 查看资源使用
docker stats

# 健康检查
docker compose -f docker-compose.prod.yml exec alcali alcali check
```
