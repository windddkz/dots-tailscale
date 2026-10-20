# 沙箱环境与限制

[中文首页](../README.zh-CN.md) · [安装与使用指南](usage.zh-CN.md) · [OpenSSH 交互终端修复](openssh-pty-fix.zh-CN.md)

本文记录 2026-10-20 对一台已运行本项目的 Dots 沙箱所做的检查。操作通过 SSH 执行，内容包括系统元数据、命令版本、安装目录、进程配置和少量有时限的网络请求。该轮环境检查没有安装软件、调整隔离配置或主动重建环境；之前的 OpenSSH 修复单独记录在修复文档中。本文路径均指被检查的沙箱。

这是一台带桌面应用的 Debian 13 沙箱，软件种类较多，管理权限和网络出口受到约束。下面区分当前实际观察、由这些条件推导出的使用限制，以及仍未确认的行为；其他 Dots 实例或后续镜像可能不同。

## 操作系统与隔离方式

| 项目 | 当前观察 |
| --- | --- |
| 系统 | Debian GNU/Linux 13，trixie；版本字段为 13.6 |
| 内核与架构 | Linux 6.18.55，x86_64 |
| 登录账号 | `agent`，UID 1000，主 GID 1000，组列表为 1000 |
| PID 1 | Bubblewrap，进程名为 `bwrap`，不是 systemd |
| 命名空间参数 | 可见 `--unshare-user`、`--unshare-pid`、`--unshare-ipc`、`--unshare-uts` |
| 权限参数 | 可见 `--cap-drop ALL`、`--new-session`、`--die-with-parent` |
| 进程权限 | 当前 SSH 检查进程的 capabilities 各集合均为 0，`NoNewPrivs=1` |
| seccomp | 当前检查进程为 `Seccomp=0`、`Seccomp_filters=0` |
| 用户与组映射 | `uid_map` 和 `gid_map` 均仅有 `1000 0 1` |

映射中的三列分别表示本命名空间 ID 起点、外层命名空间 ID 起点和映射数量。`1000 0 1` 只覆盖当前的一个用户或组；它不能被理解为当前用户具备外层管理员权限。只读挂载、清零的 capabilities 和其他隔离条件仍然有效。

当前最直接的兼容问题是：系统组数据库能查到 GID 5 的 `tty` 组，但该 GID 没有映射。OpenSSH 给新 PTY 设置组所有权时会失败，导致交互登录断开，详见[修复过程](openssh-pty-fix.zh-CN.md)。

挂载路径中出现 `kata-containers`，提示外层可能还有 Kata 相关组件；本次没有足够证据还原完整虚拟化结构。当前直接确认的是内层 Bubblewrap 隔离。也没有从可见参数中确认单独的网络命名空间隔离，因此不根据网卡列表推断宿主机拓扑。

## 权限、系统服务与设备

当前用户没有 sudo，不能修改只读系统目录。虽然存在 `systemctl`，执行状态检查得到 `offline`，不能依赖常规 systemd 服务安装和开机启动。运行普通用户进程是可行的，项目的 shim、tailscaled 和 sshd 就是按这种方式启动。

没有暴露下列设备或管理接口：

| 接口 | 对当前使用方式的影响 |
| --- | --- |
| `/dev/net/tun` | 不能按常规方式创建 TUN 网卡，本项目使用 Tailscale 用户态网络 |
| `/dev/kvm` | 没有可供当前用户使用的 KVM 加速设备 |
| `/dev/fuse` | 存在 FUSE 辅助命令也不等于可以挂载 FUSE 文件系统 |
| `/var/run/docker.sock` | 没有现成 Docker 守护进程接口；本次也未找到 Docker/Podman 命令 |
| `/dev/dri`、`/dev/nvidia0` | 没有看到可用的 GPU 设备接口，不能据此使用硬件 GPU 加速 |

普通应用可以监听获准的端口。`ip_unprivileged_port_start` 为 `0`，所以当前账号可以启动本项目的本地 22 端口 SSH 服务；这一设置与其他环境中的“低端口必须使用 root”不同。端口必须没有冲突。

## 文件系统与可见资源

| 项目 | 当前情况 |
| --- | --- |
| `/usr`、`/opt`、`/etc` | 只读 overlay 挂载，并带有 `nosuid`、`nodev` |
| `/home/agent`、`/workspace` | 可写 overlay，共用同一存储层 |
| 可写磁盘层 | 约 32 GB，检查时约剩余 30 GB；两个目录不能分别计入一份容量 |
| `/tmp` | tmpfs，显示上限约 4.9 GB，使用内存资源 |
| `/dev/shm` | tmpfs，显示上限约 4.9 GB；不能把它与 `/tmp` 相加当成额外磁盘 |
| 可见 CPU | 9 个 |
| 可见内存 | 约 9.7 GiB，检查时可用约 8.5 GiB |
| swap | 0 |
| 打开文件数限制 | 软、硬限制均为 16384 |

