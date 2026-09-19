---
type: Note
status: Active
tags:
  - StackStorm
  - Ansible
  - Automation
---

# StackStorm 自定义 Action 配置指南

本文记录如何在 StackStorm 3.9 中创建、注册、执行和维护自定义 Action。示例已在 `stackstorm` 服务器验证，使用 `/etc/ansible/hosts` 中的主机执行一个无副作用的 Hello World playbook。

## 1. 核心概念

StackStorm 的资源以 **Pack** 为组织单位。一个可执行的 Action 通常至少包含：

1. `pack.yaml`：Pack 的元数据。
2. `actions/<action>.yaml`：Action 的元数据、runner 类型和参数定义。
3. `actions/<entry_point>`：真正执行任务的脚本或程序。

如果 Action 调用 Ansible playbook，还可以增加 `playbooks/` 目录。

Action 的完整引用格式为：

```text
<pack_ref>.<action_name>
```

本例的完整引用是 `hello_world.hello`。

## 2. 示例目录结构

StackStorm 默认从 `/opt/stackstorm/packs` 加载 Pack：

```text
/opt/stackstorm/packs/hello_world/
├── pack.yaml
├── actions/
│   ├── hello.yaml
│   └── hello.sh
└── playbooks/
    └── hello.yaml
```

建议为自定义功能创建独立 Pack，不要直接修改 `core`、`ansible` 等系统 Pack，以免升级时被覆盖。

创建目录：

```bash
install -d -m 0775 -o root -g st2packs \
  /opt/stackstorm/packs/hello_world/actions \
  /opt/stackstorm/packs/hello_world/playbooks
```

## 3. 创建 Pack 元数据

文件：`/opt/stackstorm/packs/hello_world/pack.yaml`

```yaml
---
ref: hello_world
name: Hello World
description: Minimal StackStorm actions for validating the Ansible integration.
keywords:
  - example
  - ansible
version: 1.0.0
author: local
email: root@example.com
```

重要字段：

| 字段 | 作用 |
| --- | --- |
| `ref` | Pack 的机器可读标识，也是 Action 引用的前半部分 |
| `name` | Pack 的显示名称 |
| `description` | 功能说明 |
| `version` | Pack 版本 |
| `author` / `email` | 作者信息 |

注意：StackStorm 3.9 会严格校验邮箱域名，`root@localhost` 会注册失败，可使用 `root@example.com` 或真实有效邮箱。

## 4. 定义 Action

文件：`/opt/stackstorm/packs/hello_world/actions/hello.yaml`

```yaml
---
name: hello
runner_type: local-shell-script
description: Run a Hello World Ansible playbook against the configured inventory.
enabled: true
entry_point: hello.sh
parameters:
  greeting:
    type: string
    description: Greeting printed once for each inventory host.
    default: Hello World
    position: 0
```

关键字段：

| 字段 | 作用 |
| --- | --- |
| `name` | Action 名称，最终组成 `hello_world.hello` |
| `runner_type` | 执行方式，本例使用本地 Shell 脚本 |
| `enabled` | 是否允许执行 |
| `entry_point` | 相对于 Pack `actions/` 目录的入口文件 |
| `parameters` | Action 对外暴露的参数定义 |
| `position` | 将参数作为第几个命令行位置参数传入脚本，从 0 开始 |

常见 runner 可用下面的命令查看：

```bash
st2 runner list
```

常见选择包括：

| Runner | 适用场景 |
| --- | --- |
| `local-shell-script` | 在 StackStorm 节点运行 Shell、Python 等脚本 |
| `local-shell-cmd` | 运行一条本地命令 |
| `remote-shell-script` | 通过 SSH 在远程节点运行脚本 |
| `python-script` | 使用 StackStorm Python runner 开发复杂逻辑 |
| `orquesta` | 编排多个 Action，包含分支、重试和并行步骤 |

## 5. 编写入口脚本

