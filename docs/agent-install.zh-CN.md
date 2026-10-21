# Agent 安装与 SSH 验证流程

[中文首页](../README.zh-CN.md) · [参数与维护参考](usage.zh-CN.md) · [沙箱环境说明](sandbox-environment.zh-CN.md)

当用户委托你在 Dots 沙箱中安装本项目、交由用户完成浏览器认证并建立 SSH 连接时，按本文执行。阅读文档或修改仓库本身不需要在沙箱部署；只有实际受托安装时才运行下面的命令。

本文使用 fork `windddkz/dots-tailscale` 的 `main`，当前合并了上游 v0.4.0。软件包版本为 0.4.0，安装器下载的 Tailscale 版本为 1.102.4。首次安装需要完整源码及其 `source-manifest.json`；部署时记录实际提交号。占位值必须替换为本次真实数据。

## 执行位置与完成条件

| 位置 | 执行内容 |
| --- | --- |
| 沙箱 | 使用平台原生桌面终端或该沙箱的命令工具，完成检查、安装、公钥配置和状态确认 |
| 客户端 | 用户实际连接的电脑，须已加入允许访问该节点的 tailnet；私钥保留在这里，SSH 配置和外部连接检查也在这里进行 |
| 浏览器 | 用户打开本次真实认证链接，核对账号、tailnet 和设备授权 |

先用 `id`、`pwd`、`uname` 确认执行上下文，本地维护仓库不等于目标沙箱。没有沙箱终端入口时，向用户索取可用入口；新机器尚未安装 SSH，不能把它已有 SSH 可用当作前提。

客户端公钥是必要输入。能操作客户端时读取准备使用的 `.pub` 文件；否则向用户索取单行 SSH 公钥，不索取私钥。不能操作客户端时，最后交付用户执行的连接命令，并明确外部验证待完成，不报告“SSH 已验证成功”。

按下面的顺序执行。最终须确认：安装文件齐备、后端 Running、客户端公钥已授权、服务器指纹经独立通道核对、无 PTY 命令成功、真实交互终端可用。浏览器认证由用户完成，等待时间不能代替用户操作或后端检查。

## 1. 检查沙箱及已有安装

在沙箱执行以下检查，不停止进程、不更改节点身份：

```bash
uname -srm
id
pwd
for tool in bash python3 timeout flock openssl ssh-keygen /usr/sbin/sshd \
  curl tar sha256sum git nohup; do
  command -v "$tool" || printf 'missing: %s\n' "$tool"
done
test -w /workspace && printf '/workspace is writable\n'
cat /proc/sys/net/ipv4/ip_unprivileged_port_start
```

继续条件为 Linux x86_64、依赖齐全、工作区可写，以及 `ip_unprivileged_port_start` 小于或等于 22。缺失依赖或端口条件不满足时，报告具体项；不能通过修改系统路由、组映射或全局权限掩盖差异。

检查 `/workspace/.dots-tailscale` 的实际文件，选择分支：

| 情况 | 后续动作 |
| --- | --- |
| 没有安装目录 | 进入第 2 节首次安装 |
| 已有 `installed` 标记及完整文件 | 跳过第 2 节 `install`；检查现有 `start.sh check`，核对第 3、4 节的修复和公钥，再进入第 5 节 |
| 目录存在但不完整 | 保留现有文件，依据日志诊断，确认身份文件状态后再决定恢复 |
| 已认证标记存在但 state 或主机密钥丢失 | 停止并报告身份材料缺失，等待用户决定恢复方式 |

