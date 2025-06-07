# Salt 配置示例文件

本文档提供了 Salt Master 和 Minion 的配置示例文件，基于最佳实践进行组织。

## Salt Master 配置示例

### 1. 基础配置 (`/etc/salt/master.d/00-main.conf`)

```yaml
# /etc/salt/master.d/00-main.conf
# Salt Master 基础配置

# 监听所有网络接口
interface: 0.0.0.0

# 端口配置（使用默认值）
publish_port: 4505
ret_port: 4506

# 安全设置 - 不自动接受密钥
auto_accept: False

# 日志配置
log_level: warning
log_file: /var/log/salt/master

# 进程配置
user: root
timeout: 5

# 启用 keep_alive
keep_alive: True
keep_alive_interval: 30
```

### 2. 文件根目录配置 (`/etc/salt/master.d/10-file_roots.conf`)

```yaml
# /etc/salt/master.d/10-file_roots.conf
# 文件服务器和 Pillar 配置

file_roots:
  base:
    - /srv/salt
  dev:
    - /srv/salt/dev
    - /srv/salt
  prod:
    - /srv/salt/prod
    - /srv/salt

pillar_roots:
  base:
    - /srv/pillar
  dev:
    - /srv/pillar/dev
    - /srv/pillar
  prod:
    - /srv/pillar/prod
    - /srv/pillar

# 环境顺序
env_order:
  - base
  - dev
  - prod

# Top 文件合并策略
top_file_merging_strategy: merge_all
```

### 3. 性能调优配置 (`/etc/salt/master.d/20-tuning.conf`)

```yaml
# /etc/salt/master.d/20-tuning.conf
# 性能优化配置

# 工作线程数（根据 CPU 核心数调整）
worker_threads: 16

# 发布器线程池
pub_pool_size: 5

# 返回处理线程池
ret_pool_size: 5

# 事件队列大小
event_pub_queue_size: 1000000

# 文件描述符限制
max_open_files: 100000

# 作业缓存设置
job_cache: True
keep_jobs: 24

# 超时设置
gather_job_timeout: 10
timeout: 60

# 负载控制
minion_load_cutoff: 0.8
```

### 4. 安全配置 (`/etc/salt/master.d/30-security.conf`)

```yaml
# /etc/salt/master.d/30-security.conf
# 安全相关配置

# 启用密钥轮换
rotate_aes_key: True

# 外部认证示例（根据需要启用）
# external_auth:
#   pam:
#     saltadmin:
#       - .*
#       - '@wheel'
#       - '@runner'
#       - '@jobs'

# 客户端 ACL 示例
# client_acl:
#   saltadmin:
#     - .*

# 发布者 ACL 示例
# publisher_acl:
#   saltadmin:
#     - .*

# 模块黑名单（禁用危险模块）
# module_blacklist:
#   - cmd.run
#   - cmd.shell

# 启用签名验证
# sign_pub_messages: True
```

## Salt Minion 配置示例

### 1. 基础配置 (`/etc/salt/minion.d/00-main.conf`)

```yaml
# /etc/salt/minion.d/00-main.conf
# Salt Minion 基础配置

# Master 地址
master: saltmaster.yourdomain.com

# Minion ID（建议显式设置）
# FQDN = Fully Qualified Domain Name，如: web01.example.com
id: {{ grains['fqdn'] }}

# 其他 ID 选项:
# id: {{ grains['host'] }}                    # 仅主机名
# id: prod-web-{{ grains['host'] }}           # 自定义前缀
# id: {{ grains['ip4_interfaces']['eth0'][0] }} # IP 地址（不推荐）

# 端口配置
master_port: 4506
publish_port: 4505

# 日志配置
log_level: warning
log_file: /var/log/salt/minion

# DNS 重试设置
retry_dns: 30

# 连接重试设置
master_tries: -1

# 连接超时
master_alive_interval: 30
master_timeout: 10

# 启用 keep_alive
tcp_keepalive: True
tcp_keepalive_idle: 300
tcp_keepalive_cnt: 3
tcp_keepalive_intvl: 30
```

### 2. 网络配置 (`/etc/salt/minion.d/10-network.conf`)

