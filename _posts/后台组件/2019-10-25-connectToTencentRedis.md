---
layout: post
title: 腾讯云Redis不能通过外网访问
category: 后台组件
tags: Redis
keywords: Redis
---
## 通过端口转发访问腾讯云Redis
- 本地访问腾讯云CVM的端口，CVM通过内网转发至腾讯云Redis

### 使用firewall进行端口转发具体命令
1. 检查是否允许伪装ip: firewall-cmd --query-masquerade
2. 永久允许防火墙伪装ip：firewall-cmd --add-masquerade --permanent
3. 永久添加映射规则：firewall-cmd --add-port=6379/tcp --permanent
4. 重新加载生效：firewall-cmd --reload
5. firewall-cmd --list-ports 查看打开的端口
6. 永久添加映射规则：firewall-cmd --add-forward-port=port=6379:proto=tcp:toaddr=172.27.0.5:toport=6379 --permanent
7. 重新加载生效：firewall-cmd --reload
8. firewall-cmd --list-forward 查看映射规则






