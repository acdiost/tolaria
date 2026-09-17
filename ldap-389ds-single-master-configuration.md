---
type: Note
status: Active
category: "[[hpc-infrastructure]]"
related_to:
  - "[[hpc-cluster-operations-index]]"
tags:
  - LDAP
  - 389DS
  - Security
  - Tutorial
---

# 从零部署 389 Directory Server 单主 LDAP

完成本文后，集群会得到一个使用 LDAPS 的中央目录：

~~~text
LDAP 主机名     ldap.acdiost.internal
LDAP 地址       192.0.2.10
实例名          acdiost
Base DN         dc=acdiost,dc=internal
管理 DN         cn=Directory Manager
LDAP            389/TCP
LDAPS           636/TCP
拓扑            单主、无复制
匿名访问        禁止
简单绑定        必须经过 TLS
~~~

目录保存用户、组和服务账号。Linux 主机通过 SSSD 使用这些身份，Slurm 依赖所有节点看到一致的用户名、UID 和 GID。

所有命令默认由 root 执行。尖括号包围的内容是需要替换的变量。

## 1. LDAP 中几个容易混淆的概念

| 名称 | 本环境示例 | 作用 |
| --- | --- | --- |
| DNS 名称 | ldap.acdiost.internal | 客户端连接地址，也必须出现在 TLS 证书 SAN 中。 |
| Base DN / suffix | dc=acdiost,dc=internal | 业务目录树根节点。 |
| 实例名 | acdiost | systemd、dsctl 和本地 socket 使用的名称。 |
| Directory Manager DN | cn=Directory Manager | 目录超级管理员，不位于业务 Base DN 下。 |
| 用户 DN | uid=alice,ou=People,dc=acdiost,dc=internal | 某个用户条目的完整名称。 |
| Bind | 客户端提交 DN 和凭据 | LDAP 的认证动作。 |

最重要的区别：管理 DN 是：

~~~text
cn=Directory Manager
~~~

不是：

~~~text
cn=Directory Manager,dc=acdiost,dc=internal
~~~

## 2. 环境和名称规划

### 2.1 检查系统

~~~bash
cat /etc/os-release
hostnamectl --static
timedatectl
free -h
df -hT /
~~~

当前部署是 Rocky Linux 9.4、389 DS 2.8.0。RHEL 9 的 Red Hat Directory Server 文档可作为兼容参考。

### 2.2 先保证名称解析

在 DNS 中建立：

~~~text
ldap.acdiost.internal.  A  192.0.2.10
~~~

没有内部 DNS 时，LDAP 主机和两台客户端的 /etc/hosts 至少加入：

~~~text
192.0.2.10 control ldap.acdiost.internal
~~~

验证：

~~~bash
getent ahostsv4 ldap.acdiost.internal
~~~

证书应签发给 ldap.acdiost.internal。客户端使用 IP 地址连接时，即使密码正确，也会因为证书名称不匹配而失败。

### 2.3 时间同步

~~~bash
systemctl enable --now chronyd
chronyc tracking
~~~

证书有效期、日志时间和多节点认证排障都依赖正确时间。

## 3. 安装软件

Rocky Linux 9：

~~~bash
dnf install 389-ds-base openldap-clients
~~~

如果需要 Cockpit 管理界面：

~~~bash
dnf install cockpit cockpit-389-ds
systemctl enable --now cockpit.socket
~~~

在有 Red Hat Directory Server 订阅的 RHEL 9 上，官方流程是：

~~~bash
dnf module enable redhat-ds:12
dnf install 389-ds-base cockpit-389-ds
~~~

命令作用：

- 389-ds-base：目录服务、实例工具、数据库和插件。
- openldap-clients：提供 ldapsearch、ldapadd、ldapmodify、ldapwhoami 和 ldappasswd。
- cockpit-389-ds：可选 Web 管理界面，不是服务运行的必要条件。

检查：

~~~bash
rpm -q 389-ds-base openldap-clients
dscreate --help
~~~

## 4. 创建 acdiost 实例

### 4.1 推荐：交互式安装

~~~bash
dscreate interactive
~~~

按以下值回答：

| 提示 | 输入 |
| --- | --- |
| System hostname | ldap.acdiost.internal |
| Instance name | acdiost |
| LDAP port | 389 |
| Create self-signed certificate | 实验环境选 yes |
| LDAPS port | 636 |
| Directory Manager DN | cn=Directory Manager |
| Directory Manager password | 从密码管理器生成并交互输入 |
| Database library | 使用发行版支持的默认值 |
| Database suffix | dc=acdiost,dc=internal |
| Create suffix entry | yes |

