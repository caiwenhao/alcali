# Salt Stack 高频使用的 Function 详解

## 概述

本文档详细介绍了 Salt Stack 中最常用的函数，这些函数是日常运维和配置管理的核心工具。基于 Alcali 项目的实际使用情况和 Salt Stack 官方文档整理。

## 🚀 超高频核心函数

### 1. test 模块 - 连接和基础测试

#### test.ping
**用途**: 测试 minion 连接状态
```bash
salt '*' test.ping
```
**返回**: True/False，表示连接状态
**使用场景**: 健康检查、连接验证

#### test.version
**用途**: 查看 Salt 版本信息
```bash
salt '*' test.version
```
**返回**: Salt 版本号
**使用场景**: 版本管理、兼容性检查

#### test.echo
**用途**: 回显测试，验证参数传递
```bash
salt '*' test.echo 'Hello World'
```
**返回**: 传入的参数
**使用场景**: 调试、参数验证

### 2. cmd 模块 - 命令执行

#### cmd.run
**用途**: 执行 shell 命令（使用频率极高）
```bash
salt '*' cmd.run 'uptime'
salt '*' cmd.run 'systemctl status nginx'
```
**返回**: 命令输出
**注意**: 谨慎使用，确保命令安全性

#### cmd.run_all
**用途**: 执行命令并返回详细信息
```bash
salt '*' cmd.run_all 'ls -la /etc'
```
**返回**: 包含 stdout、stderr、retcode 的字典
**使用场景**: 需要详细执行结果时

#### cmd.which
**用途**: 查找程序路径
```bash
salt '*' cmd.which python3
```
**返回**: 程序的完整路径
**使用场景**: 环境检查、路径验证

### 3. grains 模块 - 系统信息

#### grains.items
**用途**: 获取所有系统信息（每次 minion 刷新都会用到）
```bash
salt '*' grains.items
```
**返回**: 完整的系统信息字典
**使用场景**: 系统信息收集、资产管理

#### grains.item
**用途**: 获取特定系统信息
```bash
salt '*' grains.item os
salt '*' grains.item cpu_model
```
**返回**: 指定的 grain 值
**使用场景**: 有针对性的信息查询

#### grains.get
**用途**: 获取指定 grain 值（支持默认值）
```bash
salt '*' grains.get 'os' 'Unknown'
```
**返回**: grain 值或默认值
**使用场景**: 安全的信息获取

### 4. pillar 模块 - 配置数据

#### pillar.items
**用途**: 获取所有 pillar 配置数据
```bash
salt '*' pillar.items
```
**返回**: 完整的 pillar 数据
**使用场景**: 配置验证、调试

#### pillar.get
**用途**: 获取特定 pillar 值
```bash
salt '*' pillar.get 'nginx:port' 80
```
**返回**: pillar 值或默认值
**使用场景**: 配置参数获取

## 📦 高频系统管理函数

### 5. pkg 模块 - 包管理

#### pkg.install
**用途**: 安装软件包（跨平台自动适配）
```bash
salt '*' pkg.install nginx
salt '*' pkg.install 'nginx,vim,htop'
```
**返回**: 安装结果
**支持**: yum、apt、zypper 等

#### pkg.remove
**用途**: 卸载软件包
```bash
salt '*' pkg.remove nginx
```
**返回**: 卸载结果
**注意**: 谨慎操作，可能影响系统

#### pkg.list_pkgs
**用途**: 列出已安装的包
```bash
salt '*' pkg.list_pkgs
```
**返回**: 包名和版本的字典
**使用场景**: 软件清单、版本管理

#### pkg.upgrade
**用途**: 升级所有包
```bash
salt '*' pkg.upgrade
```
**返回**: 升级结果
**注意**: 可能需要较长时间

### 6. service 模块 - 服务管理

#### service.start
**用途**: 启动服务
```bash
salt '*' service.start nginx
```
**返回**: 启动结果（True/False）
**使用场景**: 服务启动

#### service.stop
**用途**: 停止服务
```bash
salt '*' service.stop nginx
```
**返回**: 停止结果
**使用场景**: 服务停止

#### service.restart
**用途**: 重启服务
```bash
salt '*' service.restart nginx
```
**返回**: 重启结果
**使用场景**: 配置更新后重启

#### service.status
**用途**: 查看服务状态
```bash
salt '*' service.status nginx
```
**返回**: 服务状态（True/False）
**使用场景**: 服务监控

#### service.enable
**用途**: 启用服务自启动
```bash
salt '*' service.enable nginx
```
**返回**: 操作结果
**使用场景**: 服务持久化配置

### 7. state 模块 - 状态管理

#### state.apply
**用途**: 应用状态配置（配置管理核心）
```bash
salt '*' state.apply
salt '*' state.apply webserver
salt '*' state.apply webserver test=True
```
**返回**: 状态执行结果
**支持**: test=True 测试模式

#### state.show_highstate
**用途**: 显示 highstate 配置
```bash
salt '*' state.show_highstate
```
**返回**: 将要应用的状态配置
**使用场景**: 配置预览、调试

#### state.test
**用途**: 测试状态（不实际执行）
```bash
salt '*' state.test
```
**返回**: 测试结果
**使用场景**: 安全的配置验证

## 🗂️ 文件系统函数

### 8. file 模块 - 文件操作

#### file.copy
**用途**: 复制文件
```bash
salt '*' file.copy /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
```
**返回**: 操作结果
**使用场景**: 文件备份、复制

#### file.exists
**用途**: 检查文件是否存在
```bash
salt '*' file.exists /etc/nginx/nginx.conf
```
**返回**: True/False
**使用场景**: 文件验证

