# DeepSeek V4 Flash 0731 — 手动部署操作步骤（MiaAI 方案）

> 面向 2× DGX Spark（head = `192.168.177.11` / `spark-e3e1`，worker = `192.168.177.12`）。
> 本仓库即 MiaAI 最新方案（`start-deepseek-v4-flash-dspark.sh` + `docker-compose.dspark.yml`），
> 含 08-13 起的所有 hotfix（suppress-stops、issue#43 decode fairness、issue#26/36 prefix-cache 等）。
> 配套配置文件：`.env.dspark`（已按你的服务器实测填好）。

---

## 0. 关键结论（先读）

| 问题 | 结论 |
|---|---|
| `start.sh` 会联网下载模型吗？ | **不会**。start 只检查镜像存在、同步 compose、起容器。模型下载由 `prepare-dspark-model-cache.sh` 或 eugr `hf-download.sh` 单独完成。 |
| serve 阶段会联网吗？ | **不会**。`.env` 已设 `HF_HUB_OFFLINE=1` + `TRANSFORMERS_OFFLINE=1`。 |
| 但下载阶段呢？ | **会强制联网**：`prepare-dspark-model-cache.sh` 下载时用 `-e HF_HUB_OFFLINE=0`。所以下载必须连外网，serve 保持离线。 |
| 端口会变吗？ | 不变：API `8888`、master `25000`，与 eugr 一致。视觉默认关闭，不占 `8889`/`25100`。 |
| 容器名会冲突吗？ | MiaAI 用 `PROJECT_NAME=deepseek-v4-flash` → 容器 `deepseek-v4-flash-vllm-dspark-1`，与你现有 `vllm_node` 不同名。**但 8888 端口只有一方能用 → 必须先停旧的**。 |
| 需要 `-e HF_HUB_OFFLINE=1` 吗？ | 不用另传。`.env` 已含 `HF_HUB_OFFLINE=1`，compose 会读。 |

---

## 1. 提前准备（在 head + worker 分别执行）

### 1.1 部署目录（head, 放到 Downloads 下）
仓库所在目录：`/home/kenleo_dgx/Downloads/DeepSeek-v4-Flash-DSpark-2x-DGX-Spark`（与 `spark-vllm-docker` 同级）。
若服务器上还没有该仓库，从你的 fork 克隆并对齐：

```bash
cd /home/kenleo_dgx/Downloads
git clone git@github.com:Ken-Leo/DeepSeek-v4-Flash-DSpark-2x-DGX-Spark.git
cd DeepSeek-v4-Flash-DSpark-2x-DGX-Spark
git fetch origin main && git reset --hard origin/main
```

把本项目生成的 `.env.dspark` 放到这个目录（即 `/home/kenleo_dgx/Downloads/DeepSeek-v4-Flash-DSpark-2x-DGX-Spark/.env.dspark`）。


### 1.2 拉取运行时镜像（head 和 worker 都要）
```bash
docker pull ghcr.io/anemll/dspark-vllm-gx10:0.1.1
```
> 你的机器上已有所需镜像（`ghcr.io/anemll/dspark-vllm-gx10:0.1.1` 已在 head）。**worker 也要确认有**（`docker images` 查看）。

### 1.3 下载模型（两节点都需要完整 HF hub 缓存）

**你习惯用 eugr 的 `hf-download.sh`（推荐，下载+分发一步到位）**：

```bash
# 在 head 的 eugr spark-vllm-docker 目录
./hf-download.sh deepseek-ai/DeepSeek-V4-Flash-0731 -c --copy-parallel
```
> `-c --copy-parallel` 会下载到 head，再 rsync 分发到所有 worker，两节点缓存就绪。
> head 已缓存该模型（`models--deepseek-ai--DeepSeek-V4-Flash-0731` 存在），此命令会复用/补齐并分发到 worker。

**或使用 MiaAI 的 `prepare-dspark-model-cache.sh`（也会自动 head+worker 两节点下载）**：

```bash
# 官方版
./prepare-dspark-model-cache.sh --official
# 或非交互（用 .env 的 ABLITERATED）
./prepare-dspark-model-cache.sh --yes
```

> ⚠️ 选哪种都行，**二选一**即可。若用 eugr `hf-download.sh`，就**不要再跑** `prepare-dspark-model-cache.sh`（避免重复下载）。两者都保证 worker 有缓存。

### 1.4 关于 ABLITERATED（两套方案）

`.env.dspark` 里 `ABLITERATED=0`（官方）。**切换方案**：

```bash
# 方案一：官方（默认，当前 .env 已配好）
ABLITERATED=0

# 方案二：Keys abliterated（去审查权重）
ABLITERATED=1
# 模型 cache: eugr 命令下载 abliterated 权重
./hf-download.sh drowzeys/keys-DeepSeekV4-Flash-GA-0731-Dspark-Abliterated-32-32 -c --copy-parallel
```
> 改 `ABLITERATED` 后，需**重跑下载该版本权重**，再 `./stop-... && ./start-...` 完整重启。
> 两套权重缓存**可并存**（HF hub 里是不同的 `models--*` 目录），切换时只需改 flag + 重启，不用删缓存。

