---
type: Note
category: "[[linux-system-administration]]"
tags: [linux, identity-management, rocky-linux, sysadmin, freeipa, kerberos, ldap]
---
# FreeIPA 在 Rocky Linux 9 上的安装配置与使用指南

## 概述

FreeIPA 是一套集成了 LDAP（389 Directory Server）、Kerberos、DNS、NTP 和证书管理（Dogtag CA）的开源身份管理平台，功能类似 Microsoft Active Directory，适合在 Linux 环境中统一管理用户账号、主机、策略与认证。本文详细介绍如何在 Rocky Linux 9 上部署 FreeIPA Server 及 Client，并完成日常管理操作。

***

## 环境准备

### 系统要求

| 项目   | 建议配置                          |
| ---- | ----------------------------- |
| 操作系统 | Rocky Linux 9.x               |
| CPU  | 2 核及以上                        |
| 内存   | 4 GB 及以上（生产建议 8 GB）           |
| 磁盘   | 20 GB 可用空间                    |
| 主机名  | 完整 FQDN，如 `ipa.example.local` |
| 网络   | 静态 IP，DNS 可解析主机名              |

### 设置主机名与 hosts

FreeIPA 对主机名解析要求严格，务必先配置好：

```bash
# 设置主机名（替换为实际 FQDN）
sudo hostnamectl set-hostname ipa.example.local

# 编辑 /etc/hosts，确保正向与反向解析一致
sudo vim /etc/hosts
```

`/etc/hosts` 中添加：

```text
192.168.1.10  ipa.example.local  ipa
```

### 更新系统并重启

```bash
sudo dnf update -y
sudo reboot
```

### 开放防火墙端口

FreeIPA 需要以下服务端口：

```bash
sudo firewall-cmd --permanent --add-service={freeipa-ldap,freeipa-ldaps,freeipa-replication,dns,ntp,http,https,kerberos,kpasswd}
sudo firewall-cmd --reload
```

也可使用预置 `freeipa-4` 服务组（Rocky Linux 9）：

```bash
sudo firewall-cmd --permanent --add-service=freeipa-4
sudo firewall-cmd --reload
```

***

## 安装 FreeIPA Server

### 1. 安装软件包

```bash
sudo dnf install -y ipa-server ipa-server-dns
```

如需 AD 信任功能（跨 Windows 域），额外安装：

```bash
sudo dnf install -y ipa-server-trust-ad
```

### 2. 运行安装向导

```bash
sudo ipa-server-install
```

向导会交互式询问以下参数，也可使用参数直接传入（适合自动化）：

```bash
sudo ipa-server-install \
  --domain=example.local \
  --realm=EXAMPLE.LOCAL \
  --ds-password=DirManagerPassword \
  --admin-password=AdminPassword \
  --hostname=ipa.example.local \
  --ip-address=192.168.1.10 \
  --setup-dns \
  --no-forwarders \
  --unattended
```

| 参数                 | 说明                        |
| ------------------ | ------------------------- |
| `--domain`         | DNS 域名（小写）                |
| `--realm`          | Kerberos Realm（大写）        |
| `--ds-password`    | LDAP Directory Manager 密码 |
| `--admin-password` | IPA admin 账号密码            |
| `--setup-dns`      | 同时配置 FreeIPA 内置 DNS       |
| `--no-forwarders`  | 不使用上游 DNS 转发              |

安装完成后会输出：

```text
The ipa-server-install command was successful
```

### 3. 验证安装

```bash
# 获取 Kerberos 票据
kinit admin

# 查看 IPA 域信息
ipa env
```

***

## 安装 FreeIPA Client（客户端加域）

在需要加入 IPA 域的客户端机器上执行：

### 1. 安装客户端软件包

```bash
sudo dnf install -y ipa-client
```

### 2. 运行客户端注册

```bash
sudo ipa-client-install \
  --domain=example.local \
  --server=ipa.example.local \
  --principal=admin \
  --password=AdminPassword \
  --unattended
```

### 3. 验证客户端加域