#### file.remove
**用途**: 删除文件
```bash
salt '*' file.remove /tmp/tempfile
```
**返回**: 操作结果
**注意**: 谨慎操作，删除不可恢复

### 9. cp 模块 - Salt 文件服务器

#### cp.get_file
**用途**: 从 Salt 文件服务器获取文件
```bash
salt '*' cp.get_file salt://nginx/nginx.conf /etc/nginx/nginx.conf
```
**返回**: 操作结果
**使用场景**: 配置文件分发

#### cp.list_master
**用途**: 列出 master 上的文件
```bash
salt '*' cp.list_master
```
**返回**: 文件列表
**使用场景**: 文件服务器内容查看

## 🌐 网络和系统信息

### 10. network 模块 - 网络信息

#### network.interfaces
**用途**: 获取网络接口信息
```bash
salt '*' network.interfaces
```
**返回**: 网络接口详细信息
**使用场景**: 网络配置、故障排查

#### network.ip_addrs
**用途**: 获取 IP 地址列表
```bash
salt '*' network.ip_addrs
```
**返回**: IP 地址列表
**使用场景**: 网络信息收集

### 11. disk 模块 - 磁盘信息

#### disk.usage
**用途**: 磁盘使用情况
```bash
salt '*' disk.usage
salt '*' disk.usage /var
```
**返回**: 磁盘使用统计
**使用场景**: 磁盘监控、容量规划
**注意**: 不支持 test=True 参数

#### disk.percent
**用途**: 磁盘使用百分比
```bash
salt '*' disk.percent /
```
**返回**: 使用百分比
**使用场景**: 简单的磁盘监控

### 12. status 模块 - 系统状态

#### status.uptime
**用途**: 系统运行时间
```bash
salt '*' status.uptime
```
**返回**: 系统运行时间信息
**使用场景**: 系统监控

#### status.loadavg
**用途**: 系统负载
```bash
salt '*' status.loadavg
```
**返回**: 系统负载平均值
**使用场景**: 性能监控

## 🔧 管理和调试函数

### 13. sys 模块 - 系统函数

#### sys.doc
**用途**: 查看函数文档
```bash
salt '*' sys.doc
salt '*' sys.doc pkg.install
```
**返回**: 函数文档
**使用场景**: 学习、调试

#### sys.list_functions
**用途**: 列出所有可用函数
```bash
salt '*' sys.list_functions
```
**返回**: 函数列表
**使用场景**: 功能发现

### 14. saltutil 模块 - Salt 工具

#### saltutil.sync_all
**用途**: 同步所有自定义模块
```bash
salt '*' saltutil.sync_all
```
**返回**: 同步结果
**使用场景**: 模块更新

#### saltutil.refresh_pillar
**用途**: 刷新 pillar 数据
```bash
salt '*' saltutil.refresh_pillar
```
**返回**: 刷新结果
**使用场景**: 配置更新

## 🎯 Runner 和 Wheel 函数

### 15. Runner 函数（在 master 端执行）

#### jobs.active
**用途**: 查看活动作业
```bash
salt --client=runner jobs.active
```
**使用场景**: 作业监控

#### manage.up
**用途**: 查看在线 minions
```bash
salt --client=runner manage.up
```
**使用场景**: 节点状态监控

### 16. Wheel 函数（密钥管理）

#### key.list_all
**用途**: 列出所有密钥
```bash
salt --client=wheel key.list_all
```
**使用场景**: 密钥管理

#### key.accept
**用途**: 接受 minion 密钥
```bash
salt --client=wheel key.accept minion_id
```
**使用场景**: 新节点接入

## 📅 计划任务函数

### 17. schedule 模块

#### schedule.add
**用途**: 添加计划任务
```bash
salt '*' schedule.add job1 function='test.ping' seconds=3600
```
**使用场景**: 定时任务管理

#### schedule.list
**用途**: 列出计划任务
```bash
salt '*' schedule.list
```
**使用场景**: 任务查看

## 💡 使用建议和最佳实践

### 日常运维场景
1. **健康检查**: `test.ping`, `grains.items`, `status.uptime`
2. **软件管理**: `pkg.install`, `pkg.upgrade`, `service.restart`
3. **配置部署**: `state.apply`, `state.test`
4. **故障排查**: `cmd.run`, `network.interfaces`, `disk.usage`
5. **信息收集**: `pillar.items`, `sys.doc`

### 函数类型区分

#### Execution Modules（执行模块）- 不支持 test=True
- `disk.usage`, `network.interfaces`, `grains.items`
- `pillar.items`, `cmd.run`, `test.ping`

#### State Functions（状态函数）- 支持 test=True
- `state.apply`, `pkg.installed`, `service.running`
- `file.managed`, `user.present`

### 安全注意事项
1. **谨慎使用 cmd.run**: 确保命令安全性
2. **测试优先**: 使用 `test=True` 预览状态变更
3. **权限控制**: 合理配置 ACL 和用户权限
4. **备份重要**: 重要操作前先备份

### Alcali 中的高频使用
根据项目代码分析，Alcali 中最常用的函数：
- `test.ping` - 连接测试
- `grains.items` - 系统信息收集  
- `pillar.items` - 配置数据获取
- `state.apply` - 状态应用
- `cmd.run` - 命令执行

## 结语

这些函数构成了 Salt Stack 日常运维的核心工具集。掌握它们可以解决大部分基础设施管理需求。建议从基础函数开始学习，逐步掌握高级功能。

更多详细信息请参考 [Salt Stack 官方文档](https://docs.saltproject.io/)。
