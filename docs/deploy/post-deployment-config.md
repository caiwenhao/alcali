# Alcali 部署后配置和优化

## 1. 初始配置

### 登录和基础设置
1. 访问Web界面: `https://your_domain.com`
2. 使用创建的超级用户登录
3. 进入Settings页面进行初始配置

### 接受Minion密钥
```bash
# 在Keys页面或使用命令行
salt-key -A  # 接受所有待处理的密钥
```

### 配置Minion字段
在Settings页面添加常用的Minion字段:
```yaml
# 推荐的Minion字段配置
highstate: state.show_highstate
top_file: state.show_top
packages: pkg.list_pkgs
services: service.get_all
network: network.interfaces
disk_usage: disk.usage
```

### 配置合规性检查
添加自定义合规性检查:
```yaml
# 示例合规性配置
file_exists: file.file_exists /etc/important_config
service_running: service.status nginx
package_version: pkg.version nginx
security_updates: pkg.list_upgrades
```

## 2. 性能优化

### 数据库优化

#### MySQL/MariaDB优化
```sql
-- 添加索引优化查询性能
ALTER TABLE salt_returns ADD INDEX idx_alter_time (alter_time);
ALTER TABLE salt_returns ADD INDEX idx_success (success);
ALTER TABLE salt_events ADD INDEX idx_alter_time (alter_time);

-- 配置参数优化
SET GLOBAL innodb_buffer_pool_size = 1073741824;  -- 1GB
SET GLOBAL query_cache_size = 268435456;  -- 256MB
SET GLOBAL max_connections = 200;
```

#### PostgreSQL优化
```sql
-- 创建额外索引
CREATE INDEX CONCURRENTLY idx_salt_returns_alter_time ON salt_returns (alter_time);
CREATE INDEX CONCURRENTLY idx_salt_returns_success ON salt_returns (success);
CREATE INDEX CONCURRENTLY idx_salt_events_alter_time ON salt_events (alter_time);

-- 配置参数
ALTER SYSTEM SET shared_buffers = '256MB';
ALTER SYSTEM SET effective_cache_size = '1GB';
ALTER SYSTEM SET work_mem = '4MB';
SELECT pg_reload_conf();
```

### 应用层优化

#### Gunicorn配置优化
```python
# gunicorn.conf.py
import multiprocessing

# 基于CPU核心数计算worker数量
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = "sync"
worker_connections = 1000
max_requests = 1000
max_requests_jitter = 100

# 超时设置
timeout = 120
keepalive = 5
graceful_timeout = 30

# 内存优化
preload_app = True
max_worker_memory = 200  # MB

# 日志配置
loglevel = "info"
access_log_format = '%(h)s %(l)s %(u)s %(t)s "%(r)s" %(s)s %(b)s "%(f)s" "%(a)s" %(D)s'
```

#### Django设置优化
```python
# 在.env文件中添加
DJANGO_CACHE_BACKEND=redis
DJANGO_CACHE_LOCATION=redis://localhost:6379/1
DJANGO_SESSION_ENGINE=django.contrib.sessions.backends.cache
DJANGO_SESSION_CACHE_ALIAS=default

# 数据库连接池
DB_CONN_MAX_AGE=300
DB_CONN_HEALTH_CHECKS=True
```

### 缓存配置

#### Redis缓存
```bash
# 安装Redis
apt-get install redis-server

# 配置Redis
echo "maxmemory 512mb" >> /etc/redis/redis.conf
echo "maxmemory-policy allkeys-lru" >> /etc/redis/redis.conf
systemctl restart redis-server
```

#### 应用缓存配置
```bash
# 在.env文件中添加缓存配置
CACHE_BACKEND=redis
CACHE_LOCATION=redis://localhost:6379/1
CACHE_TIMEOUT=300
SESSION_CACHE_ALIAS=default
```

## 3. 安全配置

