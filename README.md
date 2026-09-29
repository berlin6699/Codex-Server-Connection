# Codex Server Connection

在 Windows 或 macOS 本机运行 ChatGPT/Codex 桌面端，同时把项目和命令运行在 SSH 服务器上的一套可复用方案。它也说明了如何在服务器没有直接代理出口时，经由本机的 Mihomo/Clash 提供临时网络代理，并避免与 Codex 桌面端的 SSH 连接冲突。

> 本文使用两个 SSH 别名连接同一台服务器：`HPC-HKUSTGZ` 给 VS Code / 普通 SSH 使用，`HPC-HKUSTGZ-Codex` 给 Codex 桌面端使用。主机名、用户名、私钥路径和端口请按自己的环境替换。

| SSH 别名 | 用途 | 是否建立代理转发 |
| --- | --- | --- |
| `HPC-HKUSTGZ` | VS Code Remote SSH、普通终端连接 | 是，连接时自动建立 `RemoteForward` |
| `HPC-HKUSTGZ-Codex` | Codex 桌面端连接 | 否，避免重复绑定代理端口 |

**平时连接 `HPC-HKUSTGZ` 就会自动启动代理转发，不需要再建第三个隧道别名，也不需要另外执行 `ssh -fN`。** 本机 Mihomo/Clash 必须保持运行。只连接 `HPC-HKUSTGZ-Codex` 不会启动代理；服务器需要代理时，要保持普通 SSH / VS Code 连接在线。

## 原理

这两个别名的 `HostName` 和 `User` 相同，连接的是同一台服务器，但各自承担不同用途：

```text
┌─ A. HPC-HKUSTGZ：VS Code / 普通 SSH，连接时自动转发 ──────────────┐
│  服务器 127.0.0.1:27897 ── SSH reverse forward ──> 本机 127.0.0.1:7897 │
│                                                    Mihomo / Clash    │
└─────────────────────────────────────────────────────────────────────┘

┌─ B. HPC-HKUSTGZ-Codex：Codex 桌面端，无固定转发 ─────────────────┐
│  ChatGPT Desktop ── SSH:22 ──> 服务器上的 Codex App Server          │
└─────────────────────────────────────────────────────────────────────┘
```

`RemoteForward 127.0.0.1:27897 127.0.0.1:7897` 的含义是：服务器进程连接 `127.0.0.1:27897` 时，流量被加密带回本机的代理端口 `127.0.0.1:7897`；外网响应沿同一隧道返回服务器。

Codex 桌面端会读取本机的 `~/.ssh/config`。如果桌面端使用的 Host 也含有这条固定 `RemoteForward`，它新开独立 SSH 连接时会再次申请绑定服务器端的 `27897`。已有代理隧道占用该端口时，就会出现：

```text
remote port forwarding failed for listen port 27897
```

解决方式是让 `HPC-HKUSTGZ` 在普通连接时建立转发，Codex 始终选择无转发的 `HPC-HKUSTGZ-Codex`。Codex 中运行的程序可以使用普通连接提供的代理，但要在远端设置代理环境变量。

## 前置条件

- 本机已安装 OpenSSH 客户端，且可通过密钥正常登录服务器。Windows 使用 PowerShell，macOS 使用终端。
- 本机 Mihomo/Clash 正在运行，并在 `127.0.0.1:7897` 提供 HTTP 或 mixed 代理。若端口不同，请替换下文的 `7897`。
- 服务器已安装并登录 Codex；检查命令：

  ```powershell
  ssh HPC-HKUSTGZ-Codex "command -v codex && codex --version"
  ```

- 使用最新版 ChatGPT 桌面端，并具有 Codex 使用权限。

## 1. 配置两个 SSH 别名

编辑本机 SSH 配置：

- Windows：`C:\Users\<你的用户名>\.ssh\config`。
- macOS：`~/.ssh/config`。

以下示例中，代理端口是服务器侧 `27897`，本机 Mihomo/Clash 是 `7897`。`~/.ssh/id_ed25519` 请替换为自己的私钥路径；两个别名的服务器地址、用户和私钥应保持一致。

