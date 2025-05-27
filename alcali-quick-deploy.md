# Alcali 快速部署指南

## 1. 克隆项目并启动

```bash
# 克隆项目
git clone https://github.com/latenighttales/alcali.git
cd alcali

# 一键启动（包含2个minion节点）
docker compose up --scale minion=2
```

## 2. 等待服务启动

当看到以下日志时，说明服务已准备就绪：
```
minion_1  | [ERROR   ] The Salt Master has cached the public key for this node, this salt minion will wait for 10 seconds before attempting to re-authenticate
minion_1  | [INFO    ] Waiting 10 seconds before retry.
```

## 3. 访问Web界面

- 访问地址: http://127.0.0.1:8000
- 默认用户名: admin
- 默认密码: password

## 4. 快速配置

1. 登录后进入Keys页面接受minion密钥
2. 进入Minions页面刷新minion信息
3. 进入Settings页面配置minion字段和合规性检查
4. 开始使用Alcali管理Salt环境

## 服务组件

- **Web服务**: http://localhost:8000 (Alcali主界面)
- **Salt-API**: https://localhost:8080 (Salt API服务)
- **数据库**: localhost:3306 (MariaDB)
- **Salt Master**: 容器内部
- **Salt Minions**: 2个测试节点

## 停止服务

```bash
docker compose down
```

## 数据持久化

默认配置下，数据存储在Docker卷中。如需持久化数据，请修改docker-compose.yml添加卷映射。