---

## 2. 部署前：确认网络/接口值正确

`.env.dspark` 已按实测填写，但你最好在 head 上复核一次真实值：

```bash
ip -o addr show | grep -E 'enp1s0f0np0|enP2p1s0f0np0'   # 应见 192.168.177.11 / 192.168.178.11
ls /sys/class/infiniband/                                 # 应见 roceP2p1s0f0/f1, rocep1s0f0/f1
ss -tlnp | grep -E ':8888|:25000'                          # 确认端口占用情况
```

> `NCCL_IB_HCA`/`NCCL_SOCKET_IFNAME` 我按 `rocep1s0f0`+`enp1s0f0np0` 填。若你 eugr 部署用的是别的口，请按 eugr `.env` 里的实际值改 `.env.dspark` 对应项。

---

## 3. 停掉当前 eugr 部署（用 eugr 官方停止命令）

**首选：直接调用 eugr 的 launch-cluster.sh stop（它会同时处理 head + worker 的 `vllm_node` 容器）**：

```bash
cd /home/kenleo_dgx/Downloads/spark-vllm-docker
./launch-cluster.sh stop
```

> `launch-cluster.sh stop` 内部调用 `cleanup`，会停止并移除 head 与所有 worker（来自 eugr `.env` 的 `CLUSTER_NODES`）上的 `vllm_node` 容器。
> 确认 8888 已释放：
> ```bash
> ss -tlnp | grep 8888 || echo "8888 free"
> ```

**兜底（若 launch-cluster.sh stop 异常时手动清理）**：
```bash
# head
docker stop vllm_node && docker rm vllm_node
# worker（按需）
ssh 192.168.177.12 'docker ps --filter "name=vllm_node"'
```
## 4. 启动（head 上执行 MiaAI 脚本）

```bash
cd /home/kenleo_dgx/Downloads/DeepSeek-v4-Flash-DSpark-2x-DGX-Spark
./start-deepseek-v4-flash-dspark.sh
```
该脚本会自动：
- 用 ssh 同步 compose + `.env.dspark` 到 worker（`WORKER_HOST=192.168.177.12`）
- 起 worker（rank1）+ head（rank0）容器
- 等待 API 就绪、跑一次最小 smoke chat

> 若 worker 的 `WORKER_DIR` 需要不同路径，设 `WORKER_SCRIPT_DIR` 后再启动。
> 容器名为 `deepseek-v4-flash-vllm-dspark-1`（head 和 worker 各一个）。

---

## 5. 验证

```bash
# 5.1 模型列表 + max_model_len=1048576
curl -fsS http://127.0.0.1:8888/v1/models

# 5.2 KV pool 日志
docker compose --env-file .env.dspark -f docker-compose.dspark.yml logs vllm-dspark \
  | grep -E "GPU KV cache size|Maximum concurrency"

# 5.3 官方 smoke
./smoke-deepseek-v4-flash-dspark.sh
```

---

## 6. 停止

```bash
# 项目名 deepseek-v4-flash 的容器
docker compose --env-file .env.dspark -f docker-compose.dspark.yml down
# worker 同步停
ssh 192.168.177.12 'cd /home/kenleo_dgx/Downloads/DeepSeek-v4-Flash-DSpark-2x-DGX-Spark && docker compose --env-file .env.dspark -f docker-compose.dspark.yml down'
```
> 或用仓库自带的 `./stop-deepseek-v4-flash-dspark.sh`（它 `docker rm -f` 兜底）。

---

## 7. 切回 eugr 方案（可逆，不破坏任何东西）

```bash
# 停 MiaAI 容器
docker stop deepseek-v4-flash-vllm-dspark-1   # (head)
# 用回 eugr recipe
cd /home/kenleo_dgx/Downloads/spark-vllm-docker
./run-recipe.sh deepseek-v4-flash-dspark-0731 --no-ray --port 8888 -d
```
> 两种镜像（Anemll / b12x）与两种缓存独立共存，切换只是停/起，不互相污染。

---

## 8. 端口/资源影响汇总

| 资源 | MiaAI 方案 | 你现有 | 冲突？ |
|---|---|---|---|
| API 端口 | `8888` | `8888` | 同端口，先停旧 |
| master 端口 | `25000` | `25000` | 一致 |
| 容器名 | `deepseek-v4-flash-vllm-dspark-1` | `vllm_node` | 无 |
| 镜像 | Anemll `0.1.1` | 同 | 复用 |
| 模型缓存 | `models--deepseek-ai--DeepSeek-V4-Flash-0731` | 同 | 复用 |
| 视觉端口 | 关闭（默认） | 无 | 不占用 |