文件：`/opt/stackstorm/packs/hello_world/actions/hello.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

greeting=${*:-Hello World}
extra_vars=$(/opt/stackstorm/st2/bin/python -c \
  'import json, sys; print(json.dumps({"hello_world_greeting": sys.argv[1]}))' \
  "$greeting")

exec /opt/stackstorm/virtualenvs/ansible/bin/ansible-playbook \
  --inventory-file /etc/ansible/hosts \
  --extra-vars "$extra_vars" \
  /opt/stackstorm/packs/hello_world/playbooks/hello.yaml
```

实现要点：

- `set -euo pipefail` 让命令失败、未定义变量和管道失败正确反映为 Action 失败。
- 使用 StackStorm Ansible Pack 的虚拟环境。服务器的普通登录 Shell 中没有全局 `ansible-playbook` 命令。
- extra-vars 使用 JSON 传递。`key=value` 格式会错误拆分 `Hello StackStorm World` 这类含空格的值。
- `exec` 让 Ansible 的退出码直接成为 Action 的退出码。
- inventory 明确指定为 `/etc/ansible/hosts`。

如果参数可能包含密码或令牌，应在 Action 元数据中增加 `secret: true`，并避免把它写入 stdout、命令行日志或异常信息。

## 6. 编写 Ansible Playbook

文件：`/opt/stackstorm/packs/hello_world/playbooks/hello.yaml`

```yaml
---
- name: StackStorm Hello World
  hosts: all
  gather_facts: false
  tasks:
    - name: Print greeting
      ansible.builtin.debug:
        msg: "{{ hello_world_greeting }} from {{ inventory_hostname }}"
```

这个 playbook 仅运行 `debug`，不会修改远程主机。实际生产 Action 应尽量保证 Ansible 任务幂等，并用 `limit`、主机组或明确的 host pattern 控制执行范围，避免无意中对 inventory 中所有主机执行变更。

## 7. 设置权限

```bash
chown -R root:st2packs /opt/stackstorm/packs/hello_world
chmod 0775 /opt/stackstorm/packs/hello_world/actions/hello.sh
chmod 0664 \
  /opt/stackstorm/packs/hello_world/pack.yaml \
  /opt/stackstorm/packs/hello_world/actions/hello.yaml \
  /opt/stackstorm/packs/hello_world/playbooks/hello.yaml
```

入口脚本必须可执行。Pack 文件应确保 StackStorm 进程所属用户或 `st2packs` 组可以读取。

## 8. 注册和检查

注册所有 Pack 内容：

```bash
st2ctl reload --register-all
```

开发过程中，也可以只注册相关资源以减少影响和输出：

```bash
st2-register-content --register-pack hello_world
st2-register-content --register-actions --pack hello_world
```

检查 Pack 和 Action：

```bash
st2 pack get hello_world
st2 action get hello_world.hello
st2 action list --pack hello_world
```

修改 `pack.yaml`、Action YAML 或增加 Action 后需要重新注册。只修改入口脚本或 playbook 内容时通常不需要重新注册元数据，但重新注册一次便于确认整体状态。

## 9. 执行 Action

使用默认参数：

```bash
st2 run hello_world.hello
```

传入自定义参数：

```bash
st2 run hello_world.hello greeting="Hello StackStorm World"
```

本次实际验证结果：

```text
status: succeeded
return_code: 0
ok: [10.82.99.97] => {
    "msg": "Hello StackStorm World from 10.82.99.97"
}
```

异步执行并查询结果：

```bash
st2 run hello_world.hello --async
st2 execution get <execution-id>
st2 execution get <execution-id> --detail
```

也可以查看最近的执行：

```bash
st2 execution list --action hello_world.hello
```

## 10. 常见故障排查

### Action 不在列表中

```bash
st2 action list --pack hello_world
st2ctl reload --register-all
```

检查 YAML 语法、文件路径，以及 `pack.yaml` 中的 `ref` 是否与目录和调用名称一致。

### `ansible-playbook: command not found`

本服务器的 Ansible 位于 Pack 虚拟环境：

```text
/opt/stackstorm/virtualenvs/ansible/bin/ansible-playbook
```

