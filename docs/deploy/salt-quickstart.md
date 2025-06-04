# Salt 快速开始指南

本指南提供了在公网环境中快速部署 Salt Master 和 Minion 的简化步骤。

## 前提条件

- 具有公网 IP 的 Linux 服务器（Master）
- 一台或多台需要管理的服务器（Minion）
- 服务器间网络连通性
- sudo 权限

## 第一步：准备 Salt Master

### 1. 安装 Salt Master

**CentOS/RHEL:**
```bash
# 添加 Salt 仓库
sudo rpm --import https://repo.saltproject.io/py3/redhat/8/x86_64/latest/SALTSTACK-GPG-KEY.pub
curl -fsSL https://repo.saltproject.io/py3/redhat/8/x86_64/latest.repo | sudo tee /etc/yum.repos.d/salt.repo

# 安装 Salt Master
sudo yum install -y salt-master salt-minion salt-ssh salt-syndic salt-cloud
```

**Ubuntu/Debian:**
```bash
# 添加 Salt 仓库
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://packages.broadcom.com/artifactory/api/security/keypair/SaltProjectKey/public | sudo tee /etc/apt/keyrings/salt-archive-keyring.pgp
curl -fsSL https://github.com/saltstack/salt-install-guide/releases/latest/download/salt.sources | sudo tee /etc/apt/sources.list.d/salt.sources

# 更新包列表并安装
sudo apt-get update
sudo apt-get install -y salt-master salt-minion salt-ssh salt-syndic salt-cloud salt-api
```

### 2. 配置防火墙

```bash
# CentOS/RHEL (firewalld)
sudo firewall-cmd --permanent --add-port=4505/tcp
sudo firewall-cmd --permanent --add-port=4506/tcp
sudo firewall-cmd --reload

# Ubuntu (ufw)
sudo ufw allow 4505/tcp
sudo ufw allow 4506/tcp
sudo ufw reload
```

### 3. 快速配置 Master

创建基础配置：

```bash
# 创建配置目录
sudo mkdir -p /etc/salt/master.d /srv/salt /srv/pillar

# 创建基础配置文件
sudo tee /etc/salt/master.d/00-main.conf << 'EOF'
# 基础配置
interface: 0.0.0.0
auto_accept: False
log_level: warning

# 文件根目录
file_roots:
  base:
    - /srv/salt

pillar_roots:
  base:
    - /srv/pillar

# 性能调优
worker_threads: 8
EOF

# 设置权限
sudo chmod 600 /etc/salt/master.d/00-main.conf
```

### 4. 启动 Master 服务

```bash
sudo systemctl enable salt-master
sudo systemctl start salt-master
sudo systemctl status salt-master
```

### 5. 验证 Master 状态

```bash
# 检查端口监听
sudo ss -tlnp | grep -E ':(4505|4506)'

# 检查服务日志
sudo journalctl -u salt-master --no-pager -l
```

## 第二步：准备 Salt Minion

### 1. 安装 Salt Minion

在每台需要管理的服务器上执行：

**CentOS/RHEL:**
```bash
sudo rpm --import https://repo.saltproject.io/py3/redhat/8/x86_64/latest/SALTSTACK-GPG-KEY.pub
curl -fsSL https://repo.saltproject.io/py3/redhat/8/x86_64/latest.repo | sudo tee /etc/yum.repos.d/salt.repo
sudo yum install -y salt-minion
```

**Ubuntu/Debian:**
```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://packages.broadcom.com/artifactory/api/security/keypair/SaltProjectKey/public | sudo tee /etc/apt/keyrings/salt-archive-keyring.pgp
curl -fsSL https://github.com/saltstack/salt-install-guide/releases/latest/download/salt.sources | sudo tee /etc/apt/sources.list.d/salt.sources
sudo apt-get update
sudo apt-get install -y salt-minion
```

### 2. 配置 Minion

```bash
# 创建配置目录
sudo mkdir -p /etc/salt/minion.d

# 创建基础配置（替换 YOUR_MASTER_IP_OR_DOMAIN）
sudo tee /etc/salt/minion.d/00-main.conf << 'EOF'
# Master 地址
master: YOUR_MASTER_IP_OR_DOMAIN

# Minion ID（可选，默认使用主机名）
# id: my-custom-minion-id

# 日志配置
log_level: warning

# 连接设置
retry_dns: 30
master_tries: -1
EOF

# 设置权限
sudo chmod 600 /etc/salt/minion.d/00-main.conf
```