dscreate 会创建文件目录、证书数据库、后端和 systemd 实例，并自动启动服务。

### 4.2 可重复部署：INF 文件

先获取当前软件版本支持的模板：

~~~bash
dscreate create-template /root/acdiost.inf
chmod 600 /root/acdiost.inf
~~~

至少配置：

~~~ini
[general]
full_machine_name = ldap.acdiost.internal
start = True

[slapd]
instance_name = acdiost
port = 389
secure_port = 636
self_sign_cert = True
root_password = <从安全存储临时写入>

[backend-userroot]
suffix = dc=acdiost,dc=internal
create_suffix_entry = True
~~~

创建后安全删除答案文件：

~~~bash
dscreate from-file /root/acdiost.inf
shred -u /root/acdiost.inf
~~~

INF 中可能含明文管理密码，因此必须为 600。自动化环境应从 Secret Manager 临时注入，不要把 INF 提交到 Git。

## 5. 检查实例和监听

~~~bash
dsctl acdiost status
systemctl status dirsrv@acdiost --no-pager
systemctl enable dirsrv@acdiost
ss -lntp | grep -E ':(389|636)[[:space:]]'
~~~

作用：

- dsctl：按实例执行启停、备份、证书和数据库管理。
- dirsrv@acdiost：该实例的 systemd 服务。
- ss：确认实际监听地址和端口，而不是只看配置文件。

要允许所有可路由来源访问，监听应类似：

~~~text
*:389
*:636
~~~

还要确认 systemd 没有 IP 沙箱限制：

~~~bash
systemctl show dirsrv@acdiost -p IPAddressAllow -p IPAddressDeny
~~~

两项为空代表 systemd 不限制来源。

## 6. 防火墙和 SELinux

如果 firewalld 已启用：

~~~bash
firewall-cmd --permanent --add-port=389/tcp
firewall-cmd --permanent --add-port=636/tcp
firewall-cmd --reload
firewall-cmd --list-ports
~~~

这会允许所有进入当前 zone 的来源访问 389/636。上游路由、NAT、云安全组仍可能限制连接。

安全注意事项：

- 对外提供 LDAP 时优先使用 636/LDAPS。
- 389 只用于 StartTLS 或无凭据协议探测，不允许明文密码绑定。
- “允许所有地址连接”不等于“允许匿名读取”；这是两个独立设置。
- 生产环境建议保持 SELinux Enforcing。标准端口和路径由软件包自动标记。

检查：

~~~bash
getenforce
firewall-cmd --state
~~~

当前集群的 firewalld 和 SELinux 处于关闭状态；这是现状，不是从零部署的推荐基线。

## 7. 创建目录树

本地 root 可以通过 LDAPI 和 SASL/EXTERNAL 管理实例，不需要把 Directory Manager 密码放在命令行。

创建 /root/base-tree.ldif：

~~~ldif
dn: ou=People,dc=acdiost,dc=internal
objectClass: top
objectClass: organizationalUnit
ou: People

dn: ou=Groups,dc=acdiost,dc=internal
objectClass: top
objectClass: organizationalUnit
ou: Groups

dn: ou=Services,dc=acdiost,dc=internal
objectClass: top
objectClass: organizationalUnit
ou: Services
~~~

导入：

~~~bash
ldapadd -Y EXTERNAL -H 'ldapi://%2Frun%2Fslapd-acdiost.socket' -f /root/base-tree.ldif
rm -f /root/base-tree.ldif
~~~

检查：

~~~bash
ldapsearch -LLL -Y EXTERNAL -H 'ldapi://%2Frun%2Fslapd-acdiost.socket' -b 'dc=acdiost,dc=internal' -s one dn
~~~

## 8. TLS 和 CA

### 8.1 检查证书

~~~bash
dsctl acdiost tls show-server-cert
dsctl acdiost tls list-ca
dsconf acdiost security get
~~~

确认：

- Subject Alternative Name 包含 ldap.acdiost.internal。
- 证书未过期。
- nsslapd-security 为 on。
- TLS 最低版本至少为 1.2。

### 8.2 导出自签 CA

~~~bash
dsctl acdiost tls export-cert Self-Signed-CA --output-file /root/acdiost-ca.crt

openssl x509 -in /root/acdiost-ca.crt -noout -subject -issuer -dates
~~~

将 CA 安全复制到每台客户端：

~~~text
/etc/openldap/certs/acdiost-ca.crt
~~~

自签 CA 适合封闭实验环境。生产环境应使用组织 CA。大致流程是：

