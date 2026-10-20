# 安装与使用指南

[中文首页](../README.zh-CN.md) · [沙箱环境与限制](sandbox-environment.zh-CN.md) · [OpenSSH 交互终端修复](openssh-pty-fix.zh-CN.md) · [English](usage.md)

本指南面向需要从本地终端访问 Dots 沙箱的用户。首次安装需要在平台提供的桌面终端中操作；该终端也是 SSH 故障时的恢复入口。客户端所在设备需要加入同一 tailnet，并具备访问沙箱 TCP 22 端口的权限。

这里的命令对应上游 v0.3.1 安装器和本 fork 的中文文档。已安装的沙箱可能有额外修复，更新前应先检查实际文件，不能直接用仓库脚本覆盖安装目录。

## 连接结构与目录用途

Tailscale 使用 `--tun=userspace-networking` 运行，不依赖 `/dev/net/tun`。它通过本地 `shim.py` 连接平台的 `browser-proxy:8889`。适配层处理该出口环境中 Tailscale 控制连接和 DERP 连接遇到的 TLS ALPN、HTTP Upgrade 与 WebSocket 兼容问题，具体实现见[适配层源码](../shim.py)。它不是面向任意程序的通用透明代理。

客户端通过 tailnet 访问沙箱，再由监听在 `127.0.0.1:22` 的 OpenSSH 处理登录。这里使用普通 OpenSSH 公钥认证，安装器明确设置 Tailscale 的 `--ssh=false`。

| 路径 | 用途 |
| --- | --- |
| `/workspace/dots-tailscale` | Git 源码目录，用于阅读、检查和暂存更新 |
| `/workspace/.dots-tailscale/setup.sh`、`start.sh` | 已安装的管理脚本与入口 |
| `/workspace/.dots-tailscale/tailscale`、`tailscaled` | CLI 和用户态守护进程 |
| `/workspace/.dots-tailscale/state` | 节点状态与认证材料 |
| `/workspace/.dots-tailscale/sshd` | OpenSSH 配置、主机密钥和允许登录的公钥 |
| `/workspace/.dots-tailscale/shim` | 代理适配代码、本地 CA 及其私钥 |
| `/tmp/dots-tailscale-$(id -u)` | 当前用户的运行时目录，包含套接字、PID 和日志 |

源码目录与安装目录承担不同职责。在源码目录执行 `git pull` 不会更新正在运行的安装；执行 `install` 也不会覆盖内容不同的已安装文件。

## 安装条件与依赖检查

当前安装器要求 Linux x86_64。需要 Bash、Python 3、`timeout`、`flock`、OpenSSL、`ssh-keygen` 和 `/usr/sbin/sshd`；下载与解包还需要 curl、tar、sha256sum，克隆源码需要 Git。安装器还会使用 `install`、`cmp`、`chmod` 等常见系统工具。

在沙箱终端检查关键依赖与端口条件：

```bash
uname -srm
id
for tool in bash python3 timeout flock openssl ssh-keygen /usr/sbin/sshd \
  curl tar sha256sum git install cmp; do
  command -v "$tool" || printf 'missing: %s\n' "$tool"
done
cat /proc/sys/net/ipv4/ip_unprivileged_port_start
```

`ip_unprivileged_port_start` 必须小于或等于 22，当前用户才能按本项目的配置绑定本地 22 端口。本次检查值为 `0`，因此普通用户可以绑定低端口；端口仍需处于空闲状态。脚本没有实现改用任意 SSH 端口的一键配置。

执行下面的下载测试，可以检查代理出口是否可用。它保留证书校验、丢弃响应正文，并限制连接与总耗时：

```bash
curl --fail --silent --show-error --location \
  --proxy http://browser-proxy:8889 --connect-timeout 5 --max-time 20 \
  --output /dev/null --write-out 'HTTP %{http_code}\n' \
  https://github.com/windddkz/dots-tailscale
```

新建 SSH 会话未必带有代理变量。后文的 Git 命令显式设置代理，安装器则通过 `UPSTREAM_PROXY` 指定下载及适配层的上游。不要通过关闭 HTTPS 证书校验来解决下载失败。

## 首次安装与节点登录

在没有已有安装的环境中执行：

```bash
cd /workspace
HTTPS_PROXY=http://browser-proxy:8889 HTTP_PROXY=http://browser-proxy:8889 \
  git clone --branch main --depth 1 https://github.com/windddkz/dots-tailscale.git
cd dots-tailscale
git rev-parse HEAD
bash tailscale-setup.sh check
bash tailscale-setup.sh install
bash /workspace/.dots-tailscale/start.sh login
```

保留 `git rev-parse HEAD` 输出的提交号，用于后续比对。若 `/workspace/dots-tailscale` 已存在，先检查其中的远端与本地修改，不要删除目录后重新执行上述命令。

三个管理动作的含义如下：

| 动作 | 行为 |
| --- | --- |
| `check` | 显示部分安装文件是否存在，条件允许时查询 Tailscale；不启动服务，也不完整检查依赖 |
| `install` | 下载并校验 Tailscale 归档，放置代码、配置和密钥；不启动进程 |
| `login` | 启动缺失组件，并显式发起 Tailscale 登录 |

