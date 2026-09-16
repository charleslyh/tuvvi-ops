# tuvvi-ops

tuvvi **worker 运维产物**发布仓库（公开、自动发布，由 [tuvvi] 主仓库 CI 在每次镜像发布时推送）。

> 本仓库只含部署脚本与 job 定义模板，**不含任何密钥**——所有 token 均由各运维机在本地配置注入。

## 下载运维包

无需任何 GitHub 权限，稳定地址永远指向最新版本：

```bash
curl -L -o worker-ops.zip \
  https://github.com/charleslyh/tuvvi-ops/releases/latest/download/worker-ops.zip
unzip worker-ops.zip -d worker-ops && cd worker-ops
```

也可在 [Releases](../../releases) 页面按版本下载（`worker-ops-<版本号>`，与 ghcr 镜像 tag 同源同版本）。

## 前置条件

| 依赖 | 说明 |
| --- | --- |
| `nomad` CLI | 版本与集群 server 一致（v2.0.x）；macOS 用 docker 包装（见下） |
| Docker | 拉取 worker 镜像（ghcr.io 私有镜像需 `docker login ghcr.io`） |
| 两个 token | 向管理员索取：`NOMAD_TOKEN`（集群 ACL）、`WORKER_TOKEN`（worker 注册凭据）；按插件另有 `TOKENHUB_API_KEY` / `ARK_API_KEY` |

## 快速开始

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

## 日常运维命令

```bash
./worker.sh deploy <plugin> [tag]   # 部署 / 升级（tag 可指定历史版本回退）
./worker.sh stop <plugin>           # 停单个 job
./worker.sh stop-all                # 停所有 worker job
./worker.sh status [plugin]         # 状态
./worker.sh logs <plugin>           # 跟踪日志
```

插件清单（`worker-<plugin>.hcl`）：`mock` / `vgen-tokenhub` / `vgen-volcengine` / `igen-tokenhub` / `render`（connector 系，无 GPU）与 `igen-comfyui` / `vgen-comfyui`（GPU，需 NVIDIA 节点）。

## worker.env 配置项

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

## 常见问题

- **`missing job file`**：当前目录没有 `worker-<plugin>.hcl`——重新下载运维包，或设 `JOBS_DIR`
- **镜像拉取失败（unauthorized）**：先在本机 `docker login ghcr.io`（需 read:packages PAT），或让节点管理员配置 `/etc/nomad.d/docker-auth.json`
- **job 一直 pending（No nodes eligible）**：集群没有可用算力节点——节点接入需管理员操作（Nomad client + Docker），见内部文档
- **升级后想回退**：`./worker.sh deploy <plugin> <旧tag>`，或让管理员 `nomad job revert`

## macOS 安装 nomad CLI

官方二进制在 macOS 会崩溃，推荐 docker 包装：

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
chmod +x ~/bin/nomad   # 确保 ~/bin 在 PATH 中
```

---

内部文档：集群架构、节点接入、ACL 管理见 tuvvi 主仓库 `ops/` 目录（需访问权限）。