```yaml
# /etc/salt/minion.d/10-network.conf
# 网络和连接配置

# 重连延迟设置
recon_default: 1000
recon_max: 59000
recon_randomize: True

# 多 Master 配置示例
# master:
#   - saltmaster1.yourdomain.com
#   - saltmaster2.yourdomain.com
# random_master: True
# master_shuffle: True

# 网络接口绑定（如果需要）
# interface: 0.0.0.0
# source_interface_name: eth0
# source_address: 192.168.1.100

# IPv6 支持
# ipv6: False
```

### 3. Grains 配置 (`/etc/salt/minion.d/20-grains.conf`)

```yaml
# /etc/salt/minion.d/20-grains.conf
# 自定义 Grains 配置

grains:
  # 服务器角色
  role: webserver

  # 环境标识
  environment: production

  # 数据中心位置
  datacenter: us-east-1
  availability_zone: us-east-1a

  # 团队信息
  team: devops
  owner: infrastructure

  # 应用信息
  app_stack: nginx-php-mysql
  app_version: "2.1.0"

  # 硬件信息
  server_type: virtual
  instance_size: large

  # 网络信息
  network_zone: dmz
  subnet: web-tier

  # 业务信息
  cost_center: "12345"
  project: "web-platform"

  # 监控标签
  monitoring: enabled
  backup: enabled
```

### 4. 性能配置 (`/etc/salt/minion.d/30-performance.conf`)

```yaml
# /etc/salt/minion.d/30-performance.conf
# 性能和缓存配置

# 启用作业缓存
cache_jobs: True
job_cache_store_endtime: 24

# 模块缓存
module_dirs: []

# 多进程处理
multiprocessing: True

# 超时设置
timeout: 60
gather_job_timeout: 10

# 启用压缩
compression: gzip

# 缓存设置
grains_cache: True
grains_cache_expiration: 300

# 启用 Minion 数据缓存
minion_data_cache: True
```

## 快速部署脚本

### Master 配置部署脚本

```bash
#!/bin/bash
# deploy-master-config.sh

# 创建配置目录
sudo mkdir -p /etc/salt/master.d

# 复制配置文件
sudo cp 00-main.conf /etc/salt/master.d/
sudo cp 10-file_roots.conf /etc/salt/master.d/
sudo cp 20-tuning.conf /etc/salt/master.d/
sudo cp 30-security.conf /etc/salt/master.d/

# 设置权限
sudo chown -R root:root /etc/salt/master.d
sudo chmod -R 600 /etc/salt/master.d/*.conf

# 创建必要目录
sudo mkdir -p /srv/salt /srv/pillar
sudo mkdir -p /srv/salt/{dev,prod} /srv/pillar/{dev,prod}

# 验证配置
sudo salt-master --config-dir=/etc/salt -l debug --version

# 重启服务
sudo systemctl restart salt-master
sudo systemctl status salt-master
```

### Minion 配置部署脚本

```bash
#!/bin/bash
# deploy-minion-config.sh

# 创建配置目录
sudo mkdir -p /etc/salt/minion.d

# 复制配置文件
sudo cp 00-main.conf /etc/salt/minion.d/
sudo cp 10-network.conf /etc/salt/minion.d/
sudo cp 20-grains.conf /etc/salt/minion.d/
sudo cp 30-performance.conf /etc/salt/minion.d/

# 设置权限
sudo chown -R root:root /etc/salt/minion.d
sudo chmod -R 600 /etc/salt/minion.d/*.conf

# 验证配置
sudo salt-minion --config-dir=/etc/salt -l debug --version

# 重启服务
sudo systemctl restart salt-minion
sudo systemctl status salt-minion
```

## 配置验证命令

```bash
# 验证 Master 配置
sudo salt-master --config-dir=/etc/salt -l info --version

# 验证 Minion 配置
sudo salt-minion --config-dir=/etc/salt -l info --version

# 测试连接
sudo salt-key -L
sudo salt '*' test.ping

# 检查 Grains
sudo salt '*' grains.items

# 检查服务状态
sudo systemctl status salt-master
sudo systemctl status salt-minion
```