```sshconfig
# VS Code / 普通 SSH 使用；连接时自动给服务器提供本机代理。
Host HPC-HKUSTGZ
  HostName server.example.edu
  User <你的服务器用户名>
  IdentityFile ~/.ssh/id_ed25519
  RemoteForward 127.0.0.1:27897 127.0.0.1:7897
  ExitOnForwardFailure yes
  ServerAliveInterval 30
  ServerAliveCountMax 3

# 只给 ChatGPT/Codex 桌面端使用：绝不能包含 RemoteForward。
Host HPC-HKUSTGZ-Codex
  HostName server.example.edu
  User <你的服务器用户名>
  IdentityFile ~/.ssh/id_ed25519
  ServerAliveInterval 30
  ServerAliveCountMax 3
```

若已有 `HPC-HKUSTGZ`，保留它给 VS Code 使用，并确认它包含所需的 `RemoteForward`；新增 `HPC-HKUSTGZ-Codex` 给 Codex 桌面端使用。修改配置后，已有 SSH 会话不会自动补上转发，需要重新连接一次 VS Code / SSH。

若同时连接 HPC2 和 HPC3，可按以下方式命名；每台服务器的两个别名都指向各自同一台服务器：

| 服务器 | VS Code / 普通 SSH | Codex | 服务器代理端口 |
| --- | --- | --- | --- |
| HPC2 | `HPC2-HKUSTGZ` | `HPC2-HKUSTGZ-Codex` | `27898` |
| HPC3 | `HPC3-HKUSTGZ` | `HPC3-HKUSTGZ-Codex` | `27897` |

两台服务器均可转发到本机 `7897`。各自的 `RemoteForward`、远端代理环境变量和测试命令要使用对应的服务器端口。

确认 Codex 专用别名没有携带转发：

```powershell
ssh -G HPC-HKUSTGZ-Codex | Select-String '^(hostname|user|remoteforward|exitonforwardfailure) '
```

输出不应包含 `remoteforward`。macOS 可使用：

```bash
ssh -G HPC-HKUSTGZ-Codex | grep -E '^(hostname|user|remoteforward|exitonforwardfailure) '
```

## 2. 连接普通 SSH，自动建立代理转发

先启动本机 Mihomo/Clash，再在 VS Code 的 **Remote-SSH: Connect to Host** 中选择 `HPC-HKUSTGZ`，或在本机终端运行：

```bash
ssh HPC-HKUSTGZ
```

SSH 会自动读取这个别名里的 `RemoteForward`，把服务器的 `127.0.0.1:27897` 转发到本机 `127.0.0.1:7897`。保持承载转发的 SSH 连接在线即可，不用另外执行 `ssh -N` 或 `ssh -fN`。

连接时自动建立转发，与远端程序自动使用代理是两件事；远端程序仍需设置以下代理环境变量。

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
curl -i --connect-timeout 10 --max-time 20 https://api.openai.com/v1/models
```

收到 `401 Unauthorized` 仍表示网络已经通了，只是该测试请求没有携带 API 凭据。

若使用 Bash，希望新登录的终端在转发运行时自动使用代理，可在远端 `~/.bashrc` 中加入以下内容，并确认 `~/.bash_profile` 会加载 `~/.bashrc`（需要 `timeout` 和 Bash）：

```bash
# 只有本节点的转发端口存在时才设置代理；HPC2 请把 27897 改为 27898。
if command -v timeout >/dev/null 2>&1 && \
   timeout 1 bash -c 'exec 3<>/dev/tcp/127.0.0.1/27897' 2>/dev/null; then
    export HTTP_PROXY=http://127.0.0.1:27897
    export HTTPS_PROXY="$HTTP_PROXY"
    export http_proxy="$HTTP_PROXY"
    export https_proxy="$HTTPS_PROXY"
    export NO_PROXY=localhost,127.0.0.1,::1
    export no_proxy="$NO_PROXY"
