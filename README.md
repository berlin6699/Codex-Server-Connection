# Codex Server Connection

在 Windows 本机运行 ChatGPT/Codex 桌面端，同时把项目和命令运行在 SSH 服务器上的一套可复用方案。它也说明了如何在服务器没有直接代理出口时，经由本机的 Mihomo/Clash 提供临时网络代理，并避免与 Codex 桌面端的 SSH 连接冲突。

> 本文以 HKUST(GZ) HPC 登录节点为例；主机名、用户名、私钥路径和端口请按自己的环境替换。

## 原理

这里有两条彼此独立的 SSH 连接，不能共用同一个带固定 `RemoteForward` 的别名：

```text
┌─ A. 代理隧道（需要时才保持运行） ─────────────────────────────────┐
│  服务器 127.0.0.1:27897 ── SSH reverse forward ──> 本机 127.0.0.1:7897 │
│                                                    Mihomo / Clash    │
└─────────────────────────────────────────────────────────────────────┘

┌─ B. Codex 桌面端连接（始终使用无转发别名） ───────────────────────┐
│  ChatGPT Desktop ── SSH:22 ──> 服务器上的 Codex App Server          │
└─────────────────────────────────────────────────────────────────────┘
```

`RemoteForward 127.0.0.1:27897 127.0.0.1:7897` 的含义是：服务器进程连接 `127.0.0.1:27897` 时，流量被加密带回本机的代理端口 `127.0.0.1:7897`；外网响应沿同一隧道返回服务器。

Codex 桌面端会读取 Windows 的 `~/.ssh/config`。如果桌面端使用的 Host 也含有这条固定 `RemoteForward`，它每次新开 SSH 会话都会再次申请绑定服务器端的 `27897`。已有代理隧道占用该端口时，就会出现：

```text
remote port forwarding failed for listen port 27897
```

解决方式是将“代理隧道”和“Codex 桌面端”拆成两个 SSH Host 别名。

## 前置条件

- Windows 已安装 OpenSSH 客户端，且可通过密钥正常登录服务器。
- 本机 Mihomo/Clash 正在运行，并在 `127.0.0.1:7897` 提供 HTTP 或 mixed 代理。若端口不同，请替换下文的 `7897`。
- 服务器已安装并登录 Codex；检查命令：

  ```powershell
  ssh HPC3-HKUSTGZ-Codex "command -v codex && codex --version"
  ```

- 使用最新版 ChatGPT 桌面端，并具有 Codex 使用权限。

## 1. 配置两个 SSH 别名

编辑 Windows 文件 `C:\Users\<你的用户名>\.ssh\config`。以下示例中，代理端口是服务器侧 `27897`，本机 Mihomo/Clash 是 `7897`。

```sshconfig
# 只给服务器提供本机代理；在单独终端中保持运行。
Host HPC3-HKUSTGZ-Proxy
  HostName hpc3login.hpc.hkust-gz.edu.cn
  User <你的服务器用户名>
  IdentityFile C:/Users/<你的 Windows 用户名>/.ssh/id_ed25519
  RemoteForward 127.0.0.1:27897 127.0.0.1:7897
  ExitOnForwardFailure yes
  ServerAliveInterval 30
  ServerAliveCountMax 3

# 只给 ChatGPT/Codex 桌面端使用：绝不能包含 RemoteForward。
Host HPC3-HKUSTGZ-Codex
  HostName hpc3login.hpc.hkust-gz.edu.cn
  User <你的服务器用户名>
  IdentityFile C:/Users/<你的 Windows 用户名>/.ssh/id_ed25519
  ServerAliveInterval 30
  ServerAliveCountMax 3
```

若你已有旧别名（例如 `HPC3-HKUSTGZ`）带有 `RemoteForward`，可以保留它作为代理用途；务必在桌面端选择新的 `HPC3-HKUSTGZ-Codex`，而不是旧别名。

确认 Codex 专用别名没有携带转发：

```powershell
ssh -G HPC3-HKUSTGZ-Codex | Select-String '^(hostname|user|remoteforward|exitonforwardfailure) '
```

输出不应包含 `remoteforward`。

## 2. 启动本机到服务器的代理隧道（仅服务器需要代理时）

先启动 Mihomo/Clash，然后在一个单独的 PowerShell 窗口运行：

```powershell
ssh -N HPC3-HKUSTGZ-Proxy
```

