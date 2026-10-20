# OpenSSH 交互终端修复

[中文首页](../README.zh-CN.md) · [安装与使用指南](usage.zh-CN.md) · [沙箱环境与限制](sandbox-environment.zh-CN.md)

本文记录一次用户态 OpenSSH 的 PTY 兼容问题及其处理方法。实测环境为 Debian 13、OpenSSH 10.0p2，登录用户为 UID/GID 1000 的 `agent`。当前沙箱已经按本文方式修复；本仓库仅补充操作说明，没有修改安装器来自动应用补丁。

这里的做法针对“`tty` 组存在，但其 GID 没有映射”的具体条件。普通服务器、其他 OpenSSH 实现，或 PTY 失败原因不同的环境，不应直接套用。

## 故障现象与诊断依据

客户端执行以下无终端命令可以成功：

```bash
ssh -T dots-sandbox 'id; echo SSH_OK'
```

但在真实终端执行 `ssh dots-sandbox` 时，连接立即关闭。`-T` 禁用了 PTY 分配，所以前一项只能验证命令会话，无法覆盖交互登录。排查时必须一边复现交互连接，一边看服务端日志。

在沙箱的独立桌面终端执行：

```bash
tail -n 100 -f "/tmp/dots-tailscale-$(id -u)/sshd.log"
```

本次致命错误的形式为：

```text
chown(/dev/pts/…, 1000, 5) failed: Invalid argument
```

再检查用户、映射和组数据库：

```bash
id
cat /proc/self/uid_map
cat /proc/self/gid_map
getent group tty
```

本次 UID/GID 映射均为 `1000 0 1`，只覆盖命名空间内的 ID 1000；`tty` 组的 GID 是 5。OpenSSH 查询到这个组后，尝试把 PTY 组所有权改为未映射的 GID 5，内核返回 `EINVAL`，OpenSSH 随后终止该会话。

日志还可能出现 `Attempt to write login records by non-root user (aborting)`。在本次检查的 OpenSSH 源码中，该路径记录日志后返回，不是这次 PTY 会话退出的直接原因。不要仅凭其中的 `aborting` 字样，把修复方向转到登录记录或 PAM。

## 修复原理与作用范围

OpenSSH 的 `pty_setowner()` 会先调用 `getgrnam("tty")`。找到组时使用该组并设置模式 `0620`；没有找到组时，已有代码会回退到登录用户的主组并设置模式 `0600`。

兼容库通过 `LD_PRELOAD` 让非 root 进程的 `getgrnam("tty")` 返回空值，从而选择这条现有回退路径。其他组名仍交给 libc 查询。在本次环境中，回退组为已映射的 GID 1000，因此 PTY 可以正常分配。

它不会伪造 `chown()` 成功，也不修改公钥认证、PAM、组数据库或系统 OpenSSH 二进制。加载设置只放在项目的 sshd 启动命令前，不写入 `/etc/ld.so.preload` 或用户全局 shell 配置。效果依赖动态链接器和这里使用的 OpenSSH 调用路径。

