# Salt Grains 和 FQDN 详解

本文档详细解释了 Salt 中的 Grains 概念，特别是 FQDN 的使用和 Minion ID 的配置选项。

## 什么是 FQDN？

**FQDN** = **Fully Qualified Domain Name**（完全限定域名）

### FQDN 的组成

```
hostname.subdomain.domain.tld
   ↓        ↓        ↓     ↓
 web01   .internal .company .com
```

**完整示例**:
- `web01.internal.company.com`
- `db-server.prod.example.org`
- `mail.google.com`

### 查看系统 FQDN

```bash
# 方法 1: 使用 hostname 命令
hostname -f

# 方法 2: 使用 hostnamectl
hostnamectl

# 方法 3: 在 Salt 中查看
sudo salt-call grains.get fqdn

# 方法 4: 查看 /etc/hosts 文件
cat /etc/hosts
```

**示例输出**:
```bash
$ hostname -f
web01.internal.company.com

$ hostnamectl
   Static hostname: web01
         Icon name: computer-vm
           Chassis: vm
        Machine ID: 12345678901234567890123456789012
           Boot ID: abcdef12-3456-7890-abcd-ef1234567890
    Virtualization: kvm
  Operating System: Ubuntu 20.04.3 LTS
            Kernel: Linux 5.4.0-91-generic
      Architecture: x86-64
```

## Salt Grains 详解

### 什么是 Grains？

Grains 是 Salt 中的静态信息系统，包含了关于 Minion 的各种系统信息。

### 常用的 Grains

```bash
# 查看所有 grains
sudo salt-call grains.items

# 查看特定 grains
sudo salt-call grains.get fqdn          # 完全限定域名
sudo salt-call grains.get host          # 主机名
sudo salt-call grains.get domain        # 域名
sudo salt-call grains.get nodename      # 节点名
sudo salt-call grains.get ip4_interfaces # IPv4 接口
sudo salt-call grains.get os            # 操作系统
sudo salt-call grains.get osrelease     # 操作系统版本
sudo salt-call grains.get kernel        # 内核版本
sudo salt-call grains.get mem_total     # 总内存
sudo salt-call grains.get num_cpus      # CPU 核心数
```

### Grains 示例输出

```yaml
fqdn: web01.internal.company.com
host: web01
domain: internal.company.com
nodename: web01
ip4_interfaces:
  eth0:
    - 192.168.1.100
  lo:
    - 127.0.0.1
os: Ubuntu
osrelease: 20.04
kernel: Linux
mem_total: 8192
num_cpus: 4
```

## Minion ID 配置选项

### 1. 使用 FQDN（推荐）

```yaml
# /etc/salt/minion.d/00-main.conf
id: {{ grains['fqdn'] }}
```

**优点**:
- 全局唯一性
- 包含域信息
- 便于识别服务器位置

**缺点**:
- 需要正确的 DNS 配置
- 可能较长

**适用场景**:
- 有完整 DNS 基础设施的环境
- 多域名环境
- 需要明确标识服务器位置

### 2. 使用主机名

```yaml
id: {{ grains['host'] }}
```

**优点**:
- 简洁明了
- 易于记忆

**缺点**:
- 可能不唯一（不同域的同名主机）
- 缺少域信息

**适用场景**:
- 单一域名环境
- 主机名已经具有唯一性

### 3. 使用自定义命名规则

```yaml
# 环境-角色-主机名
id: {{ grains['environment'] }}-{{ grains['role'] }}-{{ grains['host'] }}

# 数据中心-角色-序号
id: dc1-web-01

# 项目-环境-角色-序号
id: myapp-prod-web-01
```

**优点**:
- 高度可定制
- 包含业务信息
- 便于分类管理

**缺点**:
- 需要维护命名规范
- 可能需要手动设置

### 4. 使用 IP 地址（不推荐）

```yaml
id: {{ grains['ip4_interfaces']['eth0'][0] }}
```

**优点**:
- 绝对唯一
- 无需 DNS

**缺点**:
- IP 可能变化
- 不易记忆
- 缺少语义信息

