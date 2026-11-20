# muxi-p2p

端到端 P2P 隧道的总仓：把一台机器上的本地 TCP 服务（原本是 Minecraft，现在也用于 Agent Nexus
的远程桌面和远程终端）接到另一台机器上，按顺序尝试 IPv6 直连、路由器端口映射（UPnP/PCP）、
UDP 打洞 + QUIC、TCP 打洞，最后走中转。

本仓只做编排，代码在两个子仓库里，以 submodule 挂载：

| 目录 | 仓库 | 内容 |
|---|---|---|
| `agent/` | [muxi-p2p-agent](https://github.com/muxigame/muxi-p2p-agent) | .NET：协议库（`MUXI-REFLECT/1` NAT 探测、UDP/TCP 打洞、QUIC、多路复用、端口映射）、连接端、被连端 `muxi-p2p-agent` |
| `signaling/` | [muxi-p2p-signaling](https://github.com/muxigame/muxi-p2p-signaling) | Python 包 `muxi-p2p-signaling`：信令路由 `/api/v1/tunnel/*`、`MUXI-REFLECT/1` 反射器 |

```bash
git clone --recurse-submodules https://github.com/muxigame/muxi-p2p.git
```

## 该引用哪个

- 两边都要用（既跑信令又带端侧，例如 Agent Nexus 网关和客户端、MC 启动器加官网）：引用本仓，
  `git submodule add https://github.com/muxigame/muxi-p2p.git third_party/muxi-p2p`，
  再 `git submodule update --init --recursive`。
- 只要信令（例如独立的信令服务）：直接引用 muxi-p2p-signaling。
- 只要端侧（例如只带连接端的客户端）：直接引用 muxi-p2p-agent。

本仓的 CI 用固定下来的两个子仓库版本一起构建和测试；升级其中一个时在这里提交新的 submodule
指针，CI 通过再让引用方跟进。

## 来源

从 `better-mc-remake`（启动器与端侧隧道工具）和 `muxi-mc-signaling`（信令服务）中拆出，
只包含 P2P 部分，不含 Minecraft 启动器、游戏或账号代码。来源与拆分时的文件哈希见
[`SOURCE-PROVENANCE.json`](SOURCE-PROVENANCE.json)（路径是拆分后在子仓库中的位置）。

Apache-2.0，见 [`LICENSE`](LICENSE)。
