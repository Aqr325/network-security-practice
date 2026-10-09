# 01 - 网络基础

> OSI 七层模型 & TCP/IP 协议族速记

## OSI 七层（自下而上）

| 层 | 名称 | 核心概念 |
|----|------|----------|
| 7 | 应用层 | HTTP、DNS、SMTP、FTP |
| 6 | 表示层 | 数据格式、加密 |
| 5 | 会话层 | 连接管理 |
| 4 | 传输层 | TCP / UDP、端口号 |
| 3 | 网络层 | IP、路由、ICMP |
| 2 | 数据链路层 | MAC 地址、交换机、VLAN |
| 1 | 物理层 | 比特流、网线、光信号 |

## 关键 TCP 状态

`LISTEN → SYN_SENT → SYN_RECV → ESTABLISHED → FIN_WAIT_1/2 → TIME_WAIT → CLOSED`

## 动手实验（DVWA）

- [ ] 抓包：`tcpdump -i eth0 -nn port 80`
- [ ] 分析 HTTP 请求/响应头
- [ ] 理解三次握手 / 四次挥手