### SSL/TLS配置
```nginx
# Nginx SSL最佳实践
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;

# HSTS
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

# 其他安全头
add_header X-Frame-Options DENY always;
add_header X-Content-Type-Options nosniff always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

### 防火墙配置
```bash
# UFW配置示例
ufw allow ssh
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow from trusted_ip to any port 8080  # Salt-API
ufw enable

# 或使用iptables
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -p tcp --dport 8080 -s trusted_ip -j ACCEPT
```

### 访问控制
```yaml
# 在.env文件中配置
ALLOWED_HOSTS=your_domain.com,trusted_ip
SECURE_SSL_REDIRECT=True
SECURE_HSTS_SECONDS=31536000
SECURE_CONTENT_TYPE_NOSNIFF=True
SECURE_BROWSER_XSS_FILTER=True
```

## 4. 监控和日志

### 应用监控
```bash
# 创建监控脚本
cat > /usr/local/bin/alcali-health-check.sh << 'EOF'
#!/bin/bash
# Alcali健康检查脚本

# 检查Web服务
if ! curl -f -s http://localhost:8000/api/stats/ > /dev/null; then
    echo "ERROR: Alcali web service is down"
    exit 1
fi

# 检查数据库连接
if ! docker exec alcali alcali check | grep -q "db: ok"; then
    echo "ERROR: Database connection failed"
    exit 1
fi

# 检查Salt-API连接
if ! curl -k -f -s https://localhost:8080 > /dev/null; then
    echo "ERROR: Salt-API is not accessible"
    exit 1
fi

echo "OK: All services are healthy"
exit 0
EOF

chmod +x /usr/local/bin/alcali-health-check.sh
```

### 日志轮转
```bash
# 配置logrotate
cat > /etc/logrotate.d/alcali << 'EOF'
/var/log/alcali/*.log {
    daily
    missingok
    rotate 30
    compress
    delaycompress
    notifempty
    create 644 alcali alcali
    postrotate
        systemctl reload alcali
    endscript
}
EOF
```

### Prometheus监控
```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'alcali'
    static_configs:
      - targets: ['localhost:8000']
    metrics_path: '/metrics'
    scrape_interval: 30s
```

## 5. 备份策略

### 自动备份脚本
```bash
#!/bin/bash
# /usr/local/bin/alcali-backup.sh

BACKUP_DIR="/backup/alcali"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

# 创建备份目录
mkdir -p $BACKUP_DIR

# 备份数据库
mysqldump -h localhost -u alcali -p$DB_PASS salt > $BACKUP_DIR/alcali_db_$DATE.sql

# 备份配置文件
tar -czf $BACKUP_DIR/alcali_config_$DATE.tar.gz /opt/alcali/.env /etc/nginx/sites-available/alcali

# 备份用户数据
tar -czf $BACKUP_DIR/alcali_media_$DATE.tar.gz /opt/alcali/media

# 清理旧备份
find $BACKUP_DIR -name "*.sql" -mtime +$RETENTION_DAYS -delete
find $BACKUP_DIR -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup completed: $DATE"
```

### 定时备份
```bash
# 添加到crontab
0 2 * * * /usr/local/bin/alcali-backup.sh >> /var/log/alcali/backup.log 2>&1
```

## 6. 故障排除

### 常见问题诊断
```bash
# 检查服务状态
systemctl status alcali nginx mysql

# 检查端口监听
netstat -tlnp | grep -E ':(80|443|8000|8080)'

# 检查日志
journalctl -u alcali -f
tail -f /var/log/nginx/error.log
tail -f /var/log/alcali/gunicorn_error.log

# 测试数据库连接
mysql -h localhost -u alcali -p salt -e "SELECT COUNT(*) FROM salt_returns;"

# 测试Salt-API
curl -k https://localhost:8080

# 检查磁盘空间
df -h
du -sh /var/log/alcali/
```

### 性能问题排查
```bash
# 检查系统资源
top
htop
iotop

# 检查数据库性能
mysql -e "SHOW PROCESSLIST;"
mysql -e "SHOW ENGINE INNODB STATUS\G"

# 检查应用性能
ps aux | grep gunicorn
strace -p $(pgrep -f gunicorn)
```
