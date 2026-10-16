# 动手学网络安全（Hands-On Network Security Practice）

> 实战驱动的安全学习仓库。每周一推送，每周覆盖 3 个方向。

## 五大方向

| 方向 | 目录 | 核心内容 |
|------|------|----------|
| **Web 渗透** | `web-pentest/` | OWASP Top 10、SQLi、XSS、SSRF、Post-Exploitation |
| **PWN 二进制** | `pwn/` | 栈溢出、ROP、格式化字符串、栈迁移、内核利用 |
| **密码学** | `crypto/` | 对称/非对称、分组密码、哈希、侧信道 |
| **杂项** | `misc/` | 隐写、编码、Steganography、Forensics、AI 杂题 |
| **逆向** | `reverse/` | Ghidra / IDA Pro、汇编、脱壳、游戏逆向 |

## 目录结构

```
.
├── README.md
├── web-pentest/
│   ├── README.md
│   ├── cheatsheets/   # 命令/速查表
│   ├── scripts/       # 自动化工具脚本
│   └── tools/         # 工具配置与用法
├── pwn/
│   ├── README.md
│   ├── exploits/      # 可运行的 exploit 脚本
│   └── analysis/      # 二进制分析与 writeup
├── crypto/
│   ├── README.md
│   └── labs/          # 密码学实验与解题脚本
├── misc/
│   ├── README.md
│   └── catchme/       # 杂项 CTF 题目与 writeup
└── reverse/
    ├── README.md
    └── labs/          # 逆向实验与 Ghidra 工程文件
```

## 推送节奏

- 每周固定推送 **3 个方向** 的知识文档
- 每个文档 = 一个知识点 + 可复现实验/代码
- 按周编号：`2026-W40-01.md`（ISO 周号 + 周序号）

## 命名规范

| 类型 | 格式 | 示例 |
|------|------|------|
| 知识文档 | `YYYY-WW-NN-{主题}.md` | `2026-W40-01-sqli-blind.md` |
| Exploit 脚本 | `{编号}-{名称}.py` | `01-overflow.py` |
| 速查表 | `{主题}-cheatsheet.md` | `sqlmap-cheatsheet.md` |

> ⚠️ **伦理边界**：所有实验仅限 DVWA / 自建靶机 / CTF 平台 / 授权资产，严禁对未授权系统实施攻击。