fi
```

已有终端需执行 `source ~/.bashrc` 才能加载新设置；已经运行的 Codex App Server 也不会自动更新环境变量，必要时重新连接 Codex。检测到端口监听只说明转发端口存在，本机 Clash 是否可用仍以实际请求测试为准。若转发已断开，已有进程的代理变量不会自动清除；当前 Shell 可用 `unset HTTP_PROXY HTTPS_PROXY http_proxy https_proxy` 关闭代理。

HPC 的登录节点与计算节点各有自己的回环地址。如果命令在计算节点或另一个登录节点运行，`127.0.0.1:27897` 不会自动指向建立转发的登录节点；以上配置与测试适用于转发实际所在的同一节点。

## 3. 连接 Codex 桌面端

1. 完全退出并重新打开 ChatGPT 桌面端。
2. 打开 **Settings → Connections → SSH**。
3. 添加或启用 `HPC-HKUSTGZ-Codex`。
4. 选择服务器上的项目目录，或将同一 Git 仓库的对话 hand off 到该主机。

桌面端会通过 SSH 启动远端 Codex App Server；文件读取、命令运行和改动都会发生在服务器，而界面和审批仍在本机。官方要求远端登录 Shell 能在 `PATH` 中找到 `codex`。[OpenAI 文档：Remote connections](https://learn.chatgpt.com/docs/remote-connections)

## 4. 日常使用顺序

1. 启动本机 Mihomo/Clash。
2. 通过 VS Code 或 `ssh HPC-HKUSTGZ` 连接服务器，代理转发随连接自动启动。
3. 确认远端代理环境变量已设置并测试连通性；服务器可直接联网时可跳过代理步骤。
4. 打开 Codex 桌面端，选择 `HPC-HKUSTGZ-Codex`。
5. 需要代理期间保持普通 SSH / VS Code 连接在线；结束后关闭承载转发的连接。

只有在不打开 VS Code、也不需要远端 Shell，但服务器仍需借用本机代理时，才可单独运行 `ssh -N HPC-HKUSTGZ`。它只是同一个普通别名的可选用法，不是日常连接的必需步骤；已有连接占用转发端口时不要再启动。

## 排障

### `remote port forwarding failed for listen port 27897`

这通常不是密钥认证失败。它表示该远端端口已经被其他 SSH 隧道占用，而新连接又继承了同一条 `RemoteForward`。

使用无转发别名检查监听状态：

```powershell
ssh HPC-HKUSTGZ-Codex "netstat -ltn 2>/dev/null | grep ':27897 ' || true"
```

- 有 `LISTEN`：已有普通 SSH / VS Code 连接正在提供转发，或端口被其他进程占用；Codex 桌面端必须改用无转发别名。再次用普通别名独立建立转发也可能冲突。
- 没有 `LISTEN` 但仍失败：检查服务器 SSH 策略是否禁用了 `AllowTcpForwarding` 或远端监听。

### 手动 `ssh` 也报 27897 错误

说明你使用的 Host 别名本身含有 `RemoteForward`；这是 SSH 配置自动应用的结果。改用 `HPC-HKUSTGZ-Codex`，或临时忽略转发：

```powershell
ssh -o ClearAllForwardings=yes HPC-HKUSTGZ
```

### 多个普通 SSH / VS Code 连接争用转发端口

固定的远端端口只能由一个监听者绑定。已经有 `HPC-HKUSTGZ` 连接提供代理时，其他仅需远端 Shell 的连接可使用无转发的 `HPC-HKUSTGZ-Codex`；它也能用于普通 SSH 登录。

macOS / Linux 的 OpenSSH 可在 `Host HPC-HKUSTGZ` 配置中加入连接复用，让同一台本机上的多个连接共享转发：

```sshconfig
  ControlMaster auto
  ControlPath ~/.ssh/hpc-%C
  ControlPersist 60
