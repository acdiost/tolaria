---
type: Note
status: Active
category: "[[hpc-infrastructure]]"
related_to:
  - "[[ldap-389ds-single-master-configuration]]"
  - "[[hpc-cluster-operations-index]]"
tags:
  - LDAP
  - SSSD
  - Authentication
  - Tutorial
---

# 从零配置 LDAP 客户端、SSSD 与 Linux 登录

本文把 control 和 compute01 配置为 LDAP 客户端。完成后，getent、id、PAM、SSH 和 Slurm 都能使用同一套中央身份。

~~~text
Linux 应用 / SSH / Slurm
          │
          ▼
     NSS + PAM
          │
          ▼
        SSSD
          │  LDAPS 636
          ▼
ldap.acdiost.internal
~~~

SSSD 使用只读服务账号搜索用户和组；用户登录时再用用户自己的密码认证。客户端不保存 Directory Manager 密码。

## 1. 前提条件

LDAP 服务端应已具备：

~~~text
Host       ldap.acdiost.internal
Base DN    dc=acdiost,dc=internal
People     ou=People,dc=acdiost,dc=internal
Groups     ou=Groups,dc=acdiost,dc=internal
LDAPS      636/TCP
CA         acdiost-ca.crt
~~~

只读搜索账号：

~~~text
cn=sssd-bind,ou=Services,dc=acdiost,dc=internal
~~~

客户端必须能解析 LDAP 域名、连接 636，并拥有 CA 证书和服务账号密码。不要先写 SSSD 配置再排查网络；按下面顺序分层验证。

## 2. 安装客户端组件

在 control 和 compute01 都执行：

~~~bash
dnf install openldap-clients sssd sssd-ldap sssd-tools authselect oddjob oddjob-mkhomedir
~~~

组件作用：

| 软件包 | 作用 |
| --- | --- |
| openldap-clients | 提供 LDAP 协议测试命令。 |
| sssd / sssd-ldap | 从 LDAP 获取身份并完成认证。 |
| sssd-tools | 提供 sssctl 和缓存诊断工具。 |
| authselect | 正确配置系统的 NSS/PAM 栈。 |
| oddjob-mkhomedir | 用户首次登录时创建家目录。 |

检查：

~~~bash
rpm -q openldap-clients sssd sssd-ldap oddjob-mkhomedir
~~~

## 3. 名称解析与端口

有 DNS 时：

~~~bash
getent ahostsv4 ldap.acdiost.internal
~~~

没有 DNS 时，在 /etc/hosts 加入：

~~~text
192.0.2.10 control ldap.acdiost.internal
~~~

验证端口：

~~~bash
timeout 3 bash -c '</dev/tcp/ldap.acdiost.internal/636' && echo reachable
~~~

或：

~~~bash
nc -vz ldap.acdiost.internal 636
~~~

如果这里失败，先处理 DNS、路由、防火墙或安全组。SSSD 参数无法修复网络不通。

## 4. 安装 CA 证书

在 LDAP 服务端导出：

~~~bash
dsctl acdiost tls export-cert Self-Signed-CA --output-file /root/acdiost-ca.crt
~~~

将 CA 文件安全复制到每台客户端，然后执行：

~~~bash
install -o root -g root -m 0644 acdiost-ca.crt /etc/openldap/certs/acdiost-ca.crt

restorecon -v /etc/openldap/certs/acdiost-ca.crt
~~~

检查证书：

~~~bash
openssl x509 -in /etc/openldap/certs/acdiost-ca.crt -noout -subject -issuer -dates
~~~

验证服务器：

~~~bash
openssl s_client -brief -connect ldap.acdiost.internal:636 -servername ldap.acdiost.internal -CAfile /etc/openldap/certs/acdiost-ca.crt </dev/null
~~~

必须使用 ldap.acdiost.internal 作为连接名。用 192.0.2.10 连接会绕过 DNS，但通常造成证书 SAN 不匹配。

## 5. 配置 OpenLDAP 命令行默认值

编辑 /etc/openldap/ldap.conf：

~~~ini
URI ldaps://ldap.acdiost.internal:636
BASE dc=acdiost,dc=internal
TLS_CACERT /etc/openldap/certs/acdiost-ca.crt
TLS_REQCERT demand
~~~

参数作用：

- URI：LDAP 客户端默认连接地址。
- BASE：未显式指定 -b 时的默认搜索根。
- TLS_CACERT：信任的 CA。
- TLS_REQCERT demand：证书链或主机名不正确时拒绝连接。

不要为了绕过证书错误改成 allow 或 never。应修复 CA、域名或服务器证书。

## 6. 在配置 SSSD 前测试 LDAP

使用只读服务账号：

~~~bash
ldapwhoami -x -H ldaps://ldap.acdiost.internal:636 -D 'cn=sssd-bind,ou=Services,dc=acdiost,dc=internal' -W
~~~

搜索用户：

