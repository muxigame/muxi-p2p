# muxi-p2p

端到端 P2P 隧道：把一台机器上的本地 TCP 服务（原本是 Minecraft，现在也用于 Agent Nexus
的远程桌面）接到另一台机器上，按顺序尝试 IPv6 直连、路由器端口映射（UPnP/PCP）、
UDP 打洞 + QUIC、TCP 打洞，最后走中转。

从 `better-mc-remake`（启动器与端侧隧道工具）和 `muxi-mc-signaling`（信令服务）中拆出，
只包含 P2P 部分，不含 Minecraft 启动器、游戏或账号代码。来源与拆分时的文件哈希见
[`SOURCE-PROVENANCE.json`](SOURCE-PROVENANCE.json)。

## 组成

| 目录 | 作用 |
|---|---|
| `src/MuxiP2P.Protocol` | 协议库：NAT 探测（`MUXI-REFLECT/1`）、UDP/TCP 打洞、QUIC 隧道（System.Net.Quic / msquic）、多路复用、端口映射 |
| `src/MuxiP2P.Client` | 连接端：向信令要候选、按线路优先级建链（`P2PTunnel`、`TunnelClient`、`TunnelProbe`） |
| `src/MuxiP2P.Agent` | 被连端 agent（`muxi-p2p-agent`）：监听隧道端口，把每条流接到本机目标端口；向信令上报候选、响应打洞 |
| `signaling` | Python 包 `muxi-p2p-signaling`：信令路由 `/api/v1/tunnel/*`（HMAC 鉴权、候选过滤、长轮询、打洞协调）和 `MUXI-REFLECT/1` 反射器 |

## 构建

```bash
dotnet build src/MuxiP2P.Agent/MuxiP2P.Agent.csproj
dotnet publish src/MuxiP2P.Agent/MuxiP2P.Agent.csproj -c Release -r linux-x64 -o out/linux-x64
dotnet publish src/MuxiP2P.Agent/MuxiP2P.Agent.csproj -c Release -r win-x64 -o out/win-x64
```

需要 .NET 9 SDK。QUIC 在 Windows 11 / Server 2022 上使用系统自带的 msquic；Linux 需要
`libmsquic`（Microsoft 软件源，或由宿主程序随包提供）。

信令测试：

```bash
cd signaling
pip install -r requirements-dev.txt
python -B -m unittest discover -s tests -v
```

## 一个仓库，按需取用

协议库、连接端、agent 和信令放在同一个仓库里，因为它们说的是同一套线上协议，必须一起
改、一起发版。引用方把整个仓库作为 submodule（建议放在 `third_party/muxi-p2p`），只构建
自己用到的那部分：

- 启动器、桌面客户端这类**连接端**：`ProjectReference` 到 `src/MuxiP2P.Client`（会带上 `src/MuxiP2P.Protocol`）。
- **被连端**：发布 `src/MuxiP2P.Agent` 得到 `muxi-p2p-agent`，或者引用协议库自己写。
- **信令**：`pip install ./third_party/muxi-p2p/signaling`，然后

  ```python
  from muxi_p2p_signaling import router          # FastAPI APIRouter
  app.include_router(router)
  ```

  独立部署：`uvicorn muxi_p2p_signaling.main:app --workers 1`（注册表在内存里，只能单进程）。

### 嵌入进别的服务

`muxi_p2p_signaling.configure()` 让宿主接管鉴权：

```python
configure(
    secret_provider=lambda server_id: token_of(server_id),   # 每台 agent 一把 token
    access_guard=require_signed_in_user,                     # async (request, server_id) -> None 或抛 HTTPException
    diagnostics=False,                                       # 关掉不鉴权的 /diag
)
```

不调用时行为和独立部署一致：一把共享密钥（`MUXI_TUNNEL_TOKEN` / `BMC_TUNNEL_TOKEN_FILE`），
`/client-token`、`/endpoints`、`/punch` 公开。

### 反射器

`python -m muxi_p2p_signaling.reflector` 监听 UDP 45701–45703 和 TCP 45711–45712，回答
“你的包从哪个地址来”。放在两个不同公网 IP 上效果最好，agent 和连接端用
`--reflector host1,host2` 指过来。也可以在宿主进程里 `await Reflector(...).start()`。

## Agent 参数

```
muxi-p2p-agent --port <隧道监听端口> --target 127.0.0.1:<本机目标端口> --token <共享密钥>
               [--control-plane <信令地址> --server-id <id> --reflector <反射器主机,...>]
```

`--token`（或 `MUXI_TUNNEL_TOKEN`）是两端和信令共享的 HMAC 密钥。没有配置信令时，连接端
必须知道 agent 的固定地址；配置后 agent 每 30 秒上报一次候选，地址轮换也能找到。没有
内置的反射器地址：不配 `--reflector`（或 `MUXI_REFLECTOR`）时不打洞，只有隧道和中转。

## 许可

Apache-2.0，见 [`LICENSE`](LICENSE)。