`-N` 表示只建立隧道、不启动远端 Shell。保持这个窗口打开；关闭窗口或按 `Ctrl+C` 即会停止代理隧道。

在服务器上为需要走代理的命令设置环境变量：

```bash
export HTTP_PROXY=http://127.0.0.1:27897
export HTTPS_PROXY=http://127.0.0.1:27897
export http_proxy="$HTTP_PROXY"
export https_proxy="$HTTPS_PROXY"
export NO_PROXY=localhost,127.0.0.1,::1
export no_proxy="$NO_PROXY"
```

若希望每个登录 Shell 都生效，可以将这段写入远端 Shell 的启动文件；但只有在代理隧道运行时才应使用它。先做一次连通性测试：

```bash
curl -I --connect-timeout 10 https://api.openai.com
```

收到 `401 Unauthorized` 仍表示网络已经通了，只是该测试请求没有携带 API 凭据。

## 3. 连接 Codex 桌面端

1. 完全退出并重新打开 ChatGPT 桌面端。
2. 打开 **Settings → Connections → SSH**。
3. 添加或启用 `HPC3-HKUSTGZ-Codex`。
4. 选择服务器上的项目目录，或将同一 Git 仓库的对话 hand off 到该主机。

桌面端会通过 SSH 启动远端 Codex App Server；文件读取、命令运行和改动都会发生在服务器，而界面和审批仍在本机。官方要求远端登录 Shell 能在 `PATH` 中找到 `codex`。[OpenAI 文档：Remote connections](https://learn.chatgpt.com/docs/remote-connections)

## 4. 日常使用顺序

1. 启动本机 Mihomo/Clash。
2. 如果服务器需借用本机代理，在单独窗口运行 `ssh -N HPC3-HKUSTGZ-Proxy`。
3. 确认远端的代理环境变量已设置，或确认服务器可以直接联网。
4. 打开 ChatGPT 桌面端，始终选 `HPC3-HKUSTGZ-Codex`。
5. 结束工作后按需关闭代理隧道；不要把带代理转发的别名选为 Codex 主机。

## 排障

### `remote port forwarding failed for listen port 27897`

这通常不是密钥认证失败。它表示该远端端口已经被其他 SSH 隧道占用，而新连接又继承了同一条 `RemoteForward`。

使用无转发别名检查监听状态：

```powershell
ssh HPC3-HKUSTGZ-Codex "netstat -ltn 2>/dev/null | grep ':27897 ' || true"
```

- 有 `LISTEN`：旧代理隧道正在占用端口；Codex 桌面端必须改用无转发别名。
- 没有 `LISTEN` 但仍失败：检查服务器 SSH 策略是否禁用了 `AllowTcpForwarding` 或远端监听。

### 手动 `ssh` 也报 27897 错误

说明你使用的 Host 别名本身含有 `RemoteForward`；这是 SSH 配置自动应用的结果。改用 `HPC3-HKUSTGZ-Codex`，或临时忽略转发：

```powershell
ssh -o ClearAllForwardings=yes HPC3-HKUSTGZ-Proxy
```

### 桌面端找不到或无法启动远端 Codex

```powershell
ssh HPC3-HKUSTGZ-Codex "command -v codex && codex --version"
```

若找不到命令，确保 `codex` 已安装，且安装目录在远端**登录 Shell**的 `PATH` 中。若桌面端刚更新或刚修改 SSH 配置，重启桌面端后再添加该 Host。

## 安全注意事项

- `127.0.0.1` 只让代理端口监听在服务器回环地址，绝不要改成 `0.0.0.0` 或公开暴露 Codex App Server。
- HPC 登录节点是共享环境；同节点的其他本地进程理论上可能尝试访问回环端口。仅在必要时开启隧道，遵守集群政策，并使用受认证、受信任的本地代理。
- 不要把 API Key、`auth.json`、私钥、代理订阅链接或任何密码提交到此仓库。
- 使用最小权限的 SSH 账户和受信任的私钥；用完代理隧道及时关闭。

## 本次环境的验证记录

- 远端 `codex` 位于 `~/.local/bin/codex`，已能执行 `codex --version`。
- 服务器可直连 `api.openai.com`；因此 Codex 桌面端的 SSH 专用别名不需要 `RemoteForward`。
- `27897` 已有监听时，原带转发别名会与桌面端新 SSH 会话冲突；拆分别名后连接正常。