~~~bash
ldapsearch -LLL -x -H ldaps://ldap.acdiost.internal:636 -D 'cn=sssd-bind,ou=Services,dc=acdiost,dc=internal' -W -b 'ou=People,dc=acdiost,dc=internal' '(uid=alice)' dn uid uidNumber gidNumber homeDirectory loginShell
~~~

如果 ldapwhoami 失败，不要继续配置 SSSD。先解决 DN、密码、TLS 或 ACI。

## 7. 配置 SSSD

创建 /etc/sssd/sssd.conf：

~~~ini
[sssd]
config_file_version = 2
services = nss, pam
domains = acdiost.internal

[domain/acdiost.internal]
id_provider = ldap
auth_provider = ldap
chpass_provider = ldap
access_provider = permit

ldap_uri = ldaps://ldap.acdiost.internal:636
ldap_search_base = dc=acdiost,dc=internal
ldap_user_search_base = ou=People,dc=acdiost,dc=internal
ldap_group_search_base = ou=Groups,dc=acdiost,dc=internal
ldap_schema = rfc2307

ldap_tls_reqcert = demand
ldap_tls_cacert = /etc/openldap/certs/acdiost-ca.crt

ldap_default_bind_dn = cn=sssd-bind,ou=Services,dc=acdiost,dc=internal
ldap_default_authtok_type = password
ldap_default_authtok = <只读服务账号密码>

cache_credentials = true
enumerate = false
use_fully_qualified_names = false
fallback_homedir = /home/%u
default_shell = /bin/bash
min_id = 10000
max_id = 59999
ldap_network_timeout = 3
~~~

配置说明：

| 参数 | 作用 |
| --- | --- |
| id_provider / auth_provider | 身份查询和密码认证都使用 LDAP。 |
| chpass_provider | 允许通过 LDAP 修改密码；不需要时可以移除。 |
| access_provider=permit | 允许所有可解析 LDAP 用户通过 SSSD 访问。生产环境可改为基于组或 LDAP filter 的策略。 |
| ldap_schema=rfc2307 | 使用 posixAccount、posixGroup 和 memberUid。 |
| ldap_default_bind_dn | 匿名访问关闭后，用专用账号执行搜索。 |
| cache_credentials=true | LDAP 短暂离线时允许已登录过的用户使用缓存认证。 |
| enumerate=false | 不在启动时遍历全目录，减少负载。 |
| min_id / max_id | 只接受规划范围内的 UID/GID。 |

本配置使用 ldaps://，因此不要再设置 ldap_id_use_start_tls=true。LDAPS 和 StartTLS 是两种不同连接方式。

保护配置：

~~~bash
chown root:root /etc/sssd/sssd.conf
chmod 600 /etc/sssd/sssd.conf
restorecon -v /etc/sssd/sssd.conf
~~~

SSSD 配置含服务账号密码。不能设为 644，也不能上传到 Git 或工单附件。

检查语法：

~~~bash
sssctl config-check
~~~

## 8. 接入 NSS 和 PAM

~~~bash
authselect select sssd with-mkhomedir --force
authselect current
systemctl enable --now oddjobd
systemctl enable --now sssd
~~~

作用：

- authselect select sssd：让 getent、id 和 PAM 使用 SSSD。
- with-mkhomedir：首次成功登录后创建 /home/用户名。
- oddjobd：执行受控的家目录创建任务。

不要手工拼接 /etc/pam.d 和 /etc/nsswitch.conf；authselect 会维护一套一致配置。

## 9. 分层验收

### 9.1 SSSD 在线状态

~~~bash
sssctl domain-status acdiost.internal
~~~

预期：

~~~text
Online status: Online
Active servers:
LDAP: ldap.acdiost.internal
~~~

### 9.2 NSS 身份解析

~~~bash
getent passwd alice
id alice
getent group hpc
~~~

成功结果应包含 LDAP 中配置的 UID、GID、家目录和 shell。

### 9.3 PAM 检查

~~~bash
sssctl user-checks alice
~~~

根据环境测试控制台或 SSH 登录：

~~~bash
ssh alice@compute01
~~~

首次登录后检查：

~~~bash
id
pwd
stat -c '%U:%G %a %n' "$HOME"
~~~

不要直接在生产账号上反复试错。先创建临时测试用户，完成后删除 LDAP 条目、Slurm association 和家目录。

## 10. 创建家目录和文件系统注意事项

with-mkhomedir 只在当前节点创建本地目录。两节点 HPC 集群常见选择：

1. 使用 NFS、Lustre 或其他共享 /home。
2. 每节点本地家目录，但用户文件不自动同步。
3. 登录节点创建家目录，计算节点通过共享存储挂载。

如果 control 和 compute01 各自创建本地 /home/alice，目录内容不是同一份。投入生产前必须明确存储策略。

## 11. 与 Slurm 记账联动

先确认两节点身份一致：

~~~bash
id alice
ssh root@compute01 id alice
~~~