较大的归档、源码和下载文件适合放在 `/workspace`，不要把 `/tmp` 当作大容量落盘目录。HOME 与工作区可写，只能说明当前执行上下文允许写入；跨环境销毁、重建后的保留策略需要另行验证。

当前上下文没有挂载可读的 `/sys/fs/cgroup`，因此无法读取实际 CPU、内存和进程配额。表中的数值是当前能看到的资源，不是平台承诺的独占额度。部分 ulimit 显示 unlimited，也不能排除外层存在限制。

## 网络出口与 TLS 行为

沙箱可见两个非回环接口和默认路由，DNS 服务器配置为 `168.63.129.16`。本次 `browser-proxy` 解析为 `172.31.7.93`，公网测试域名也能正常解析。代理名称比该次解析的 IP 更适合写进配置，地址可能随环境变化。

本次检查结果如下：

| 检查 | 结果与范围 |
| --- | --- |
| 去除代理变量后直连公网测试站点 HTTPS | 连接失败 |
| 经 `http://browser-proxy:8889` 请求同一站点的 HTTP、HTTPS | 均返回 200；HTTPS 证书校验通过 |
| 在代理 CONNECT 后检查测试站点 TLS | 证书签发者字段为 OpenAI，说明至少该目标经过 TLS 介入；协商 ALPN 为 `h2` |
| 直连一个 Cloudflare Tunnel 端点的 TCP 7844 | 报 `Network is unreachable` |
| 经平台代理 CONNECT 到该端点的 7844 | 返回 200；没有据此判定完整隧道已经建立 |

证书校验通过，表示当前系统信任收到的证书链，不等于 TLS 一定直达原站。代理是否对其他域名采取相同处理，以及是否允许特定 Upgrade、长连接和非 HTTP 流量，需要分别测试。CONNECT 返回 200 也不能代替后续 TLS 握手和应用协议验证。

新建 SSH 会话中没有自动出现 `HTTP_PROXY`、`HTTPS_PROXY` 等代理变量。需要下载时，可在该会话设置：

```bash
export HTTP_PROXY=http://browser-proxy:8889
export HTTPS_PROXY=http://browser-proxy:8889
```

这只对读取这些变量的程序有效，不会改变路由，也不会让任意 TCP/UDP 程序自动走代理。桌面程序的环境可能由平台另外注入，本次没有将 SSH 的环境变量检查推广到所有进程。

本项目启动 tailscaled 时，给它传入的是本地 `127.0.0.1:18089` 适配层及专用 CA bundle；适配层再连接平台代理。这些设置由启动脚本传给相关子进程，没有修改系统全局证书库或全局代理配置。

## 软件包管理与运行时路径

系统中确实有 apt、apt-get、dpkg，但 `/var/lib/dpkg/status`、`/var/lib/apt/lists` 等元数据未暴露。`dpkg-query` 没有列出软件包，不能解读为系统没有安装软件。`/usr/local/bin/apt` 和 apt-get 包装器处理的是平台镜像源配置，不提供权限提升。

结合只读的系统目录、缺失的软件包数据库及当前权限，不能按完整 Debian 服务器的方式管理系统包。本次未执行 `apt install`。用户目录中的虚拟环境和便携二进制是可考虑的安装方式，仍需满足架构、动态库、网络和存储条件。

SSH 的默认 PATH 为 `/usr/local/bin:/usr/bin:/bin:/usr/games`，没有包含 `/usr/sbin` 和 `/opt/codex` 下的运行时。因而 `command -v` 找不到一个命令，只能说明它不在当前 PATH 中。

| 运行时 | 路径与已确认内容 |
| --- | --- |
| 系统 Python | `/usr/bin/python3`，3.13.5；有 NumPy、SciPy 等，但没有 pip、ensurepip 和 pandas |
| 平台 Python | `/opt/codex/runtimes/codex-primary-runtime/dependencies/python/bin/python3`，3.12.14；带 pip 26.2.1、uv 0.12.23 及文档、数据、CAD 库 |
| 桌面自动化 Node | `/opt/codex/cua_node/bin/node`，24.21.0；同目录有 npm 11.19.0、npx、corepack |
| 平台 Node | `/opt/codex/runtimes/codex-primary-runtime/dependencies/node/bin/node`，24.19.0 |
| Codex CLI | `/opt/codex/bin/codex`，0.163.0 |
| OpenSSH 服务端 | `/usr/sbin/sshd`，10.0p2；不在默认 PATH 中 |

