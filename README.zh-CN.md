# dots-tailscale 中文说明

[English](README.md) · [한국어](README.ko.md)

本项目用于在 Dots 云电脑中，以普通用户运行 Tailscale 和 OpenSSH。安装文件、Tailscale 节点状态和 SSH 主机密钥默认放在 `/workspace/.dots-tailscale`；运行时的套接字、PID 文件和日志放在 `/tmp/dots-tailscale-当前用户UID`。

在没有 TUN 设备、不能使用 sudo，且公网出口需要经过 HTTP 代理的环境中，项目使用 Tailscale 用户态网络和本地 TLS/WebSocket 适配层建立连接。SSH 会话由本机 OpenSSH 服务处理，Tailscale SSH 功能关闭。本项目独立维护，并非 OpenAI 或 Tailscale 官方工具。

## 文档目录

| 文档 | 内容 |
| --- | --- |
| [安装与使用指南](docs/usage.zh-CN.md) | 依赖检查、首次安装、节点登录、SSH 公钥、客户端配置、文件传输和日常恢复 |
| [沙箱环境与限制](docs/sandbox-environment.zh-CN.md) | 操作系统、权限、文件系统、网络出口、资源和预装软件，以及检查范围 |
| [OpenSSH 交互终端修复](docs/openssh-pty-fix.zh-CN.md) | PTY 断连现象、GID 映射原因、兼容库源码、编译、加载、验证和回退 |
| [私有 HTTP API 使用方案（英文）](docs/private-http-api.md) | 通过可选应用代理访问 tailnet 内的服务，及对应的访问控制与验证方法 |

## 首次安装

以下命令面向尚未安装的 Linux x86_64 沙箱。先阅读[安装条件](docs/usage.zh-CN.md#安装条件与依赖检查)；已有安装应按[恢复与更新说明](docs/usage.zh-CN.md#环境恢复与版本更新)处理，避免覆盖节点状态或手工修复。

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

`main` 包含本 fork 的中文文档，会随提交变化；部署时应记录完整提交号。当前安装器沿用上游 v0.3.1 的实现，安装的 Tailscale 版本为 1.102.4。`login` 会输出登录链接，需要在浏览器中确认账号和所属 tailnet。

完成节点登录后，还要添加客户端 SSH 公钥并独立核对服务器指纹。启动输出中的 `ready` 只说明 Tailscale 后端及本地 SSH banner 检查通过，交互终端必须另行验证。

本次实测沙箱曾出现普通 SSH 命令成功、交互终端立即断开的情况，原因是 `tty` 组的 GID 没有映射。该沙箱已手工部署兼容库；本 fork 的安装器仍保持上游行为，**不会自动编译或加载该修复**。遇到相同日志时，按[修复文档](docs/openssh-pty-fix.zh-CN.md)处理。

## 运行与状态检查

```bash
bash /workspace/.dots-tailscale/start.sh
/workspace/.dots-tailscale/dots-ts cli status
/workspace/.dots-tailscale/dots-ts doctor
```

安装器不注册自动启动。环境更换后，只有平台保留了安装目录及有效凭据，才具备复用节点身份的条件；进程和 `/tmp` 下的运行文件仍需重新创建。本次没有进行沙箱销毁重建测试。

节点状态、SSH 私钥、本地 CA 私钥和 auth key 都是敏感文件，不应提交到仓库或复制到另一台同时运行的沙箱中。代码按 [MIT 许可证](LICENSE)提供。