~~~bash
dsctl acdiost tls generate-server-cert-csr
dsctl acdiost tls import-ca
dsctl acdiost tls import-server-cert
~~~

具体参数以每个子命令的 --help 为准。

### 8.3 TLS 验收

~~~bash
openssl s_client -brief -connect ldap.acdiost.internal:636 -servername ldap.acdiost.internal -CAfile /root/acdiost-ca.crt </dev/null
~~~

成功标准：证书链验证通过，主机名匹配，协议为 TLS 1.2 或 TLS 1.3。

## 9. 禁止明文和匿名绑定

~~~bash
dsconf acdiost config replace nsslapd-require-secure-binds=on

dsconf acdiost config replace nsslapd-allow-anonymous-access=off

dsconf acdiost config replace nsslapd-allow-unauthenticated-binds=off

dsconf acdiost config replace passwordStorageScheme=PBKDF2-SHA512

systemctl restart dirsrv@acdiost
~~~

设置作用：

| 设置 | 作用 |
| --- | --- |
| nsslapd-require-secure-binds=on | 简单用户名密码绑定必须经过 LDAPS 或 StartTLS。 |
| nsslapd-allow-anonymous-access=off | 未提供 DN 和密码的客户端不能读取目录。 |
| nsslapd-allow-unauthenticated-binds=off | 提供 DN 但空密码时拒绝绑定，防止客户端误判为登录成功。 |
| PBKDF2-SHA512 | 新设置的用户密码按强哈希方案存储。 |

验证：

~~~bash
dsconf acdiost config get nsslapd-require-secure-binds nsslapd-allow-anonymous-access nsslapd-allow-unauthenticated-binds passwordStorageScheme
~~~

匿名负向测试：

~~~bash
LDAPTLS_CACERT=/root/acdiost-ca.crt ldapsearch -x -H ldaps://ldap.acdiost.internal:636 -b 'dc=acdiost,dc=internal' -s base dn
~~~

预期失败：

~~~text
ldap_bind: Inappropriate authentication (48)
additional info: Anonymous access is not allowed
~~~

关闭匿名绑定后，依赖匿名搜索“先查用户名、再绑定”的客户端会失败。它们必须配置只读搜索账号，或直接提交完整用户 DN。

## 10. 创建 SSSD 只读服务账号

不要让所有 Linux 主机保存 Directory Manager 密码。为 SSSD 创建最小权限账号。

安全读取密码并生成 root-only LDIF：

~~~bash
read -rsp 'SSSD bind password: ' SSSD_BIND_PASSWORD
echo
install -m 600 /dev/null /root/sssd-bind.ldif

{
  echo 'dn: cn=sssd-bind,ou=Services,dc=acdiost,dc=internal'
  echo 'objectClass: top'
  echo 'objectClass: organizationalRole'
  echo 'objectClass: simpleSecurityObject'
  echo 'cn: sssd-bind'
  printf 'userPassword: %s\n' "$SSSD_BIND_PASSWORD"
} > /root/sssd-bind.ldif
~~~

导入并清理：

~~~bash
ldapadd -Y EXTERNAL -H 'ldapi://%2Frun%2Fslapd-acdiost.socket' -f /root/sssd-bind.ldif

shred -u /root/sssd-bind.ldif
unset SSSD_BIND_PASSWORD
~~~

创建 /root/sssd-aci.ldif，为该 DN 授予只读、搜索和比较权限，但排除密码与 ACI：

~~~ldif
dn: dc=acdiost,dc=internal
changetype: modify
add: aci
aci: (targetattr != "userPassword || aci")(version 3.0; acl "SSSD read-only directory access"; allow (read,search,compare) userdn = "ldap:///cn=sssd-bind,ou=Services,dc=acdiost,dc=internal";)
~~~

应用：

~~~bash
ldapmodify -Y EXTERNAL -H 'ldapi://%2Frun%2Fslapd-acdiost.socket' -f /root/sssd-aci.ldif
rm -f /root/sssd-aci.ldif
~~~

重复执行 add: aci 可能返回属性已存在，自动化脚本应先查询后修改。

## 11. 创建 POSIX 用户和组

### 11.1 组条目

~~~ldif
dn: cn=hpc,ou=Groups,dc=acdiost,dc=internal
objectClass: top
objectClass: posixGroup
cn: hpc
gidNumber: 20000
memberUid: alice
~~~

### 11.2 用户条目