两个结果的 UID/GID 必须相同。然后在 SlurmDBD 建立 association：

~~~bash
sacctmgr add account hpc Description='HPC users' Organization='acdiost'

sacctmgr add user alice Account=hpc Cluster=cluster

sacctmgr show user name=alice withassoc
~~~

提交测试：

~~~bash
sudo -iu alice srun --job-name=ldap-slurm-test -N1 -w compute01 /usr/bin/id
~~~

查询：

~~~bash
sacct -S today -X -o JobID,JobName,User,Account,State,NodeList
~~~

Linux 能解析用户不代表 Slurm 已授权。出现 Invalid account 时，应检查 association，而不是修改 LDAP 密码。

## 12. 图形 LDAP 客户端

~~~text
Host            ldap.acdiost.internal
Port            636
Version         3
Base            dc=acdiost,dc=internal
Authentication  Simple
SSL             开启
TLS/StartTLS    关闭
Anonymous       关闭
Username        cn=Directory Manager
Password        交互输入或从密码管理器读取
~~~

常见错误：

- Username 写成 cn=Directory Manager,dc=acdiost,dc=internal。
- Host 填 IP，导致证书名不匹配。
- 在 636 上同时启用 SSL 和 StartTLS。
- 没有把 CA 导入客户端信任库。
- 匿名访问已关闭，但客户端仍尝试匿名搜索。

## 13. 缓存和变更生效

修改 LDAP 用户或 SSSD 配置后：

~~~bash
sss_cache -E
systemctl restart sssd
~~~

作用：

- sss_cache -E：使所有缓存条目过期，下次查询重新读取 LDAP。
- restart sssd：重新加载配置并重新建立连接。

不要在 LDAP 离线时随意删除 SSSD 数据库目录。cache_credentials=true 保存的离线认证记录可能是维护窗口期间唯一可用的登录方式。

## 14. 轮换 SSSD 服务账号密码

轮换必须按顺序完成：

1. 备份两台客户端的 sssd.conf。
2. 在 LDAP 更新 sssd-bind 密码。
3. 立即同步到 control 和 compute01 的 root-only 配置。
4. 重启两台 SSSD。
5. 分别执行真实 LDAPS bind 和 sssctl domain-status。
6. 确认 Online 后删除临时密码文件。

验证时使用 -W 或受限临时文件，不要使用 -w 明文密码参数，因为密码可能进入进程列表和 shell 历史。

## 15. 故障排查路径

### 15.1 域状态 Offline

~~~bash
sssctl config-check
sssctl domain-status acdiost.internal
journalctl -u sssd -b --no-pager | tail -n 200
~~~

按顺序检查 DNS、636 端口、CA、Bind DN、密码和 ACI。

### 15.2 LDAP 成功，getent 查不到用户

检查用户是否包含：

~~~text
objectClass: posixAccount
uid
uidNumber
gidNumber
homeDirectory
loginShell
~~~

再检查搜索基准、UID 范围和缓存：

~~~bash
ldapsearch -LLL -x -D '<只读 Bind DN>' -W -b 'ou=People,dc=acdiost,dc=internal' '(uid=alice)'

sss_cache -E
getent passwd alice
~~~

### 15.3 getent 成功，密码登录失败

~~~bash
sssctl user-checks alice
journalctl -u sssd -u sshd -b --no-pager | tail -n 200
authselect current
~~~

检查 auth_provider、用户 DN、密码策略、账号状态和 PAM 栈。

### 15.4 LDAP 错误码

| 代码 | 含义 | 常见原因 |
| --- | --- | --- |
| 32 | No such object | Base DN 或用户 DN 写错。 |
| 48 | Inappropriate authentication | 匿名访问被禁止，或没有提供可用认证。 |
| 49 | Invalid credentials | DN、密码或账号状态错误。 |
| 50 | Insufficient access | ACI 没有授予所需搜索或读取权限。 |
| 81 | Cannot contact LDAP server | DNS、端口、TLS 或服务状态问题。 |

## 16. 安全检查表

- SSSD 不保存 Directory Manager 密码。
- /etc/sssd/sssd.conf 为 root:root 600。
- CA 校验使用 demand，不关闭证书验证。
- 服务账号只有 read、search、compare 权限。
- LDAP 用户 UID/GID 在所有节点一致且不与本地账号冲突。
- 生产环境根据组或访问过滤器替代 access_provider=permit。
- 修改前保留一条 root 本地登录通道，防止错误配置把管理员锁在系统外。

## 参考资料

- [RHEL 9：使用 SSSD 和 authselect 配置 LDAP 身份认证](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/pdf/configuring_authentication_and_authorization_in_rhel/configuring-user-authentication-using-authselect_configuring-authentication-and-authorization-in-rhel)
- [Red Hat Directory Server：安全与访问控制](https://docs.redhat.com/en/documentation/red_hat_directory_server/12/html-single/securing_red_hat_directory_server/index)