入口脚本应使用绝对路径，不要依赖交互式 Shell 的 `PATH`。

### Action 成功但参数被截断

带空格的参数不要直接拼为：

```bash
--extra-vars "hello_world_greeting=$greeting"
```

Ansible 会再次解析该字符串。本例先用 Python `json.dumps` 生成 JSON，再把整个 JSON 作为一个 `--extra-vars` 参数传入。

### Action Runner 与手工 SSH 行为不同

Action 由 StackStorm runner 用户和服务环境执行，不继承登录用户的环境变量、SSH agent、当前目录或 Shell 配置。排查时重点检查：

```bash
st2 execution get <execution-id> --detail
sudo -u stanley ssh <target-host> true
ls -l /etc/ansible/hosts
```

实际用户名称取决于 StackStorm 配置，不应假设一定是 `root`。

### Action 一直运行或超时

为可能耗时的 Action 设置合理的 `timeout`，并确保调用的程序能够响应终止信号。Ansible 任务还应配置连接超时、重试策略和明确的主机范围。

## 11. 后续创建 Action 的检查清单

- 为业务功能建立独立 Pack，并使用稳定的 `ref`。
- 选择与任务匹配的 runner。
- 在 Action YAML 中声明参数类型、默认值、必填规则和是否为 secret。
- 入口脚本使用绝对路径、严格错误处理并返回真实退出码。
- 对外部输入正确引用和序列化，不直接拼接命令。
- Ansible playbook 保持幂等，并限制目标主机范围。
- 设置正确的 `root:st2packs` 所有权和入口文件执行权限。
- 注册后用 `st2 action get` 检查元数据。
- 分别验证默认参数、边界参数和失败路径。
- 使用 `st2 execution get --detail` 保存可追踪的执行证据。

## 12. 完整创建脚本

以下脚本可在 StackStorm 服务器上以具备 `/opt/stackstorm/packs` 写权限的用户执行：

```bash
#!/usr/bin/env bash
set -euo pipefail

pack_dir=/opt/stackstorm/packs/hello_world

install -d -m 0775 -o root -g st2packs "$pack_dir/actions" "$pack_dir/playbooks"

cat > "$pack_dir/pack.yaml" <<'EOF'
---
ref: hello_world
name: Hello World
description: Minimal StackStorm actions for validating the Ansible integration.
keywords: [example, ansible]
version: 1.0.0
author: local
email: root@example.com
EOF

cat > "$pack_dir/actions/hello.yaml" <<'EOF'
---
name: hello
runner_type: local-shell-script
description: Run a Hello World Ansible playbook against the configured inventory.
enabled: true
entry_point: hello.sh
parameters:
  greeting:
    type: string
    description: Greeting printed once for each inventory host.
    default: Hello World
    position: 0
EOF

cat > "$pack_dir/actions/hello.sh" <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
greeting=${*:-Hello World}
extra_vars=$(/opt/stackstorm/st2/bin/python -c \
  'import json, sys; print(json.dumps({"hello_world_greeting": sys.argv[1]}))' \
  "$greeting")
exec /opt/stackstorm/virtualenvs/ansible/bin/ansible-playbook \
  --inventory-file /etc/ansible/hosts \
  --extra-vars "$extra_vars" \
  /opt/stackstorm/packs/hello_world/playbooks/hello.yaml
EOF

cat > "$pack_dir/playbooks/hello.yaml" <<'EOF'
---
- name: StackStorm Hello World
  hosts: all
  gather_facts: false
  tasks:
    - name: Print greeting
      ansible.builtin.debug:
        msg: "{{ hello_world_greeting }} from {{ inventory_hostname }}"
EOF

chown -R root:st2packs "$pack_dir"
chmod 0775 "$pack_dir/actions/hello.sh"
chmod 0664 "$pack_dir/pack.yaml" "$pack_dir/actions/hello.yaml" \
  "$pack_dir/playbooks/hello.yaml"

st2ctl reload --register-all
st2 action get hello_world.hello
st2 run hello_world.hello greeting="Hello StackStorm World"
```
