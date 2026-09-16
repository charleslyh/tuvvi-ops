# tuvvi-ops

tuvvi **worker 运维产物**发布仓库（公开、自动发布，由主仓库 CI 在每次镜像发布时推送）。

> 本仓库只含部署脚本与 job 定义模板，**不含任何密钥**——所有 token 均由各运维机在本地配置注入。

## 目录

- [下载运维包](#下载运维包)
- [运维机快速开始](#运维机快速开始)（部署 / 升级 / 停止 worker）
- [算力节点接入](#算力节点接入nomad-client)（让一台机器能跑 worker）
- [常见问题](#常见问题)

## 下载运维包

无需任何 GitHub 权限，稳定地址永远指向最新版本：

```bash
curl -L -o worker-ops.zip \
  https://github.com/charleslyh/tuvvi-ops/releases/latest/download/worker-ops.zip
unzip worker-ops.zip -d worker-ops && cd worker-ops
```

也可在 [Releases](../../releases) 页面按版本下载（`worker-ops-<版本号>`，与 ghcr 镜像 tag 同源同版本）。

## 运维机快速开始

前置条件：

| 依赖 | 说明 |
| --- | --- |
| `nomad` CLI | 与集群 server 同大版本（v2.0.x）；Linux 直接装官方包，**macOS 必须用 docker wrapper**（官方二进制会崩，见 [macOS 安装 nomad CLI](#macos-安装-nomad-cli)） |
| Docker | 拉取 worker 镜像（ghcr.io 私有镜像需 `docker login ghcr.io`，用 read:packages 的 PAT） |
| 两个 token | 向管理员索取：`NOMAD_TOKEN`（集群 ACL）、`WORKER_TOKEN`（worker 注册凭据）；按插件另有 `TOKENHUB_API_KEY` / `ARK_API_KEY` |

```bash
# 1. 配置（填 token；worker.env 含密钥，勿提交/勿共享）
cp worker.env.example worker.env
vim worker.env

# 2. 部署（默认 TAG = 本次发布版本；滚动替换，失败自动回滚）
./worker.sh deploy vgen-tokenhub

# 3. 验证
./worker.sh status              # 节点 + job 总览
./worker.sh logs vgen-tokenhub  # 跟踪 worker 日志（应有 heartbeat 200 / lease 204）
```

日常命令：

```bash
./worker.sh deploy <plugin> [tag]   # 部署 / 升级（tag 可指定历史版本回退）
./worker.sh stop <plugin>           # 停单个 job
./worker.sh stop-all                # 停所有 worker job
./worker.sh status [plugin]         # 状态
./worker.sh logs <plugin>           # 跟踪日志
```

插件清单（`worker-<plugin>.hcl`）：`mock` / `vgen-tokenhub` / `vgen-volcengine` / `igen-tokenhub` / `render`（connector 系，无 GPU）与 `igen-comfyui` / `vgen-comfyui`（GPU，需 NVIDIA 节点）。

### worker.env 配置项

| 键 | 必填 | 说明 |
| --- | --- | --- |
| `NOMAD_ADDR` | ✅ | Nomad 集群地址（运维侧入口，HTTP 4646） |
| `NOMAD_TOKEN` | ✅ | ACL token（向管理员索取） |
| `HUB_BASE` | ✅ | worker 连接的 Hub 地址 |
| `WORKER_TOKEN` | ✅ | worker 注册凭据 |
| `TAG` | — | 默认镜像 tag（缺省 `latest`；deploy 参数优先） |
| `REGISTRY` | — | 缺省 `ghcr.io/charleslyh/tuvvi` |
| `JOBS_DIR` | — | hcl 目录（缺省 = 脚本所在目录） |
| `TOKENHUB_API_KEY` / `ARK_API_KEY` | 按插件 | provider 密钥 |

同名**环境变量优先**于 worker.env（适合 CI / 一次性覆盖）。

## 算力节点接入（Nomad client）

让一台 Linux 机器成为可被调度的 worker 算力节点。**机器只需要 Docker + Nomad client，无需本仓库、无需任何代码**。

### 1. 安装 Docker 与 Nomad

```bash
# Docker（已装可跳过）
curl -fsSL https://get.docker.com | sh

# Nomad（官方 apt 源）
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install nomad
nomad version   # v2.0.x，与 server 同大版本
```

GPU 节点另需：NVIDIA 驱动 + `nvidia-container-toolkit`（`docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi` 跑通为准）。

### 2. 配置 client

```bash
sudo mkdir -p /etc/nomad.d /opt/nomad/data

# client 配置（servers 地址向管理员索取，通常是集群 RPC 端口 4647）
sudo tee /etc/nomad.d/nomad.hcl >/dev/null <<'EOF'
datacenter = "dc1"
data_dir   = "/opt/nomad/data"
bind_addr  = "0.0.0.0"

client {
  enabled = true
  servers = ["<nomad-server-rpc-addr>:4647"]
}
EOF

# ghcr 镜像拉取凭据（read:packages 的 PAT，向管理员索取）
sudo tee /etc/nomad.d/docker-auth.json >/dev/null <<'EOF'
{
  "ghcr.io": {
    "auth": "<echo -n '<github-user>:<PAT>' | base64 的输出>"
  }
}
EOF
sudo chmod 600 /etc/nomad.d/docker-auth.json

# 启动
sudo systemctl enable --now nomad
sudo journalctl -u nomad -f   # 看到 "node registration complete" 即接入成功
```

### 3. 网络要求

| 方向 | 目标 | 用途 |
| --- | --- | --- |
| 出站 | Nomad server **4647**（RPC） | 注册 / 心跳 / 领任务（**必需**） |
| 出站 | ghcr.io **443** | docker pull 镜像 |
| 出站 | Hub 地址（`HUB_BASE`） | worker 容器心跳 / 领 job |
| 入站 | 来自 server 的 4647 | `nomad alloc logs/exec` 等运维命令（建议放行） |

### 4. 验证

管理员在运维机执行 `./worker.sh status`（或 `nomad node status`），新节点应出现且 `ready`。之后部署的 job 会自动调度到该节点。

## 常见问题

- **`missing job file`**：当前目录没有 `worker-<plugin>.hcl`——重新下载运维包，或设 `JOBS_DIR`
- **镜像拉取失败（unauthorized）**：先在本机 `docker login ghcr.io`（需 read:packages PAT），节点侧则配好 `/etc/nomad/d/docker-auth.json` 后 `systemctl restart nomad`
- **job 一直 pending（No nodes eligible）**：集群没有可用算力节点——按上文接入，或找管理员
- **升级后想回退**：`./worker.sh deploy <plugin> <旧tag>`，或 `nomad job revert <job>`

## macOS 安装 nomad CLI

官方二进制在 macOS 会崩溃，用 docker wrapper（一次配置，体验与原生 CLI 无异）：

```bash
mkdir -p ~/bin
cat > ~/bin/nomad <<'EOF'
#!/bin/sh
ARGS=""
for v in $(env | sed -n 's/^$NOMAD_[A-Za-z0-9_]*$=.*/\1/p'); do
  ARGS="$ARGS -e $v"
done
exec docker run --rm -i $ARGS \
  -v "$PWD:$PWD" -w "$PWD" \
  hashicorp/nomad:2.0.6 "$@"
EOF
chmod +x ~/bin/nomad   # 确保 ~/bin 在 PATH 中（zshrc: export PATH="$HOME/bin:$PATH"）
```

---

内部文档：集群架构、ACL 管理见 tuvvi 主仓库 `ops/` 目录（需访问权限）。