```

修改后重新连接。`ControlPersist 60` 会在最后一个会话结束后最多继续保持连接 60 秒。连接复用不跨设备生效；另一台设备若也尝试绑定同一个服务器代理端口，仍可能冲突。Windows 内置 OpenSSH 不应直接照搬这一段；可以保持一条普通连接提供转发，其他连接使用无转发别名。

### 系统代理模式能连，TUN/虚拟网卡模式不能连

这通常是本机 TUN 的 DNS 或分流路径问题，与服务器端代理隧道 `HPC-HKUSTGZ`、远端 Codex 安装和 `RemoteForward` 端口冲突是不同问题。Codex 专用 SSH Host 仍应保持不含 `RemoteForward`。

启用 Mihomo/Clash 的 TUN 时，DNS 查询可能被接管；`fake-ip` 会对未排除的域名返回虚拟地址。若 SSH 主机名依赖校园网、公司网或 VPN 的分流 DNS，而 Mihomo 只使用公网 DNS，主机名可能被解析成虚拟地址或得到 NXDOMAIN。此时 SSH 会在认证前失败。仅看到 `198.18.0.0/16` 的地址还不能证明配置错误；应确认 Mihomo 能否为该域名取得真实地址，以及实际命中了哪条路由规则。

若主机应走内网直连，在 Mihomo 的 DNS 配置中仅为该 SSH 主机名排除 fake-IP，并指定能解析它的内网 DNS：

```yaml
dns:
  fake-ip-filter:
    # 保留现有条目，并加入准确的 SSH 主机名。
    - server.example.edu
  nameserver-policy:
    'server.example.edu':
      # 替换成当前校园网、公司网或 VPN 实际提供的 DNS 地址。
      - 10.0.0.53
      - 10.0.0.54
```

不要把示例 DNS 地址照抄；先确认它们能解析该主机。不要为解决这个问题把 `RemoteForward` 加到 Codex Host，也不必删除通用的 `.edu.cn` 直连规则：直连规则只有在 DNS 能返回真实服务器地址时才能工作。Mihomo 的 `fake-ip-filter` 可排除指定域名的虚拟地址映射，`nameserver-policy` 可为域名指定解析器。如果另外配置了 `direct-nameserver`，还要设置 `direct-nameserver-follow-policy: true`，确保直连解析遵循域名策略；未配置 `direct-nameserver` 时可省略。[Mihomo DNS 配置](https://wiki.metacubex.one/config/dns/)、[DNS 解析流程](https://wiki.metacubex.one/config/dns/diagram/)

修改后重载 Mihomo 并清理 DNS/Fake-IP 缓存，再验证：

```powershell
Resolve-DnsName server.example.edu -Type A
ssh -vvv HPC-HKUSTGZ-Codex
```

第一条命令应返回真实服务器地址，而不是 Fake-IP 网段中的地址；SSH 调试应收到服务器的 SSH 版本标识并通过密钥认证。若 `Resolve-DnsName` 没有真实地址，先检查 TUN 使用的 DNS 是否可达、是否能解析该主机，再检查域名命中的路由规则。

### 桌面端找不到或无法启动远端 Codex

```powershell
ssh HPC-HKUSTGZ-Codex "command -v codex && codex --version"
```

若找不到命令，确保 `codex` 已安装，且安装目录在远端**登录 Shell**的 `PATH` 中。若桌面端刚更新或刚修改 SSH 配置，重启桌面端后再添加该 Host。

## 安全注意事项

- `127.0.0.1` 只让代理端口监听在服务器回环地址，绝不要改成 `0.0.0.0` 或公开暴露 Codex App Server。
- 共享登录节点中，同节点的其他本地进程理论上可能尝试访问回环端口。仅在必要时开启隧道，遵守服务器政策，并使用受认证、受信任的本地代理。
- 不要把 API Key、`auth.json`、私钥、代理订阅链接或任何密码提交到此仓库。
- 使用最小权限的 SSH 账户和受信任的私钥；用完代理隧道及时关闭。

## 本次环境的验证记录

- 远端 `codex` 位于 `~/.local/bin/codex`，已能执行 `codex --version`。
- Codex 专用别名不带 `RemoteForward`：需要代理时由普通连接提供，不需要代理时可直接连接。
- `27897` 已有监听时，原带转发别名会与桌面端新 SSH 会话冲突；拆分别名后连接正常。
- 本机 HPC2 已验证普通连接自动建立 `27898 → 7897` 转发，并在远端访问 `/v1/models` 收到 `401`。
- 本机 HPC3 已验证无转发的 Codex 别名可以登录，并执行远端 `codex --version`。