`check` 即使输出 `missing:` 也可能返回 `0`。登录后显示 `running: Tailscale and local SSH ready`，表示后端为 Running 且本地 SSH banner 可读，随后仍需完成公钥认证与终端验证。

`login` 输出登录链接后，在浏览器中确认账号和 tailnet。需要管理员批准设备时，处理现有设备的批准请求；不要反复删除节点状态重新登录。日常启动使用不带 `login` 的命令：

```bash
bash /workspace/.dots-tailscale/start.sh
```

安装器支持下列环境变量；自定义配置要在安装与后续启动时保持一致。

| 变量 | 默认值与范围 |
| --- | --- |
| `DOTS_TS_DIR` | `/workspace/.dots-tailscale`；必须为无空白字符的绝对路径 |
| `UPSTREAM_PROXY` | `http://browser-proxy:8889`；当前实现要求无账号密码的 HTTP 代理 URL |
| `NODE_HOSTNAME` | `dots-sandbox`；Tailscale 节点名 |

适配层固定使用 18089 端口，OpenSSH 使用 22 端口。本项目按单实例设计，不应在同一环境重复部署多套相同监听端口的安装。

## SSH 公钥与首次连接

在客户端读取准备使用的公钥，例如：

```bash
cat ~/.ssh/id_ed25519.pub
```

将公钥作为完整的一行追加到沙箱中的 `/workspace/.dots-tailscale/sshd/authorized_keys`。仅传输 `.pub` 公钥文件，客户端私钥留在客户端。如果首次安装时 `$HOME/.ssh/authorized_keys` 已存在，安装器会复制它；后续修改 HOME 中的文件不会自动同步到安装目录。

在沙箱终端设置公钥文件权限并读取服务器指纹：

```bash
chmod 600 /workspace/.dots-tailscale/sshd/authorized_keys
ssh-keygen -lf /workspace/.dots-tailscale/sshd/ssh_host_ed25519_key.pub
bash /workspace/.dots-tailscale/start.sh check
```

记录输出中的 Tailscale 地址和 SHA256 指纹。在客户端，将下列 `<TAILSCALE_IP>` 换成实际地址；如用户不是 `agent`，一并替换用户名：

```bash
ssh -T -o StrictHostKeyChecking=ask -o IdentitiesOnly=yes \
  -i ~/.ssh/id_ed25519 'agent@<TAILSCALE_IP>' 'id; pwd; echo SSH_OK'
```

首次连接的指纹必须与平台桌面终端中独立取得的指纹一致，再确认接受。指纹不符时停止连接，先检查实际节点、密钥和安装目录。`ssh-keyscan` 的输出本身不能代替独立核验。

确认首次连接后，可在客户端 `~/.ssh/config` 中加入以下配置。若同名 Host 已存在，应编辑原配置，避免重复条目造成误判。

```sshconfig
Host dots-sandbox
    HostName <TAILSCALE_IP>
    User agent
    Port 22
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    ConnectTimeout 30
    ConnectionAttempts 3
    ServerAliveInterval 20
    ServerAliveCountMax 6
    StrictHostKeyChecking yes
```

`StrictHostKeyChecking yes` 依赖上一步已经记录的可信主机密钥。连接重试与保活可以处理部分短暂波动，不能修复 PTY 分配错误。

## 交互终端与文件传输验证

无终端命令与交互终端是两项独立检查，均在客户端执行：

```bash
ssh -T dots-sandbox 'id; echo SSH_OK'
ssh dots-sandbox
```

第二条命令应在真正的本地终端中运行。进入远端 shell 后执行 `tty` 和 `id`，检查是否取得 `/dev/pts/数字`；再验证 Tab 补全、Ctrl+C 中断和短时间空闲后的响应。自动化工具需要分配本地 PTY，必要时用 `ssh -tt` 辅助诊断，不能只用 `ssh -T` 判断交互登录成功。

若第一条成功，第二条立即断开，同时服务端出现 `chown(/dev/pts/…): Invalid argument`，请按 [OpenSSH 交互终端修复](openssh-pty-fix.zh-CN.md)检查 GID 映射。当前安装器不会自动应用兼容库。

公钥认证通过后，可以单独验证 SFTP；文件传输不需要交互 PTY：

```bash
sftp dots-sandbox
```

需要上传本地文件时，例如将已有的 `example.txt` 放入工作目录：

```bash
scp ./example.txt dots-sandbox:/workspace/
```

## 日常检查与故障定位

在沙箱终端执行：

```bash
/workspace/.dots-tailscale/dots-ts cli status
/workspace/.dots-tailscale/dots-ts doctor
/workspace/.dots-tailscale/dots-ts doctor --json
tail -n 100 "/tmp/dots-tailscale-$(id -u)/sshd.log"
tail -n 100 "/tmp/dots-tailscale-$(id -u)/tailscaled.log"
tail -n 100 "/tmp/dots-tailscale-$(id -u)/shim.log"
```