**重要：** 将 `YOUR_MASTER_IP_OR_DOMAIN` 替换为实际的 Master 服务器 IP 地址或域名。

### 3. 启动 Minion 服务

```bash
sudo systemctl enable salt-minion
sudo systemctl start salt-minion
sudo systemctl status salt-minion
```

### 4. 验证 Minion 连接

```bash
# 检查服务日志
sudo journalctl -u salt-minion --no-pager -l

# 测试到 Master 的连接
nc -zv YOUR_MASTER_IP_OR_DOMAIN 4505
nc -zv YOUR_MASTER_IP_OR_DOMAIN 4506
```

## 第三步：建立连接

### 1. 在 Master 上查看待接受的密钥

```bash
sudo salt-key -L
```

你应该看到类似输出：
```
Accepted Keys:
Denied Keys:
Unaccepted Keys:
    minion-hostname
Rejected Keys:
```

### 2. 接受 Minion 密钥

```bash
# 接受单个密钥
sudo salt-key -a minion-hostname

# 或接受所有待处理密钥（谨慎使用）
sudo salt-key -A
```

### 3. 测试连接

```bash
# 测试所有 Minion
sudo salt '*' test.ping

# 测试特定 Minion
sudo salt 'minion-hostname' test.ping
```

成功的输出应该是：
```
minion-hostname:
    True
```

## 第四步：基础管理命令

### 系统信息收集

```bash
# 查看所有 Minion 状态
sudo salt '*' test.ping

# 获取系统信息
sudo salt '*' grains.items

# 查看运行时间
sudo salt '*' cmd.run 'uptime'

# 检查磁盘使用
sudo salt '*' cmd.run 'df -h'

# 查看内存使用
sudo salt '*' cmd.run 'free -h'
```

### 包管理

```bash
# 更新包列表
sudo salt '*' pkg.refresh_db

# 安装软件包
sudo salt '*' pkg.install vim

# 查看已安装包
sudo salt '*' pkg.list_pkgs
```

### 服务管理

```bash
# 查看服务状态
sudo salt '*' service.status sshd

# 启动服务
sudo salt '*' service.start httpd

# 停止服务
sudo salt '*' service.stop httpd

# 重启服务
sudo salt '*' service.restart httpd
```

## 故障排除

### 常见问题

**1. Minion 无法连接到 Master**
```bash
# 检查网络连接
ping YOUR_MASTER_IP_OR_DOMAIN
telnet YOUR_MASTER_IP_OR_DOMAIN 4505

# 检查防火墙
sudo iptables -L -n | grep -E '4505|4506'

# 查看 Minion 日志
sudo journalctl -u salt-minion -f
```

**2. 密钥问题**
```bash
# 在 Master 上删除有问题的密钥
sudo salt-key -d minion-hostname

# 在 Minion 上重新生成密钥
sudo systemctl stop salt-minion
sudo rm -rf /etc/salt/pki/minion/
sudo systemctl start salt-minion
```

**3. 配置语法错误**
```bash
# 验证配置文件
sudo salt-master --config-dir=/etc/salt -l debug --version
sudo salt-minion --config-dir=/etc/salt -l debug --version
```

### 有用的日志命令

```bash
# 实时查看 Master 日志
sudo journalctl -u salt-master -f

# 实时查看 Minion 日志
sudo journalctl -u salt-minion -f

# 查看最近的错误
sudo journalctl -u salt-master --since "1 hour ago" | grep -i error
```

## 下一步

1. **学习 Salt States**: 创建可重复的配置管理
2. **配置 Pillar**: 管理敏感数据和配置变量
3. **设置 Grains**: 自定义系统信息用于目标选择
4. **实施安全措施**: 配置外部认证和访问控制
5. **性能优化**: 根据环境调整配置参数

## 参考资源

- [Salt 官方文档](https://docs.saltproject.io/)
- [Salt 最佳实践指南](./salt-deploy.md)
- [Salt 配置示例](./salt-config-examples.md)
