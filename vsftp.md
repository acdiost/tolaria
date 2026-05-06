---
type: Note
tags:
  - linux
  - ftp
  - vsftpd
  - rocky-linux
  - 运维
---

# Rocky Linux 9 vsftpd 安装配置全攻略

## 1. 简介

vsftpd（Very Secure FTP Daemon）是 Linux 系统上最常用的 FTP 服务器软件之一，以安全性高、性能稳定、配置灵活著称。本文将在 Rocky Linux 9 环境下，从零开始完整演示 vsftpd 的安装、配置与日常使用。

---

## 2. 环境说明

| 项目 | 版本 |
|------|------|
| 操作系统 | Rocky Linux 9.x |
| vsftpd | 3.0.5 |
| 防火墙 | firewalld |
| SELinux | Enforcing（按需调整）|

---

## 3. 安装 vsftpd

### 3.1 更新系统并安装

```bash
sudo dnf update -y
sudo dnf install -y vsftpd
```

### 3.2 设置开机自启并立即启动

```bash
sudo systemctl enable --now vsftpd
```

### 3.3 验证服务状态

```bash
sudo systemctl status vsftpd
```

输出中看到 `active (running)` 即表示服务正常运行。

---

## 4. 防火墙配置

Rocky Linux 9 默认启用 `firewalld`，需要放行 FTP 相关端口。

```bash
# 放行 FTP 服务（端口 21）
sudo firewall-cmd --permanent --add-service=ftp

# 若使用被动模式（PASV），还需放行被动端口范围（此处示例 30000-31000）
sudo firewall-cmd --permanent --add-port=30000-31000/tcp

# 重载防火墙规则
sudo firewall-cmd --reload

# 验证规则
sudo firewall-cmd --list-all
```

---

## 5. 核心配置文件详解

vsftpd 的主配置文件位于 `/etc/vsftpd/vsftpd.conf`。

### 5.1 备份原始配置

```bash
sudo cp /etc/vsftpd/vsftpd.conf /etc/vsftpd/vsftpd.conf.bak
```

### 5.2 典型生产配置示例

编辑 `/etc/vsftpd/vsftpd.conf`，按需修改以下关键参数：

```ini
# ── 基础设置 ──────────────────────────────
# 禁止匿名登录（生产环境必须关闭）
anonymous_enable=NO

# 允许本地系统用户登录
local_enable=YES

# 允许写入操作（上传、删除、改名）
write_enable=YES

# 本地用户文件权限掩码（022 表示上传文件权限为 644）
local_umask=022

# ── 目录限制 ──────────────────────────────
# 将用户锁定在其 home 目录，防止越权访问
chroot_local_user=YES

# 锁定后允许 chroot 目录本身可写（需配合下面一行）
allow_writeable_chroot=YES

# ── 被动模式（PASV）─────────────────────
pasv_enable=YES
pasv_min_port=30000
pasv_max_port=31000

# 服务器对外的公网 IP（云服务器必须填写）
# pasv_address=YOUR_PUBLIC_IP

# ── 日志 ─────────────────────────────────
xferlog_enable=YES
xferlog_file=/var/log/vsftpd.log
xferlog_std_format=YES

# ── 连接设置 ──────────────────────────────
# 最大客户端连接数
max_clients=100

# 同一 IP 最大并发连接数
max_per_ip=10

# 数据传输超时（秒）
data_connection_timeout=120

# ── 欢迎信息 ──────────────────────────────
ftpd_banner=Welcome to Rocky Linux 9 FTP Server.
```

### 5.3 重启服务使配置生效

```bash
sudo systemctl restart vsftpd
```

---

## 6. 创建 FTP 专用用户

不建议直接使用 root 或系统关键账号进行 FTP 操作，推荐创建独立 FTP 用户。

```bash
# 创建用户，指定 home 目录，禁止 SSH 登录
sudo useradd -m -d /home/ftpuser -s /sbin/nologin ftpuser

# 设置密码
sudo passwd ftpuser

# 确认 home 目录所有者
ls -la /home/ftpuser
```

若启用了 `chroot_local_user=YES`，需确保用户根目录（`/home/ftpuser`）**不可被该用户写入**（vsftpd 的安全要求）：

```bash
# 将 home 目录所有者改为 root，权限设为 755
sudo chown root:root /home/ftpuser
sudo chmod 755 /home/ftpuser

# 在其下创建一个可写子目录供用户实际存放文件
sudo mkdir /home/ftpuser/upload
sudo chown ftpuser:ftpuser /home/ftpuser/upload
```

---

## 7. SELinux 配置

Rocky Linux 9 默认开启 SELinux，需要额外配置才能让 vsftpd 正常工作。

```bash
# 允许 vsftpd 读取本地用户 home 目录
sudo setsebool -P ftp_home_dir on

# 若使用自定义数据目录，需设置正确的 SELinux 上下文
sudo semanage fcontext -a -t public_content_rw_t "/home/ftpuser/upload(/.*)?"
sudo restorecon -Rv /home/ftpuser/upload
```

> 如果测试阶段想临时关闭 SELinux 干扰，可执行 `sudo setenforce 0`（重启后恢复），**生产环境不建议永久关闭**。

---

## 8. 用户访问控制

### 8.1 黑名单模式（默认）

