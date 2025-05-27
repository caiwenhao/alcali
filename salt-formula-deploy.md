# Alcali Salt Formula 部署

## 概述

Salt Formula是部署Alcali最推荐的方式，特别适合已有Salt环境的生产部署。

## 1. 安装Formula

### 方法1: 使用gitfs
```yaml
# /etc/salt/master
gitfs_remotes:
  - https://github.com/latenighttales/alcali-formula.git

# 刷新pillar和状态
salt-run saltutil.refresh_pillar
salt-run saltutil.sync_all
```

### 方法2: 手动下载
```bash
# 下载到formula目录
cd /srv/formulas
git clone https://github.com/latenighttales/alcali-formula.git alcali

# 配置master
echo "file_roots:" >> /etc/salt/master
echo "  base:" >> /etc/salt/master
echo "    - /srv/salt" >> /etc/salt/master
echo "    - /srv/formulas/alcali" >> /etc/salt/master
```

## 2. 配置Pillar

### 基础配置
```yaml
# /srv/pillar/alcali.sls
alcali:
  # 数据库配置
  db:
    backend: mysql  # 或 postgresql
    name: salt
    user: alcali
    password: your_secure_password
    host: localhost
    port: 3306

  # Django配置
  django:
    secret_key: your_secret_key_here
    allowed_hosts:
      - your_domain.com
      - your_server_ip
    debug: False

  # Salt配置
  salt:
    url: https://localhost:8080
    auth: rest
    master_minion_id: master

  # Web服务器配置
  web:
    bind: 127.0.0.1:8000
    workers: 4
    user: alcali
    group: alcali

  # Nginx配置
  nginx:
    enabled: True
    server_name: your_domain.com
    ssl:
      cert: /path/to/cert.pem
      key: /path/to/key.pem
```

### 高级配置示例
```yaml
# /srv/pillar/alcali.sls
alcali:
  # 版本控制
  version: "3006.3.0"
  
  # 安装方式
  install_method: pip  # pip, source, docker
  
  # 数据库配置
  db:
    backend: postgresql
    name: salt
    user: alcali
    password: !vault |
      $ANSIBLE_VAULT;1.1;AES256
      66386439653...
    host: db.example.com
    port: 5432
    
  # 认证配置
  auth:
    backend: ldap  # django, ldap, social
    ldap:
      server_uri: ldap://ldap.example.com
      bind_dn: cn=admin,dc=example,dc=org
      bind_password: ldap_password
      user_base_cn: ou=users,dc=example,dc=org
      user_search_filter: "(uid=%(user)s)"
      
  # 社交认证
  social:
    google:
      client_id: your_google_client_id
      client_secret: your_google_client_secret
      whitelisted_domains:
        - example.com
        
  # 系统配置
  system:
    user: alcali
    group: alcali
    home: /opt/alcali
    virtualenv: /opt/alcali/venv
    
  # 服务配置
  service:
    enabled: True
    running: True
    
  # 日志配置
  logging:
    level: INFO
    file: /var/log/alcali/alcali.log
    max_size: 100MB
    backup_count: 5
    
  # 备份配置
  backup:
    enabled: True
    schedule: "0 2 * * *"  # 每天凌晨2点
    retention: 30  # 保留30天
    destination: /backup/alcali
```

### Pillar Top文件
```yaml
# /srv/pillar/top.sls
base:
  'alcali-server':
    - alcali
```

## 3. 部署状态文件

### 基础部署状态
```yaml
# /srv/salt/alcali-deploy.sls
include:
  - alcali

# 确保数据库已创建
alcali_database:
  mysql_database.present:
    - name: salt
    - require:
      - pkg: mysql-server

# 确保数据库用户存在
alcali_db_user:
  mysql_user.present:
    - name: alcali
    - password: {{ pillar['alcali']['db']['password'] }}
    - host: '%'
    - require:
      - mysql_database: alcali_database

# 授予权限
alcali_db_grants:
  mysql_grants.present:
    - grant: all privileges
    - database: salt.*
    - user: alcali
    - host: '%'
    - require:
      - mysql_user: alcali_db_user
```

### 完整部署状态
```yaml
# /srv/salt/alcali-full.sls
# 包含数据库、Web服务器、SSL等完整配置

include:
  - alcali
  - nginx
  - mysql

# SSL证书
alcali_ssl_cert:
  file.managed:
    - name: /etc/ssl/certs/alcali.crt
    - contents_pillar: alcali:ssl:cert
    - mode: 644
    - require_in:
      - service: nginx

alcali_ssl_key:
  file.managed:
    - name: /etc/ssl/private/alcali.key
    - contents_pillar: alcali:ssl:key
    - mode: 600
    - require_in:
      - service: nginx

# 防火墙配置
alcali_firewall_http:
  firewalld.present:
    - name: public
    - ports:
      - 80/tcp
      - 443/tcp

# 监控配置
alcali_monitoring:
  file.managed:
    - name: /etc/nagios/nrpe.d/alcali.cfg
    - contents: |
        command[check_alcali]=/usr/lib/nagios/plugins/check_http -H localhost -p 8000 -u /api/stats/
```

## 4. 执行部署

### 单机部署
```bash
# 应用状态
salt 'alcali-server' state.apply alcali

# 或使用完整配置
salt 'alcali-server' state.apply alcali-full

# 验证部署
salt 'alcali-server' cmd.run 'systemctl status alcali'
salt 'alcali-server' cmd.run 'curl -f http://localhost:8000/api/stats/'
```

### 高可用部署
```yaml
# /srv/salt/alcali-ha.sls
# 多节点高可用部署

{% for node in pillar['alcali']['ha']['nodes'] %}
alcali_{{ node }}:
  salt.state:
    - tgt: {{ node }}
    - sls: alcali
    - pillar:
        alcali:
          db:
            host: {{ pillar['alcali']['ha']['db_cluster'] }}
          web:
            bind: {{ grains['ip4_interfaces']['eth0'][0] }}:8000
{% endfor %}

# 负载均衡器配置
alcali_lb:
  salt.state:
    - tgt: 'role:loadbalancer'
    - sls: haproxy
    - pillar:
        haproxy:
          backends:
            {% for node in pillar['alcali']['ha']['nodes'] %}
            - {{ node }}:8000
            {% endfor %}
```

## 5. 维护和更新

### 更新Alcali
```bash
# 更新pillar中的版本
salt 'alcali-server' pillar.item alcali:version

# 应用更新
salt 'alcali-server' state.apply alcali

# 重启服务
salt 'alcali-server' service.restart alcali
```

### 备份和恢复
```bash
# 执行备份
salt 'alcali-server' state.apply alcali.backup

# 恢复数据
salt 'alcali-server' state.apply alcali.restore pillar='{"restore_date": "2024-01-15"}'
```

### 监控和日志
```bash
# 检查服务状态
salt 'alcali-server' service.status alcali

# 查看日志
salt 'alcali-server' cmd.run 'journalctl -u alcali -n 50'

# 性能监控
salt 'alcali-server' cmd.run 'ps aux | grep gunicorn'
salt 'alcali-server' cmd.run 'netstat -tlnp | grep 8000'
```

## 6. 故障排除

### 常见问题
```bash
# 检查配置
salt 'alcali-server' cmd.run 'alcali check'

# 测试数据库连接
salt 'alcali-server' cmd.run 'alcali dbshell'

# 检查Salt-API连接
salt 'alcali-server' cmd.run 'curl -k https://localhost:8080'

# 重新应用状态
salt 'alcali-server' state.apply alcali test=True
```