~~~ldif
dn: uid=alice,ou=People,dc=acdiost,dc=internal
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: posixAccount
uid: alice
cn: Alice
sn: User
uidNumber: 20000
gidNumber: 20000
homeDirectory: /home/alice
loginShell: /bin/bash
~~~

先通过 LDAPI 导入组和用户，用户 LDIF 不写密码。然后交互设置密码：

~~~bash
LDAPTLS_CACERT=/root/acdiost-ca.crt ldappasswd -x -H ldaps://ldap.acdiost.internal:636 -D 'cn=Directory Manager' -W -S 'uid=alice,ou=People,dc=acdiost,dc=internal'
~~~

- -W 询问管理员密码。
- -S 两次询问新用户密码。
- 两个密码都不会出现在 shell 历史中。

UID/GID 规划：

- 所有节点必须看到同一个用户名对应同一个 UID/GID。
- 避开系统账号范围；当前 SSSD 只接受 10000–59999。
- UID/GID 不得与本地账号冲突。
- Slurm accounting 按用户名记录，MUNGE 认证又依赖 UID，一致性非常重要。

## 12. 管理员与目录验证

管理员绑定：

~~~bash
LDAPTLS_CACERT=/root/acdiost-ca.crt ldapwhoami -x -H ldaps://ldap.acdiost.internal:636 -D 'cn=Directory Manager' -W
~~~

查找用户：

~~~bash
LDAPTLS_CACERT=/root/acdiost-ca.crt ldapsearch -LLL -x -H ldaps://ldap.acdiost.internal:636 -D 'cn=Directory Manager' -W -b 'ou=People,dc=acdiost,dc=internal' '(uid=alice)' dn uid uidNumber gidNumber homeDirectory loginShell
~~~

不要在普通工作站长期使用 Directory Manager。日常应用应使用最小权限服务账号。

## 13. 图形客户端填写方式

~~~text
Host            ldap.acdiost.internal
Port            636
LDAP Version    3
Base            dc=acdiost,dc=internal
Authentication  Simple
SSL / LDAPS     开启
StartTLS        关闭
Anonymous       关闭
Username        cn=Directory Manager
Password        从密码管理器读取
~~~

636 使用 SSL/LDAPS；389 才使用 StartTLS。不要同时勾选两种模式。客户端还必须导入 acdiost-ca.crt。

## 14. 备份和恢复准备

在线备份：

~~~bash
dsctl acdiost db2bak
dsctl acdiost backups
~~~

同时保护：

~~~text
/etc/dirsrv/slapd-acdiost/
/etc/dirsrv/ssca/
/var/lib/dirsrv/slapd-acdiost/bak/
~~~

注意：

- 单主拓扑没有实时副本，主机故障会导致新认证和目录查询不可用。
- 备份应复制到另一台主机，并定期演练恢复。
- bak2db 会替换数据库内容，属于破坏性恢复操作，必须在维护窗口确认备份名后执行。
- 更换 CA 或服务器证书时，先分发新 CA，再切换服务器证书。

## 15. 常见故障

### 密码正确但代码 49

~~~text
ldap_bind: Invalid credentials (49)
~~~

依次检查：

1. Bind DN 是否完整。
2. 管理员是否错误地写成位于 Base DN 下。
3. 密码是否包含全角字符或尾部空格。
4. 账号是否被禁用或密码策略锁定。

### 证书验证失败

~~~bash
openssl s_client -connect ldap.acdiost.internal:636 -servername ldap.acdiost.internal -CAfile /root/acdiost-ca.crt </dev/null
~~~

检查 CA、有效期、SAN 和系统时间。不要关闭证书验证作为长期修复。

### 端口可达但无法绑定

~~~bash
dsctl acdiost status
dsconf acdiost config get nsslapd-require-secure-binds
journalctl -u dirsrv@acdiost -b --no-pager | tail -n 150
~~~

如果通过 389 做简单绑定而没有 StartTLS，require-secure-binds=on 会按设计拒绝。

## 参考资料

- [Red Hat：安装 Directory Server](https://docs.redhat.com/en/documentation/red_hat_directory_server/12/html-single/installing_red_hat_directory_server/installing_red_hat_directory_server)
- [Red Hat：禁止匿名绑定](https://docs.redhat.com/en/documentation/red_hat_directory_server/12/html/securing_red_hat_directory_server/assembly_disabling-anonymous-binds_securing-rhds)
- [Red Hat：要求 LDAPS 或 StartTLS](https://docs.redhat.com/en/documentation/red_hat_directory_server/12/html/securing_red_hat_directory_server/requiring-ldaps-or-starttls-for-encrypted-connections_securing-rhds)