```bash
# 获取 Kerberos 票据
kinit admin

# 确认能解析 IPA 服务
ipa user-find admin
```

***

## Web 管理界面

FreeIPA 提供基于浏览器的 Web UI，访问地址：

```yaml
https://ipa.example.local/ipa/ui/
```

使用 `admin` 账号登录。初次访问需要将 IPA 的根证书导入浏览器，或忽略证书警告（仅测试环境）。

***

## 用户管理

### 创建用户

```bash
ipa user-add zhangsan \
  --first=San \
  --last=Zhang \
  --email=zhangsan@example.local \
  --password
```

### 查看用户

```bash
ipa user-find
ipa user-show zhangsan
```

### 修改用户

```bash
ipa user-mod zhangsan --title="Engineer"
```

### 禁用 / 启用用户

```bash
ipa user-disable zhangsan
ipa user-enable zhangsan
```

### 删除用户

```bash
ipa user-del zhangsan
```

***

## 用户组管理

### 创建用户组

```bash
ipa group-add devteam --desc="Development Team"
```

### 添加成员到组

```bash
ipa group-add-member devteam --users=zhangsan,lisi
```

### 查看组成员

```bash
ipa group-show devteam
```

***

## 主机管理

### 查看已注册主机

```bash
ipa host-find
```

### 手动添加主机记录

```bash
ipa host-add client01.example.local --ip-address=192.168.1.20
```

### 删除主机

```bash
ipa host-del client01.example.local
```

***

## 策略管理（HBAC）

HBAC（Host-Based Access Control）控制哪些用户可以登录哪些主机。

### 查看默认策略

```bash
ipa hbacrule-find
```

默认存在 `allow_all` 规则允许所有用户登录所有主机，生产环境建议禁用：

```bash
ipa hbacrule-disable allow_all
```

### 创建精细访问策略

```bash
# 创建规则：只允许 devteam 组登录 client01
ipa hbacrule-add dev-access --desc="Dev team can login to client01"
ipa hbacrule-add-user dev-access --groups=devteam
ipa hbacrule-add-host dev-access --hosts=client01.example.local
ipa hbacrule-add-service dev-access --hbacsvcs=sshd
```

### 测试 HBAC 规则

```bash
ipa hbactest --user=zhangsan --host=client01.example.local --service=sshd
```

***

## Sudo 规则管理

### 创建 Sudo 规则

```bash
# 允许 devteam 组在 client01 上以 root 运行所有命令
ipa sudorule-add dev-sudo
ipa sudorule-add-user dev-sudo --groups=devteam
ipa sudorule-add-host dev-sudo --hosts=client01.example.local
ipa sudorule-mod dev-sudo --cmdcat=all --runasusercat=all
```

***

## DNS 管理

FreeIPA 内置 DNS 可统一管理内部解析。

### 添加 A 记录

```bash
ipa dnsrecord-add example.local webserver --a-rec=192.168.1.50
```

### 添加 CNAME 记录

```bash
ipa dnsrecord-add example.local www --cname-rec=webserver.example.local.
```

### 查看 DNS Zone

```bash
ipa dnszone-find
ipa dnsrecord-find example.local
```

***

## 证书管理

FreeIPA 内置 Dogtag CA，可签发内部证书。

### 申请证书

```bash
ipa cert-request --principal=host/client01.example.local client01.csr
```

### 查看证书列表

```bash
ipa cert-find
```

### 吊销证书

```bash
ipa cert-revoke <serial-number> --revocation-reason=0
```

***

## SSH 公钥管理（sshpubkey）

FreeIPA 支持在用户账号上集中存储 SSH 公钥，客户端通过 SSSD 自动下发，实现免密码 SSH 登录的统一管控。

### 工作原理

1. 管理员将用户的 SSH 公钥上传至 FreeIPA LDAP。
2. 客户端 SSSD 配置 `ssh_provider = ipa`，登录时向 IPA 查询公钥。
3. SSH 服务通过 `AuthorizedKeysCommand` 调用 `sss_ssh_authorizedkeys` 获取公钥，无需维护 `~/.ssh/authorized_keys`。

