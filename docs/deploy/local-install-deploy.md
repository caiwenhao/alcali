# Alcali 本地安装部署

## 系统要求

- Python 3.8+
- 数据库: MySQL/MariaDB 或 PostgreSQL
- Salt Master 和 Salt-API
- 至少2GB内存

## 1. 系统依赖安装

### Debian/Ubuntu
```bash
# 基础依赖
apt-get update
apt-get install -y python3 python3-pip python3-venv git

# 数据库连接器依赖
# MySQL/MariaDB
apt-get install -y libmariadbclient-dev gcc

# PostgreSQL
apt-get install -y libpq-dev gcc

# LDAP支持（可选）
apt-get install -y libldap2-dev libsasl2-dev ldap-utils
```

### CentOS/RHEL
```bash
# 基础依赖
yum install -y python3 python3-pip git gcc

# 数据库连接器依赖
# MySQL/MariaDB
yum install -y mysql-devel

# PostgreSQL
yum install -y libpq-devel

# LDAP支持（可选）
yum install -y openldap-devel
```

## 2. 创建虚拟环境

```bash
# 创建专用用户（推荐）
useradd -m -s /bin/bash alcali
su - alcali

# 创建虚拟环境
python3 -m venv ~/.venv/alcali
source ~/.venv/alcali/bin/activate

# 升级pip
pip install --upgrade pip
```

## 3. 安装Alcali

### 从PyPI安装（推荐）
```bash
# 基础安装
pip install alcali

# 数据库支持
# MySQL/MariaDB
pip install mysqlclient

# PostgreSQL
pip install psycopg2

# 可选功能
# LDAP支持
pip install alcali[ldap]

# Google OAuth2支持
pip install alcali[social]

# 完整安装
pip install alcali[ldap,social] mysqlclient psycopg2
```

### 从源码安装
```bash
# 克隆项目
git clone https://github.com/latenighttales/alcali.git
cd alcali

# 切换到稳定版本
git checkout 3006.3.0  # 或最新版本

# 安装
pip install .[ldap,social] mysqlclient psycopg2
```

## 4. 配置环境

### 创建配置目录
```bash
mkdir -p /opt/alcali
cd /opt/alcali
```

### 创建环境配置文件
```bash
cat > .env << EOF
# 数据库配置
DB_BACKEND=mysql
DB_NAME=salt
DB_USER=alcali
DB_PASS=your_secure_password
DB_HOST=localhost
DB_PORT=3306

# Django配置
SECRET_KEY=$(openssl rand -base64 32)
ALLOWED_HOSTS=localhost,127.0.0.1,your_server_ip
DEBUG=False

# Salt配置
MASTER_MINION_ID=master
SALT_URL=https://localhost:8080
SALT_AUTH=rest

# 静态文件目录
STATIC_ROOT=/opt/alcali/static
MEDIA_ROOT=/opt/alcali/media
EOF
```

## 5. 初始化应用

```bash
# 设置环境变量
export ENV_PATH=/opt/alcali/.env

# 验证配置
alcali check

# 运行数据库迁移
alcali migrate

# 收集静态文件
alcali collectstatic --noinput

# 创建超级用户
alcali createsuperuser
```

## 6. 配置Web服务器

### 使用Gunicorn + Nginx

#### 创建Gunicorn配置
```python
# /opt/alcali/gunicorn.conf.py
bind = "127.0.0.1:8000"
workers = 4
worker_class = "sync"
worker_connections = 1000
max_requests = 1000
max_requests_jitter = 100
timeout = 120
keepalive = 5
preload_app = True
user = "alcali"
group = "alcali"
tmp_upload_dir = None
errorlog = "/var/log/alcali/gunicorn_error.log"
accesslog = "/var/log/alcali/gunicorn_access.log"
loglevel = "info"
```

#### 创建systemd服务
```ini
# /etc/systemd/system/alcali.service
[Unit]
Description=Alcali Salt Management Interface
After=network.target

[Service]
Type=notify
User=alcali
Group=alcali
WorkingDirectory=/opt/alcali
Environment=ENV_PATH=/opt/alcali/.env
Environment=PATH=/home/alcali/.venv/alcali/bin
ExecStart=/home/alcali/.venv/alcali/bin/gunicorn config.wsgi:application -c /opt/alcali/gunicorn.conf.py --chdir $(alcali location)
ExecReload=/bin/kill -s HUP $MAINPID
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

#### 配置Nginx
```nginx
# /etc/nginx/sites-available/alcali
server {
    listen 80;
    server_name your_domain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your_domain.com;

    ssl_certificate /path/to/your/cert.pem;
    ssl_certificate_key /path/to/your/key.pem;

    client_max_body_size 100M;

    location /static/ {
        alias /opt/alcali/static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    location /media/ {
        alias /opt/alcali/media/;
        expires 7d;
    }

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

## 7. 启动服务

```bash
# 创建日志目录
mkdir -p /var/log/alcali
chown alcali:alcali /var/log/alcali

# 启用并启动服务
systemctl enable alcali
systemctl start alcali

# 启用Nginx
ln -s /etc/nginx/sites-available/alcali /etc/nginx/sites-enabled/
systemctl reload nginx

# 检查状态
systemctl status alcali
systemctl status nginx
```

## 8. 维护操作

### 更新Alcali
```bash
# 激活虚拟环境
source ~/.venv/alcali/bin/activate

# 更新包
pip install --upgrade alcali

# 运行迁移
ENV_PATH=/opt/alcali/.env alcali migrate

# 收集静态文件
ENV_PATH=/opt/alcali/.env alcali collectstatic --noinput

# 重启服务
systemctl restart alcali
```

### 日志管理
```bash
# 查看应用日志
journalctl -u alcali -f

# 查看Gunicorn日志
tail -f /var/log/alcali/gunicorn_error.log

# 查看Nginx日志
tail -f /var/log/nginx/error.log
```