源码依据为 OpenSSH Portable V_10_0_P2 的 [sshpty.c](https://github.com/openssh/openssh-portable/blob/V_10_0_P2/sshpty.c) 和 [loginrec.c](https://github.com/openssh/openssh-portable/blob/V_10_0_P2/loginrec.c)。

## 编译兼容库

先打开独立于 Tailscale SSH 的平台桌面终端，并保留该恢复入口。确认存在 GCC：

```bash
command -v gcc
/usr/sbin/sshd -V
```

如果下面的源码或共享库已经存在，先检查是否为已部署修复，不要覆盖正在使用的文件。对于尚未修复的默认安装，在桌面终端执行：

```bash
(
  set -euo pipefail
  umask 077
  D=/workspace/.dots-tailscale
  C="$D/sshd/pty-compat"
  test -f "$D/installed"
  test ! -e "$C/mapped-tty-group.c"
  test ! -e "$C/mapped-tty-group.so"
  install -d -m 700 "$C"
  cat > "$C/mapped-tty-group.c" <<'C_SOURCE'
#define _GNU_SOURCE
#include <dlfcn.h>
#include <grp.h>
#include <string.h>
#include <unistd.h>

/* Rootless sshd must use its account's mapped group for PTYs.
 * OpenSSH already falls back to pw_gid when no tty group is found. */
struct group *getgrnam(const char *name)
{
    static struct group *(*original)(const char *);
    if (geteuid() != 0 && strcmp(name, "tty") == 0)
        return NULL;
    if (!original)
        original = (struct group *(*)(const char *))dlsym(RTLD_NEXT, "getgrnam");
    return original ? original(name) : NULL;
}
C_SOURCE
  gcc -shared -fPIC -O2 -Wall -Wextra -Werror \
    -o "$C/mapped-tty-group.so" "$C/mapped-tty-group.c" -ldl
  chmod 600 "$C/mapped-tty-group.c" "$C/mapped-tty-group.so"
)
```

该源码与本次沙箱已部署版本一致。编译成功只说明共享库生成，仍需验证加载与实际 PTY 行为。

## 加载检查与启动脚本修改

先以当前非 root 用户运行一次局部加载检查。它只在子进程中调用 `getgrnam()`，不启动或停止服务：

```bash
LD_PRELOAD=/workspace/.dots-tailscale/sshd/pty-compat/mapped-tty-group.so \
  python3 - <<'PY'
import ctypes
import grp
import os

assert os.geteuid() != 0, 'Run as the sandbox account, not root'
libc = ctypes.CDLL(None)
lookup = libc.getgrnam
lookup.argtypes = [ctypes.c_char_p]
lookup.restype = ctypes.c_void_p
assert not lookup(b'tty'), 'tty lookup did not select the fallback'
group = grp.getgrgid(os.getgid()).gr_name
assert group != 'tty'
assert lookup(group.encode()), 'The account group must remain available'
print('PTY compatibility lookup checks passed')
PY
```

检查通过后，将安装后的 `setup.sh` 中这一条启动命令：

```bash
    /usr/sbin/sshd -f "$S/sshd_config" -E "$R/sshd.log" 9>&-
```

替换为：

```bash
    LD_PRELOAD="$S/pty-compat/mapped-tty-group.so${LD_PRELOAD:+:$LD_PRELOAD}" /usr/sbin/sshd -f "$S/sshd_config" -E "$R/sshd.log" 9>&-
```

这里保留已有的 `LD_PRELOAD` 值，但不将新库导出为全局变量。可用下面的命令完成有备份的精确替换；如果脚本内容不同，它会停止并要求人工核对，而不是继续猜测位置。

```bash
(
  set -euo pipefail
  D=/workspace/.dots-tailscale
  SETUP_BACKUP="$D/setup.sh.bak-pty-$(date -u +%Y%m%dT%H%M%SZ)"
  python3 - "$D/setup.sh" "$SETUP_BACKUP" <<'PY'
from pathlib import Path
import shutil
import subprocess
import sys

path, backup = map(Path, sys.argv[1:])
old = '    /usr/sbin/sshd -f "$S/sshd_config" -E "$R/sshd.log" 9>&-'
new = '    LD_PRELOAD="$S/pty-compat/mapped-tty-group.so${LD_PRELOAD:+:$LD_PRELOAD}" /usr/sbin/sshd -f "$S/sshd_config" -E "$R/sshd.log" 9>&-'
source = path.read_text()
if new in source:
    sys.exit('The PTY preload is already configured; inspect the existing deployment')
if source.count(old) != 1:
    sys.exit('Expected exactly one sshd launch command; review setup.sh manually')
if backup.exists():
    sys.exit('Backup already exists; choose another backup path')
candidate = source.replace(old, new, 1)
subprocess.run(['bash', '-n'], input=candidate, text=True, check=True)
shutil.copy2(path, backup)
path.write_text(candidate)
print(f'Backup: {backup}')
PY
  bash -n "$D/setup.sh"
)
```

保存打印的备份路径。该操作只改变下次 sshd 启动时的环境；已经运行的旧监听进程不会自动加载兼容库。

## 受控重启与连接验证

本次排障先在回环地址的独立测试端口 36457 上启动候选 sshd，确认兼容库可以解决 PTY 分配问题，再替换正式的 22 端口监听进程。测试使用独立 PID 和日志，验证后删除测试实例，Tailscale 和 shim 没有重启。这是本次已经完成的验证过程，不是安装器自动执行的检查。

实际应用修改时，从独立桌面终端读取监听 PID 并核对命令：

```bash
R="/tmp/dots-tailscale-$(id -u)"
SSHD_PID="$(cat "$R/sshd.pid")"
ps -p "$SSHD_PID" -o pid=,user=,args=
```

确认它属于当前用户，且命令指向 `/workspace/.dots-tailscale/sshd/sshd_config`，为项目的 sshd 监听进程。PID 文件可能过期，无法确认时不要执行下一步。不要使用 `pkill sshd`，以免影响其他会话或服务。

确认无误后，仍在这个桌面终端中执行：

```bash
kill -TERM "$SSHD_PID"
sleep 1
bash /workspace/.dots-tailscale/start.sh
```

随后在客户端的真实本地终端中检查两种会话：

```bash
ssh -T dots-sandbox 'id; echo SSH_OK'
ssh dots-sandbox
```

进入 shell 后执行 `tty`、`id` 和 `stat -c '%a %u %g' "$(tty)"`。本次预期 PTY 所有者为当前用户，组为其主组，权限为 `600`。还应测试 Tab 补全、Ctrl+C，以及空闲后继续输入；同时观察服务端日志是否再出现 PTY 的 `chown` 致命错误。

本次正式连接已验证：分配 `/dev/pts` 终端、账号 UID/GID 1000、Tab 补全、Ctrl+C 中断后保留 SSH 会话，以及约 65 秒空闲后的命令响应。这个结果覆盖当前沙箱，不包含后续镜像升级或沙箱销毁重建。

## 回退与后续更新

需要回退时，保持独立桌面终端可用，选定上一步记录的原始 `setup.sh` 备份并核对内容。将它恢复到 `/workspace/.dots-tailscale/setup.sh`，保持权限 `700`，运行 `bash -n`，然后按同样的 PID 核对流程，仅重启项目的 sshd 监听进程。

回退时不必删除共享库；只要启动语句不再加载它，新进程就不会使用。已建立的会话可能仍持有旧映射，所以不要在回退过程中以“清理文件”为由删除节点状态、SSH 主机密钥、公钥文件或 CA。

回退会恢复原始行为。在仍然缺少 `tty` GID 映射的环境中，交互登录故障也会回来；此时可继续用独立桌面终端或已验证的无 PTY 命令处理。

以后更新安装脚本时，必须重新检查 sshd 启动语句和兼容库是否保留。跨发行版或更换动态链接器、OpenSSH 版本后，重新编译并验证；本方案不替代更完整的用户命名空间映射配置。