已有安装使用 `bash /workspace/.dots-tailscale/start.sh check` 检查。该命令可能输出 `missing:` 后仍返回 `0`，不能只看退出码。旧安装可能带有手工修复，也可能没有 v0.4.0 的命令；首次安装流程不包含强制升级已有节点。需要升级时另按[维护参考](usage.zh-CN.md#update-management)核对。

## 2. 获取并安装完整源码

本节仅用于尚未安装的沙箱。源码目录与安装目录不同。若 `/workspace/dots-tailscale` 已存在，先检查远端、提交和本地修改，复用已核对的源码或另选空目录，不先删除再克隆。

在沙箱执行，确认克隆成功后再继续：

```bash
cd /workspace
HTTPS_PROXY=http://browser-proxy:8889 HTTP_PROXY=http://browser-proxy:8889 \
  git clone --branch main --depth 1 https://github.com/windddkz/dots-tailscale.git
```

```bash
(
  set -euo pipefail
  cd /workspace/dots-tailscale
  git rev-parse HEAD
  python3 scripts/source_manifest.py --check
  bash -n tailscale-setup.sh
  bash tailscale-setup.sh install
)
```

记录完整提交号。任一步失败则保留输出并处理该项，不继续宣告成功。安装器会校验源码清单和下载归档，把程序、配置及初始密钥放到 `/workspace/.dots-tailscale`；此步骤不启动服务。

保持源码检出中的四个源文件和 `source-manifest.json` 不变。下一节的 PTY 修复修改的是安装后的 `setup.sh`；提前修改源码会触发 v0.4.0 的初始完整性校验失败。

## 3. 检查并处理 PTY 兼容问题

在沙箱执行只读判断：

```bash
python3 - <<'PY'
import grp
import os
from pathlib import Path

try:
    tty_gid = grp.getgrnam('tty').gr_gid
except KeyError:
    print('PTY_FIX_REQUIRED=0: tty group is absent; OpenSSH has a fallback')
else:
    ranges = [tuple(map(int, line.split()))
              for line in Path('/proc/self/gid_map').read_text().splitlines()]
    mapped = any(start <= tty_gid < start + count for start, outer, count in ranges)
    needed = os.geteuid() != 0 and not mapped
    print(f'PTY_FIX_REQUIRED={int(needed)}: tty_gid={tty_gid}, mapped={mapped}')
PY
```

输出 `PTY_FIX_REQUIRED=1` 时，必须阅读并完成 [OpenSSH 交互终端修复](openssh-pty-fix.zh-CN.md)中的“编译兼容库”和“加载检查与启动脚本修改”，包括实际加载检查。这个链接是该分支的必做步骤：在已观察到的沙箱条件下，遗漏修复会导致交互 SSH 断开。缺少 GCC 时报告该依赖，不能声称交互终端已具备条件。

全新安装尚未启动 sshd 时，完成启动语句修改后回到本文第 4 节；此时没有监听 PID，不执行修复文档中针对已有进程的重启命令。已有安装若需要应用修复，则在独立平台终端按该文档受控重启。已有兼容库和加载语句时先核对内容，不重复覆盖。

输出为 `0` 时继续，仍须在第 8 节验证真实 PTY。其他错误按日志诊断，不自动归因于同一问题。

v0.4.0 中，修改安装后的 `setup.sh` 会使版本检查报告 `integrity=mismatch`、`reason=modified_or_mixed`。记录修复前后的哈希和修改内容，不重写安装清单来伪装原版；带修复的安装不能直接套用要求原始完整性的更新器，见[修复后的维护说明](openssh-pty-fix.zh-CN.md#回退与后续更新)。

## 4. 配置客户端公钥

将用户确认的单行 SSH 公钥放到沙箱的 `/workspace/client-access.pub`。这是 `.pub` 内容，不是客户端私钥，也不是 Tailscale auth key。下面的命令校验格式和指纹，再追加公钥并保留已有授权项：

```bash
python3 - <<'PY'
from pathlib import Path
import subprocess

public = Path('/workspace/client-access.pub')
key = public.read_text().strip()
if len(key.splitlines()) != 1 or not key.startswith(('ssh-', 'ecdsa-', 'sk-')):
    raise SystemExit('Expected one SSH public-key line; never supply a private key')
subprocess.run(['ssh-keygen', '-lf', str(public)], check=True)
target = Path('/workspace/.dots-tailscale/sshd/authorized_keys')
if target.is_symlink() or not target.is_file():
    raise SystemExit('Review the installed authorized_keys file before proceeding')
existing = target.read_text()
if key not in existing.splitlines():
    with target.open('a') as stream:
        stream.write(('' if not existing or existing.endswith('\n') else '\n') + key + '\n')
target.chmod(0o600)
print('Client public key is present in installed authorized_keys')
PY
```

完成条件为指纹与客户端实际使用的公钥一致，公钥已写入安装目录。只改 `/home/agent/.ssh/authorized_keys`，不能确保安装目录中的副本同步。

## 5. 启动并向用户发送认证链接

已有节点先在沙箱读取状态；需要登录时 CLI 可能非零退出，仍要检查实际输出：

```bash
/workspace/.dots-tailscale/dots-ts cli status --json
```

旧安装没有 `dots-ts` 时，用同目录 CLI：

```bash
/workspace/.dots-tailscale/tailscale \
  --socket="/tmp/dots-tailscale-$(id -u)/tailscaled.sock" status --json
```

| 状态 | 动作 |
| --- | --- |
| `Running` | 已认证，跳过 `login`；需要时运行现有 `start.sh` 补齐组件，继续第 7 节 |
| `NeedsMachineAuth` | 告知用户需要管理员批准当前设备，等待批准，不重新注册 |
| `NeedsLogin` | 确认没有另一条 `login` 运行，再执行下方命令 |
| 未启动、socket 不存在 | 新安装可直接 `login` 启动组件；已有安装先运行普通 `start.sh`，再按状态决定是否登录 |
| 当前上下文只读、权限不足或看不到 socket | 回到平台原生终端检查，不直接重装或清除状态 |

在沙箱使用支持持续会话的命令工具，运行并持续读取输出：

```bash
bash /workspace/.dots-tailscale/start.sh login
```

一旦出现真实认证链接，立即原样发给用户，请用户在浏览器中核对账号和 tailnet 后完成认证。不要等命令退出才发送，也不要把链接仅留在 Agent 的工具输出里。可按下面的格式交接，替换为本次真实链接：

> 请打开本次沙箱的 Tailscale 认证链接：`<本次实际输出的完整认证URL>`。在浏览器中核对账号与 tailnet，完成后告诉我；我会继续检查节点状态和 SSH 连接。

没有链接时先检查错误和状态，不编造 URL，不把历史日志中的旧链接当作本次认证链接。已为 Running 的节点不需要人为制造认证链接。

工具不支持持续会话时，可在沙箱二选一地改用下面的后台方式。确认没有另一条 `login` 运行后执行一次，并记录 PID：

```bash
(
  set -euo pipefail
  umask 077
  R="/tmp/dots-tailscale-$(id -u)"
  test -d "$R"
  nohup bash /workspace/.dots-tailscale/start.sh login \
    > "$R/login-agent.log" 2>&1 < /dev/null &
  printf 'LOGIN_PID=%s\n' "$!"
)
```

随后读取输出，看到链接就发给用户：

```bash
tail -n 80 "/tmp/dots-tailscale-$(id -u)/login-agent.log"
```

前台与后台方式不能并发。日志仅留在私有运行目录，不提交 Git，也不公开整段内容。

## 6. 等待用户认证并确认结果

发送链接后等待用户操作，可以读取同一次登录的增量输出或只读状态。时间过去、命令退出或用户口头确认，均不能单独替代后端检查。

`login` 的 `tailscale up` 阶段有 180 秒时限，整个命令还包括其他启动与检查。超时后保留现有 state，先查状态：用户稍后完成认证，守护进程仍可能进入 Running。确实仍需登录且旧链接失效时，再发起一次 `login`，把新链接发给用户并说明旧链接已替换。

用户确认认证完成、原登录流程已结束后，在沙箱执行普通启动；若失败先处理该错误，再查状态：

```bash
bash /workspace/.dots-tailscale/start.sh
```

```bash
/workspace/.dots-tailscale/dots-ts cli status --json
```

不要在原 `login` 仍持有管理锁时并发执行 `start.sh`。旧安装缺少辅助工具时，使用第 5 节 CLI 替代命令。完成条件为有效状态中的 `BackendState` 是 `Running`；`NeedsMachineAuth` 仍需管理员批准，`NeedsLogin` 则表示认证尚未在这个节点生效。不通过删除状态、`--force-reauth` 或重建主机密钥绕过这些分支。

## 7. 取得连接信息并核对主机密钥

在沙箱执行，记录实际用户名、Tailscale IPv4 地址和 SHA256 指纹：

```bash
id -un
/workspace/.dots-tailscale/tailscale \
  --socket="/tmp/dots-tailscale-$(id -u)/tailscaled.sock" ip -4
ssh-keygen -lf /workspace/.dots-tailscale/sshd/ssh_host_ed25519_key.pub
```

以平台终端取得的指纹作为独立核验依据，不沿用其他沙箱的地址或指纹。客户端需在线加入有权访问该节点的 tailnet，访问策略允许 TCP 22。

在客户端替换以下占位值和私钥路径，保留交互输入进行首次连接：

```bash
ssh -T -o StrictHostKeyChecking=ask -o IdentitiesOnly=yes \
  -i ~/.ssh/id_ed25519 '<SANDBOX_USER>@<TAILSCALE_IP>' 'id; echo SSH_OK'
```

主机密钥提示处的指纹与平台终端取得的值逐字一致才接受。不一致则停止调查，不通过关闭校验或删除旧记录让命令表面成功。候选密钥扫描结果本身不能作为信任依据。

首次信任建立后，在客户端 `~/.ssh/config` 添加或更新同名 Host，使用真实值：

```sshconfig
Host dots-sandbox
    HostName <TAILSCALE_IP>
    User <SANDBOX_USER>
    Port 22
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    ConnectTimeout 30
    ConnectionAttempts 3
    ServerAliveInterval 20
    ServerAliveCountMax 6
    StrictHostKeyChecking yes
```

先核对已有同名配置，避免重复条目；用 `ssh -G dots-sandbox` 检查实际生效的主机、用户与密钥。私钥须与已授权公钥配对。

## 8. 验证命令与真实交互终端

在客户端先验证无 PTY 命令：

```bash
ssh -T -o BatchMode=yes dots-sandbox 'id; echo SSH_OK'
```

返回预期用户且输出 `SSH_OK`，才完成命令会话验证。再从真实本地终端连接：

```bash
ssh dots-sandbox
```

Agent 的命令工具必须分配本地 PTY。确实需要强制请求远端终端时，可用 `ssh -tt dots-sandbox`；它不能修复服务器端未映射 GID 的错误。

进入 shell 后执行 `tty`、`id` 和 `stat -c '%a %u %g' "$(tty)"`。应有 `/dev/pts/数字`，所属用户正确；应用上述兼容库的实例预期权限为 `600`、组为登录用户主组。再检查 Tab 补全、Ctrl+C 中断后会话是否仍可用。

若命令成功而终端立即断开，从平台终端读取 `/tmp/dots-tailscale-$(id -u)/sshd.log`，按第 3 节处理。banner、Running 或一次 `ssh -T` 成功，都不能替代这项验证。

## 9. 向用户交付结果

给出实际源码提交号、软件包与 Tailscale 版本、用户名与节点地址、主机指纹、客户端连接命令，以及命令会话和交互终端各自的结果。应用了 PTY 兼容库时，说明它造成的源码差异与更新条件。

浏览器确认、管理员批准或客户端验证未完成时，指出具体停在哪一步和下一动作。分别报告“组件已安装”“链接已发送”“节点已在线”“SSH 已验证”的实际状态，不把它们合并成一个成功结论。