### 服务端：为用户添加 SSH 公钥

```bash
# 上传本地公钥文件
ipa user-mod zhangsan --sshpubkey="$(cat ~/.ssh/id_rsa.pub)"

# 上传多个公钥（重复 --sshpubkey 参数）
ipa user-mod zhangsan \
  --sshpubkey="$(cat ~/.ssh/id_ed25519.pub)" \
  --sshpubkey="$(cat ~/.ssh/id_rsa.pub)"
```

### 查看用户公钥

```bash
ipa user-show zhangsan --all | grep SSH
```

### 删除用户所有公钥

```bash
ipa user-mod zhangsan --sshpubkey=
```

### 客户端：配置 SSSD 与 SSH

客户端加域后，编辑 `/etc/sssd/sssd.conf`，确保域段包含：

```ini
[domain/example.local]
id_provider = ipa
auth_provider = ipa
ssh_provider = ipa
```

编辑 `/etc/ssh/sshd_config`，添加或确认以下两行：

```text
AuthorizedKeysCommand /usr/bin/sss_ssh_authorizedkeys
AuthorizedKeysCommandUser nobody
```

重启相关服务使配置生效：

```bash
sudo systemctl restart sssd
sudo systemctl restart sshd
```

### 验证公钥下发

```bash
# 在客户端查询用户的公钥（应返回 IPA 中存储的内容）
sss_ssh_authorizedkeys zhangsan
```

### 主机公钥管理

FreeIPA 同样可以管理主机的 SSH 公钥，用于 known_hosts 集中维护：

```bash
# 查看主机公钥（客户端注册时自动上传）
ipa host-show client01.example.local --all | grep SSH

# 手动为主机添加公钥
ipa host-mod client01.example.local \
  --sshpubkey="$(ssh-keyscan -t ed25519 client01.example.local 2>/dev/null | awk '{print $2, $3}')"
```

客户端可通过 `sss_ssh_knownhostsproxy` 自动获取受信主机公钥，避免首次连接时的人工确认：

```bash
# 在 ~/.ssh/config 或全局 /etc/ssh/ssh_config 中添加
ProxyCommand /usr/bin/sss_ssh_knownhostsproxy -p %p %h
```

***

## 备份与恢复

### 完整备份

```bash
sudo ipa-backup
# 备份文件存放在 /var/lib/ipa/backup/
```

### 仅备份数据（不含配置）

```bash
sudo ipa-backup --data
```

### 恢复

```bash
sudo ipa-restore /var/lib/ipa/backup/<backup-dir>
```

***

## 常见问题

**Q：安装时报 "hostname does not match"？**\
\
检查 `/etc/hosts` 中主机名与 IP 的对应关系，确保 `hostname -f` 输出完整 FQDN。

**Q：kinit admin 报 "Cannot contact any KDC"？**\
\
确认防火墙已放行 88/tcp 和 88/udp（Kerberos），以及 FreeIPA 服务已启动：

```bash
sudo ipactl status
sudo ipactl start
```

**Q：Web UI 访问报证书错误？**\
\
将 `/etc/ipa/ca.crt` 导入浏览器信任列表，或在客户端运行：

```bash
sudo cp /etc/ipa/ca.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust
```

**Q：客户端加域后 SSH 无法用域账号登录？**\
\
检查 HBAC 规则是否允许该用户，并确认 `sssd` 服务正常运行：

```bash
sudo systemctl status sssd
sudo sssctl user-checks zhangsan
```

**Q：如何升级 FreeIPA？**\
\
直接通过 `dnf` 升级后运行 `ipa-server-upgrade`：

```bash
sudo dnf upgrade -y ipa-server
sudo ipa-server-upgrade
```

***

## 参考资料

- [FreeIPA 官方文档](https://www.freeipa.org/page/Documentation)
- [Red Hat Identity Management 指南](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/installing_identity_management/index)
- [Rocky Linux 官网](https://rockylinux.org/)