`/etc/vsftpd/ftpusers` 和 `/etc/vsftpd/user_list` 中列出的用户默认被**拒绝**登录。

```bash
# 查看被禁止的用户列表
cat /etc/vsftpd/user_list
```

### 8.2 白名单模式

在 `vsftpd.conf` 中开启白名单，只允许 `user_list` 中的用户登录：

```ini
userlist_enable=YES
userlist_deny=NO    # 改为 NO 表示白名单模式
```

然后将允许的用户名写入 `/etc/vsftpd/user_list`：

```bash
echo "ftpuser" | sudo tee -a /etc/vsftpd/user_list
```

---

## 9. 配置虚拟用户（进阶）

虚拟用户不是真实的系统账号，安全性更高，适合对外提供多账号 FTP 服务。

### 9.1 安装 PAM 相关工具

```bash
sudo dnf install -y libdb-utils
```

### 9.2 创建虚拟用户凭据文件

```bash
# 格式：奇数行为用户名，偶数行为密码
cat << 'EOF' | sudo tee /etc/vsftpd/vusers.txt
alice
alice_password
bob
bob_password
EOF
```

### 9.3 生成 Berkeley DB 数据库

```bash
sudo db_load -T -t hash -f /etc/vsftpd/vusers.txt /etc/vsftpd/vusers.db
sudo chmod 600 /etc/vsftpd/vusers.db
```

### 9.4 配置 PAM 认证

创建 `/etc/pam.d/vsftpd_virtual`：

```
auth    required pam_userdb.so db=/etc/vsftpd/vusers
account required pam_userdb.so db=/etc/vsftpd/vusers
```

### 9.5 在 vsftpd.conf 中启用虚拟用户

```ini
guest_enable=YES
guest_username=ftpuser          # 映射到的本地系统用户
pam_service_name=vsftpd_virtual
user_config_dir=/etc/vsftpd/vusers_conf
virtual_use_local_privs=YES
```

### 9.6 为每个虚拟用户创建独立配置（可选）

```bash
sudo mkdir -p /etc/vsftpd/vusers_conf

# alice 的单独配置
sudo tee /etc/vsftpd/vusers_conf/alice << 'EOF'
local_root=/home/ftpuser/alice
write_enable=YES
EOF

sudo mkdir -p /home/ftpuser/alice
sudo chown ftpuser:ftpuser /home/ftpuser/alice
```

重启服务：

```bash
sudo systemctl restart vsftpd
```

---

## 10. 常见问题排查

### 10.1 连接被拒绝

```bash
# 检查服务是否运行
sudo systemctl status vsftpd

# 检查端口是否监听
sudo ss -tlnp | grep :21

# 检查防火墙规则
sudo firewall-cmd --list-all
```

### 10.2 530 Login incorrect

- 确认用户名和密码正确
- 检查 `/etc/vsftpd/ftpusers` 和 `user_list` 是否包含该用户
- 查看 `/var/log/vsftpd.log` 和 `/var/log/secure` 获取详细错误

```bash
sudo tail -f /var/log/vsftpd.log
sudo tail -f /var/log/secure
```

### 10.3 500 OOPS: vsftpd: refusing to run with writable root inside chroot

chroot 根目录不能被登录用户写入，按第 6 节将目录权限改为 755、所有者改为 root。

### 10.4 被动模式无法传输数据

- 确认 `pasv_min_port` / `pasv_max_port` 范围已在防火墙开放
- 云服务器需要在 `vsftpd.conf` 中设置 `pasv_address` 为公网 IP
- 检查安全组是否放行对应端口范围

### 10.5 SELinux 导致的权限拒绝

```bash
# 查看 SELinux 拒绝日志
sudo ausearch -m avc -ts recent | audit2why
```

---

## 11. 安全加固建议

- 禁用匿名访问（`anonymous_enable=NO`）
- 使用 FTPS（显式 TLS）加密传输，避免明文密码

```ini
# 在 vsftpd.conf 末尾追加 TLS 配置
ssl_enable=YES
rsa_cert_file=/etc/ssl/certs/vsftpd.pem
rsa_private_key_file=/etc/ssl/private/vsftpd.key
allow_anon_ssl=NO
force_local_data_ssl=YES
force_local_logins_ssl=YES
ssl_tlsv1=YES
ssl_sslv2=NO
ssl_sslv3=NO
```

- 启用 `chroot_local_user` 将用户限制在家目录
- 限制 `max_clients` 和 `max_per_ip` 防止暴力破解
- 定期审查 `/var/log/vsftpd.log` 中的异常登录

---

## 12. 小结

| 步骤 | 关键命令 |
|------|---------|
| 安装 | `dnf install -y vsftpd` |
| 启动 | `systemctl enable --now vsftpd` |
| 开放端口 | `firewall-cmd --permanent --add-service=ftp` |
| 核心配置 | `/etc/vsftpd/vsftpd.conf` |
| SELinux | `setsebool -P ftp_home_dir on` |
| 查日志 | `tail -f /var/log/vsftpd.log` |

按照本文完成配置后，你将拥有一个安全可用的 FTP 服务，适合内网文件共享、持续集成构建产物分发等场景。如需对外提供服务，强烈建议配合 TLS 加密和严格的用户访问控制。