使用平台 Python 时，优先明确指定解释器，避免把它的库与系统 Python 3.13 混用。例如检查 pip 应运行平台解释器的 `-m pip --version`，而不是仅凭系统终端找不到 `pip` 就重新安装一套。

## 已确认的预装软件

以下清单依据命令版本、可执行文件、桌面入口和 Python 包元数据整理。列出的桌面软件已找到对应程序，但没有逐个打开并验证全部功能。

| 用途 | 软件及版本信息 |
| --- | --- |
| 编译与源码管理 | GCC/G++ 14.2、make、pkg-config、Git 2.52、GitHub CLI、Perl |
| 终端与排查 | Bash、curl、wget、OpenSSH、OpenSSL、jq、ripgrep、tree、htop、strace、iproute2、Bubblewrap |
| 浏览器与桌面 | Chromium 154、Xfce、Thunar、Mousepad、Xfce Terminal、Qt 5 Designer/Assistant/Linguist |
| 图像与视频 | GIMP 3、Inkscape、ImageMagick、Blender、Kdenlive、FFmpeg 7.1.5 |
| CAD 与电子设计 | FreeCAD、OpenSCAD、KiCad 及原理图、PCB、Gerber 辅助程序 |
| GIS 与科学可视化 | QGIS、GDAL 工具、ParaView、3D Slicer 5.10.0 |
| 文档与 OCR | Pandoc 3.1.11.1、Tesseract 5.5.0、Ghostscript、Poppler、LaTeX、pdfLaTeX、XeLaTeX、LuaLaTeX |
| 平台原生组件 | `/opt/codex/runtimes/codex-primary-runtime/dependencies/native` 下还有无界面 LibreOffice、Poppler、libheif、jxrlib 和 Git |
| 游戏引擎 | Godot 4.6.3 |
| Java | OpenJDK 21 运行时，未找到 `javac` 编译器 |

平台 Python 3.12 中可见的主要库包括：

| 用途 | 库 |
| --- | --- |
| 数据与计算 | NumPy 2.3.5、pandas 2.2.3、SciPy 1.17.0、Matplotlib 3.10.8、Seaborn 0.13.2、scikit-learn 1.8.0、SymPy |
| Office 与 PDF | openpyxl、xlsxwriter、python-docx、python-pptx、PyMuPDF、pdfplumber、pypdf、ReportLab |
| 自动化、网络与服务 | Playwright 1.62.0、httpx、aiohttp、FastAPI、Uvicorn、MCP SDK |
| CAD 与可视化 | CadQuery 2.7.0、build123d 0.10.0、ezdxf、VTK 9.3.1 |

系统 Python 3.13 另有 GDAL、pyproj、Plotly、PyQt5、wxPython、pikepdf 和 OCRmyPDF 等软件包。两套解释器的库和版本并不相同。

本次没有在这两套 Python 的包元数据中找到 PyTorch、TensorFlow 或 OpenCV。默认 PATH 中也没有找到 Rust、CMake、Ninja、tmux 或 rsync。另需注意，`/usr/bin/go` 实际是围棋游戏，不能把这个同名程序当成 Go 编译器。上述检查不排除某个应用内部还有自己的私有运行时。

## 现有部署与后续安装判断

当前 Tailscale 1.102.4 由本项目安装，shim、tailscaled 和本地 sshd 位于项目的安装与运行目录中；它们不应被列为基础镜像预装组件。OpenSSH PTY 兼容库是本次排障后额外部署的文件，也尚未集成到上游安装器。

普通用户二进制具备运行条件，并不意味着它的网络连接天然可用。例如 Cloudflare Tunnel 需要访问 7844 端口，当前官方 HTTP/2 连接实现直接拨号，不能仅靠 `HTTPS_PROXY` 变量适配；Xray 支持显式 HTTP 上游配置，但选定传输方式仍须通过该出口的 TLS 与应用协议验证。这里没有安装或验证这两个工具。

相关实现与要求可查阅 [Cloudflare Tunnel 网络要求](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/tunnel-with-firewall/)、[cloudflared 连接实现](https://github.com/cloudflare/cloudflared/blob/master/edgediscovery/dial.go)、[Xray HTTP 出站](https://xtls.github.io/en/config/outbounds/http.html)和 [Xray dialerProxy 配置](https://xtls.github.io/en/config/transports/sockopt.html)。

本次检查没有覆盖平台完整网络策略、所有图形程序的可用性、沙箱最长存活时间、销毁后的文件保留和外层资源配额。后续镜像更新时，应重新检查这些与部署直接相关的条件。