**适用场景**:
- 临时环境
- 无 DNS 环境
- 容器环境（谨慎使用）

## 配置 FQDN 的最佳实践

### 1. 确保正确的主机名设置

```bash
# 设置主机名
sudo hostnamectl set-hostname web01.internal.company.com

# 或者编辑 /etc/hostname
echo "web01.internal.company.com" | sudo tee /etc/hostname

# 更新 /etc/hosts
sudo tee -a /etc/hosts << EOF
127.0.0.1 web01.internal.company.com web01
EOF
```

### 2. 验证 DNS 解析

```bash
# 正向解析
nslookup web01.internal.company.com

# 反向解析
nslookup 192.168.1.100

# 测试解析
dig web01.internal.company.com
```

### 3. 重启网络服务

```bash
# Ubuntu/Debian
sudo systemctl restart systemd-resolved

# CentOS/RHEL
sudo systemctl restart NetworkManager
```

## 自定义 Grains

### 创建自定义 Grains

**方法 1: 在配置文件中定义**

```yaml
# /etc/salt/minion.d/20-grains.conf
grains:
  environment: production
  role: webserver
  datacenter: us-east-1
  team: devops
  project: myapp
```

**方法 2: 使用 grains 文件**

```yaml
# /etc/salt/grains
environment: production
role: webserver
datacenter: us-east-1
team: devops
project: myapp
```

**方法 3: 使用 Python 脚本**

```python
# /etc/salt/grains.d/custom.py
def custom_grains():
    grains = {}
    
    # 根据主机名确定角色
    import socket
    hostname = socket.gethostname()
    
    if 'web' in hostname:
        grains['role'] = 'webserver'
    elif 'db' in hostname:
        grains['role'] = 'database'
    elif 'lb' in hostname:
        grains['role'] = 'loadbalancer'
    
    # 根据 IP 确定数据中心
    import subprocess
    ip = subprocess.check_output(['hostname', '-I']).decode().strip().split()[0]
    
    if ip.startswith('10.1.'):
        grains['datacenter'] = 'us-east-1'
    elif ip.startswith('10.2.'):
        grains['datacenter'] = 'us-west-1'
    
    return grains
```

### 使用自定义 Grains 创建 Minion ID

```yaml
# 使用多个 grains 组合
id: {{ grains['datacenter'] }}-{{ grains['role'] }}-{{ grains['host'] }}

# 结果示例: us-east-1-webserver-web01
```

## 故障排除

### 常见问题

**1. FQDN 返回 localhost 或不正确的值**

```bash
# 检查 /etc/hosts
cat /etc/hosts

# 应该包含类似这样的条目:
# 127.0.0.1 localhost
# 192.168.1.100 web01.internal.company.com web01

# 修复方法
sudo hostnamectl set-hostname web01.internal.company.com
```

**2. Grains 信息不更新**

```bash
# 刷新 grains 缓存
sudo salt-call saltutil.refresh_grains

# 重启 minion 服务
sudo systemctl restart salt-minion
```

**3. 自定义 grains 不生效**

```bash
# 检查语法
python3 -c "import yaml; yaml.safe_load(open('/etc/salt/grains'))"

# 检查权限
ls -la /etc/salt/grains

# 重新加载配置
sudo salt-call saltutil.refresh_grains
```

### 调试命令

```bash
# 查看当前 Minion ID
sudo salt-call grains.get id

# 查看所有网络相关 grains
sudo salt-call grains.get ip_interfaces
sudo salt-call grains.get fqdn_ip4
sudo salt-call grains.get fqdn_ip6

# 测试模板渲染
sudo salt-call grains.get fqdn --out=json

# 查看配置文件加载情况
sudo salt-minion --config-dir=/etc/salt -l debug --version
```

## 总结

- **FQDN** 是完全限定域名，提供全局唯一的主机标识
- **Grains** 是 Salt 的系统信息收集机制
- **Minion ID** 应该根据环境需求选择合适的命名策略
- **自定义 Grains** 可以提供额外的业务信息用于目标选择和配置管理

选择合适的 Minion ID 策略对于 Salt 环境的可维护性和可扩展性至关重要。