`doctor` 是当前执行上下文中的只读诊断，不会启动服务、登录或修复文件。`degraded` 可能仍返回退出码 `0`，应同时查看各项检查。套接字不可见时，可回到独立桌面终端再次检查；同一用户名和 UID 不保证两个执行上下文的挂载与套接字可见性相同。

| 现象 | 重点检查 |
| --- | --- |
| SSH 超时、节点离线 | 本地 Tailscale 状态、沙箱后端状态、tailnet 访问策略、shim 与 tailscaled 日志 |
| `Permission denied (publickey)` | 客户端实际选择的密钥、沙箱安装目录中的 `authorized_keys`、用户名 |
| 命令成功、交互 shell 断开 | PTY 错误、`gid_map` 与 `tty` 组；见修复文档 |
| 新 SSH 会话下载失败 | 是否显式设置平台 HTTP 代理，保持 TLS 校验开启 |
| `existing file differs` | 已有安装与待安装源码不同；先比对和备份，不通过删除状态解决 |
| 原先正常，环境变化后缺少 socket | 确认安装目录仍在，再在独立桌面终端运行 `start.sh` |

常见启动退出码为：`10` 需要登录、`11` 后端未就绪、`12` 在时限内未取得有效后端状态、`13` 需要管理员批准设备。显式认证失败也可能返回原命令的非零退出码，不能只凭单个数字覆盖日志中的具体原因。

## 可选应用代理与非交互认证

v0.3.1 的辅助工具可以为应用提供本地 SOCKS5/HTTP 代理，默认关闭。查看与保存配置的命令如下：

```bash
/workspace/.dots-tailscale/dots-ts proxy show
/workspace/.dots-tailscale/dots-ts proxy configure --socks-port 1080 --http-port 8080
```

`configure` 只保存配置；它不会立即开放监听端口。使用 `proxy apply` 需要重启 tailscaled，会中断依赖该节点的 SSH 连接，应先取得独立桌面终端，确认端口空闲，并按[英文应用代理说明](usage.md#development-helpers)准备回退。代理不自动接管其他应用，也不替代 tailnet 访问策略。

需要非交互登录时，先通过安全方式创建 auth key 私有文件，再执行：

```bash
chmod 600 /workspace/.dots-tailscale/auth.key
DOTS_TS_AUTH_KEY_FILE=/workspace/.dots-tailscale/auth.key \
  bash /workspace/.dots-tailscale/start.sh auth
```

文件须为当前用户拥有的普通文件，绝对路径及其父目录不能含符号链接，权限为 `600` 或 `400`。脚本通过文件路径传给 Tailscale，不需要把 key 放在命令行。上游文档未报告真实 auth key 注册验证，本次也未执行该注册流程；普通 `start` 不会自动使用 key。

## 环境恢复与版本更新

本项目把身份文件放在 `/workspace`，目的是在平台保留该目录时复用安装。目录可写不等于平台承诺永久保留。本次验证覆盖现有环境的运行和 OpenSSH 监听进程替换，没有主动销毁沙箱来验证重建恢复。

新环境中确认文件仍在后，先执行 `check`，再执行 `start.sh`。若提示已认证状态缺失、主机密钥丢失或 CA 文件不完整，保留现场并检查备份，不让安装器重新生成另一套身份替代现有节点。

需要观察 HOME 与工作区是否持续保留，可使用源码中的标记探针：

```bash
python3 /workspace/dots-tailscale/home-persistence-probe.py
```

探针首次运行会在 HOME 和 `/workspace` 写入标记与基线，后续比较的是这些标记；它不是只读命令，也不验证节点认证状态。结果应另存一份到沙箱外，避免基线一起丢失后无法比较。

更新时，先在源码目录之外保留私有回退副本，记录旧文件哈希、新提交号和新源码哈希。v0.3.1 的四个源文件应来自同一提交：`tailscale-setup.sh`、`shim.py`、`dots-ts`、`dots_proxy.py`。替换安装文件前，按[英文更新步骤](usage.md#update-installed-scripts)检查、暂存并准备独立终端恢复。

部署过 PTY 修复的安装还需要保留 `sshd/pty-compat` 和 `setup.sh` 中的作用域加载语句。直接覆盖 `setup.sh` 会丢失这条语句；重新执行 `install` 则可能因文件不同而停止。更新后重新验证节点状态、主机指纹、公钥认证和交互终端。

## 凭据与文件保护

`state`、SSH 主机私钥、shim CA 私钥、auth key、生成的回退副本及运行日志均不应进入 Git。安装目录的访问权限应由使用者控制，不能因为它位于 `/workspace` 就认为其中的文件天然私密。

SSH 配置沿用上游的 `StrictModes no`，因此更需要确认安装目录、公钥文件和私钥只能被预期用户修改。它与 PTY 修复是两回事；修改这个配置不能解决未映射 GID 引起的终端分配失败。

不要把同一份 Tailscale 节点状态复制到多台同时运行的沙箱。撤销访问时，可删除对应的 SSH 授权公钥；若设备本身不再可信，还应在 tailnet 管理端撤销该设备的访问。
