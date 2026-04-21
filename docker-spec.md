# Open TTS Docker 镜像规范

> 版本: v1.0 · 最后更新: 2026-04-21
>
> 本规范定义了 `*-open-tts` 项目的 Docker 镜像构建与运行时约定,覆盖 Dockerfile 模板、entrypoint、基础镜像、依赖安装顺序、环境变量、卷挂载与 Compose 示例。

---

## 1. 交付物

每个项目**只提供一个** Docker 镜像,按上游引擎的硬件支持二选一:

| 引擎能力 | 发布镜像 | 基础镜像 |
|---|---|---|
| 支持 CUDA(绝大多数 TTS 引擎) | **CUDA 版本** | `nvidia/cuda:12.8.0-cudnn-runtime-ubuntu22.04` |
| 仅 CPU(或仅 Apple Silicon 等非 CUDA 加速) | **CPU 版本** | `python:3.10-slim-bookworm` |

对应 Dockerfile:`docker/Dockerfile`(单一文件,**无** `.cpu` / `.cuda` 后缀)。

**运行时回退**:CUDA 镜像在无 GPU 主机上可通过设置 `{ENGINE}_DEVICE=cpu` 回退到 CPU 推理(前提是引擎本身支持双路径),因此一份 CUDA 镜像即可覆盖 CPU/CUDA 两种部署场景。无需为 CPU 单独构建。

GHCR 发布标签策略见 `github-workflow-spec.md`。

---

## 2. `docker/` 目录结构

```
docker/
├── Dockerfile                       # 唯一 Dockerfile
├── entrypoint.sh
├── requirements.api.txt             # 必需,7 行基线见下文
├── requirements.engine.txt        # 可选,引擎特定额外依赖
└── docker-compose.example.yml       # 必需,示例 compose 文件
```

不允许在根目录放 Dockerfile、docker-compose 文件。

---

## 3. Dockerfile 模板:CUDA 版本(默认)

项目仅发布 CUDA 版本时使用此模板,文件命名为 `docker/Dockerfile`(不带后缀)。

**所有 Dockerfile 必须使用 multi-stage 构建**,通过 builder → runtime 两个阶段隔离构建时工具(`build-essential`、`git`、编译产物)和最终镜像。构建依赖只存在于 builder,不污染 runtime 层,典型可减少运行时镜像 1-3 GB。

```dockerfile
# syntax=docker/dockerfile:1.7

# ============================================
# 可配置 ARG(默认值应匹配上游引擎的 torch 约束)
#   CUDA_VERSION    — nvidia/cuda tag 主体,如 12.8.0 / 12.4.1 / 12.1.1
#   UBUNTU_VERSION  — nvidia/cuda tag 的 ubuntu 版本后缀,如 22.04 / 24.04
#   TORCH_CUDA_TAG  — PyTorch wheel 索引的 CUDA 标签,如 cu128 / cu124 / cu121
# CI 可通过 --build-arg CUDA_VERSION=12.4.1 --build-arg TORCH_CUDA_TAG=cu124 覆盖。
# ============================================
ARG CUDA_VERSION=12.8.0
ARG UBUNTU_VERSION=22.04
ARG TORCH_CUDA_TAG=cu128

# ======================================================================
# Builder: 安装 torch + 上游引擎 + API 依赖到独立 venv
# ======================================================================
FROM nvidia/cuda:${CUDA_VERSION}-cudnn-runtime-ubuntu${UBUNTU_VERSION} AS builder

ARG TORCH_CUDA_TAG

ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1

RUN apt-get update && apt-get install -y --no-install-recommends \
        python3.10 \
        python3.10-venv \
        python3.10-dev \
        python3-pip \
        build-essential \
        git \
        ca-certificates \
    && ln -sf /usr/bin/python3.10 /usr/local/bin/python \
    && ln -sf /usr/bin/python3.10 /usr/local/bin/python3 \
    && rm -rf /var/lib/apt/lists/*

# 独立 venv,便于整体复制到 runtime
RUN python -m venv /opt/venv
ENV PATH=/opt/venv/bin:$PATH
RUN pip install --upgrade pip

# PyTorch(版本与引擎约束对齐;CUDA 标签由 ARG 控制)
RUN pip install --index-url https://download.pytorch.org/whl/${TORCH_CUDA_TAG} \
        torch==X.Y.Z torchaudio==X.Y.Z

# 上游引擎(submodule)
WORKDIR /build
COPY {UpstreamModule} /build/{UpstreamModule}
RUN pip install /build/{UpstreamModule} && \
    pip install hf_transfer

# API 层依赖
COPY docker/requirements.api.txt /tmp/requirements.api.txt
RUN pip install -r /tmp/requirements.api.txt

# (可选)引擎特定依赖
# COPY docker/requirements.engine.txt /tmp/requirements.engine.txt
# RUN pip install -r /tmp/requirements.engine.txt

# 剥离 .pyc 缓存,减少 venv 体积
RUN find /opt/venv -type d -name __pycache__ -exec rm -rf {} + 2>/dev/null || true

# ======================================================================
# Runtime: 只保留运行所需的系统库 + 从 builder 复制好的 venv
# ======================================================================
FROM nvidia/cuda:${CUDA_VERSION}-cudnn-runtime-ubuntu${UBUNTU_VERSION}

ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8 \
    HF_HOME=/root/.cache/huggingface \
    HF_HUB_ENABLE_HF_TRANSFER=1 \
    PATH=/opt/venv/bin:$PATH

# 运行时只需 Python 解释器 + 音频库,不装 build-essential / git / pip 等
RUN apt-get update && apt-get install -y --no-install-recommends \
        python3.10 \
        ffmpeg \
        libsndfile1 \
        ca-certificates \
    && ln -sf /usr/bin/python3.10 /usr/local/bin/python \
    && ln -sf /usr/bin/python3.10 /usr/local/bin/python3 \
    && rm -rf /var/lib/apt/lists/*

# 从 builder 复制预装好的 venv(含 torch、上游、API 依赖)
COPY --from=builder /opt/venv /opt/venv

WORKDIR /opt/api

# 应用代码(放最后以利用缓存)
COPY app /opt/api/app
COPY docker/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

VOLUME ["/voices", "/root/.cache"]
EXPOSE 8000
ENTRYPOINT ["/entrypoint.sh"]
CMD ["uvicorn", "app.server:app", "--host", "0.0.0.0", "--port", "8000"]
```

**占位符说明**:
- `{ENGINE}` → 引擎名大写(如 `OMNIVOICE`, `QWEN3`, `FISHSPEECH`)
- `{UpstreamModule}` → submodule 目录名(保留原大小写,如 `OmniVoice`, `Qwen3-TTS`)

**ARG 默认值原则**:

- ARG 的**默认值**(`ARG CUDA_VERSION=...`)应直接满足上游引擎的硬性要求——直接 `docker build` 不传任何 `--build-arg` 也能得到**可用**镜像。
- 例:上游要求 `torch>=2.5,<2.6`,默认值应设为 `CUDA_VERSION=12.4.1 / TORCH_CUDA_TAG=cu124`;上游要求 `torch>=2.6`,默认值应设为 `CUDA_VERSION=12.8.0 / TORCH_CUDA_TAG=cu128`。
- ARG **不是**用来"让用户按需下调"——下调到不兼容版本会直接构建失败。ARG 的用途是当上游允许 CUDA 浮动时,下游可以为更老的 GPU(如 T4 只支持 CUDA 12.4 及以下)指定精确版本。

**CUDA 版本参考矩阵**(按上游引擎对 torch 的约束):

| PyTorch 约束 | `CUDA_VERSION` 默认值 | `TORCH_CUDA_TAG` |
|---|---|---|
| 2.4.x | `12.4.1` | `cu124` |
| 2.5.x | `12.4.1` | `cu124` |
| 2.6.x–2.9.x | `12.8.0` | `cu128` |

**builder 基础镜像选择**:默认 builder 与 runtime 使用同一 CUDA runtime 镜像(避免 glibc 差异)。只有当需要 `nvcc` 编译自定义 CUDA 扩展(如 flash-attn 源码编译)时,才将 builder 的 `FROM` 换成 `nvidia/cuda:${CUDA_VERSION}-cudnn-devel-ubuntu${UBUNTU_VERSION}`;runtime 保持 `runtime` 后缀不变。

---

## 4. Dockerfile 模板:CPU 版本(仅当上游不支持 CUDA)

**只有在上游引擎不支持 CUDA 时**才使用此模板(例如只支持 CPU 或 Metal 推理的轻量级引擎)。绝大多数项目**不**走这个分支——支持 CUDA 的引擎必须发布 CUDA 版本(CPU 运行通过 `{ENGINE}_DEVICE=cpu` 在 CUDA 镜像内实现)。

文件命名依然为 `docker/Dockerfile`(不带 `.cpu` 后缀),同样使用 multi-stage。

```dockerfile
# syntax=docker/dockerfile:1.7

# ============================================
# 可配置 ARG(默认值应满足上游引擎对 Python 版本的要求)
#   PYTHON_VERSION   — python 官方镜像的主版本,如 3.10 / 3.11 / 3.12
#   DEBIAN_CODENAME  — Debian 代号,如 bookworm / trixie
# CI 可通过 --build-arg PYTHON_VERSION=3.12 覆盖。
# ============================================
ARG PYTHON_VERSION=3.10
ARG DEBIAN_CODENAME=bookworm

# ======================================================================
# Builder
# ======================================================================
FROM python:${PYTHON_VERSION}-slim-${DEBIAN_CODENAME} AS builder

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1

RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential \
        git \
        ca-certificates \
    && rm -rf /var/lib/apt/lists/*

RUN python -m venv /opt/venv
ENV PATH=/opt/venv/bin:$PATH
RUN pip install --upgrade pip

# CPU-only PyTorch
RUN pip install --extra-index-url https://download.pytorch.org/whl/cpu \
        torch==X.Y.Z+cpu torchaudio==X.Y.Z+cpu

WORKDIR /build
COPY {UpstreamModule} /build/{UpstreamModule}
RUN pip install /build/{UpstreamModule} && \
    pip install hf_transfer

COPY docker/requirements.api.txt /tmp/requirements.api.txt
RUN pip install -r /tmp/requirements.api.txt

# (可选)引擎特定依赖
# COPY docker/requirements.engine.txt /tmp/requirements.engine.txt
# RUN pip install -r /tmp/requirements.engine.txt

RUN find /opt/venv -type d -name __pycache__ -exec rm -rf {} + 2>/dev/null || true

# ======================================================================
# Runtime
# ======================================================================
FROM python:${PYTHON_VERSION}-slim-${DEBIAN_CODENAME}

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8 \
    HF_HOME=/root/.cache/huggingface \
    HF_HUB_ENABLE_HF_TRANSFER=1 \
    PATH=/opt/venv/bin:$PATH \
    {ENGINE}_DEVICE=cpu \
    {ENGINE}_DTYPE=float32

RUN apt-get update && apt-get install -y --no-install-recommends \
        ffmpeg \
        libsndfile1 \
        ca-certificates \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /opt/venv /opt/venv

WORKDIR /opt/api
COPY app /opt/api/app
COPY docker/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

VOLUME ["/voices", "/root/.cache"]
EXPOSE 8000
ENTRYPOINT ["/entrypoint.sh"]
CMD ["uvicorn", "app.server:app", "--host", "0.0.0.0", "--port", "8000"]
```

**ARG 默认值**:`PYTHON_VERSION` 默认值应满足上游引擎最低 Python 要求(绝大多数情况为 `3.10`)。如上游明确需要 3.11+ (如 MOSS-TTS 的 3.12),将默认值直接设为 `3.12`,不要依赖调用方传 `--build-arg`。

---

## 5. Entrypoint 模板

`docker/entrypoint.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# 引擎默认值
: "${ENGINE_MODEL:=org/model-id}"
: "${ENGINE_DEVICE:=auto}"

# 服务默认值(不带引擎前缀)
: "${VOICES_DIR:=/voices}"
: "${HOST:=0.0.0.0}"
: "${PORT:=8000}"
: "${LOG_LEVEL:=info}"

export ENGINE_MODEL ENGINE_DEVICE VOICES_DIR HOST PORT LOG_LEVEL

if [ "$#" -eq 0 ]; then
  exec uvicorn app.server:app --host "$HOST" --port "$PORT" --log-level "$LOG_LEVEL"
fi
exec "$@"
```

**替换规则**:
- `ENGINE_*` → 实际引擎前缀(如 `OMNIVOICE_MODEL`, `OMNIVOICE_DEVICE`)
- `VOICES_DIR` **不带**前缀——声音目录挂载点是跨引擎的服务级约定,不随引擎名变化,避免多引擎共存时用户要为每个容器写不同变量名

**行为**:
- 无参数:启动 uvicorn。
- 有参数:`exec "$@"` 执行传入命令(便于 `docker run image python -c "..."` 调试)。

---

## 6. 基础镜像原则

### 6.1 CPU 镜像

- **统一使用 `python:3.10-slim-bookworm`**。这是 Debian bookworm + Python 3.10 的官方 slim 镜像,体积约 45MB,已经过 glibc 兼容性验证。
- 例外:若引擎对 Python 版本有硬要求(如 MOSS-TTS 需要 3.10+),可升级到 `python:3.12-slim-bookworm`。但版本下限不低于 3.10。
- 不使用 `alpine`(musl 与 PyTorch 兼容性问题)。

### 6.2 CUDA 镜像

- **统一使用 `nvidia/cuda:*-cudnn-runtime-ubuntu22.04` 系列**,而非 `devel`。`runtime` 变体已经包含 cuDNN 运行时库,体积比 `devel` 小 2-3GB。
- **不使用 `pytorch/pytorch`**:它的 CUDA 版本与 PyTorch 版本绑定得太死,且体积比官方 CUDA + 手装 torch 大。

### 6.3 多阶段构建(默认)

Multi-stage 构建是**所有 Dockerfile 的默认模式**,模板已在 §3 / §4 给出。核心原则:

- **builder 阶段**:装 `build-essential` / `git` / `python3.x-dev` / `pip`,把所有 Python 依赖安装到独立 `/opt/venv`,然后清掉 `__pycache__`。
- **runtime 阶段**:只装 Python 解释器 + `ffmpeg` + `libsndfile1` + `ca-certificates`,通过 `COPY --from=builder /opt/venv /opt/venv` 引入已装好的环境。

这样做的收益:

1. Runtime 镜像**不含** `build-essential` / `git` / `pip` 缓存 / 源码包,典型可省 1-3 GB。
2. 构建层缓存更好——builder 的系统包 / torch / 上游引擎层变化少,应用代码变化后只需重建 runtime 的最后两层。
3. CI 缓存可按 builder 阶段独立 scope,命中率更高。

**不要**为了省事而退回单阶段——即使是"简单项目",multi-stage 带来的 runtime 瘦身也是净收益。

---

## 7. 系统依赖基线

最少需要安装:

```
git
ffmpeg
libsndfile1
ca-certificates
build-essential      # CPU 基础镜像;CUDA 基础镜像含 gcc 无需显式装
```

其他依赖按需追加

---

## 8. 环境变量规范

### 8.1 镜像内预设(Dockerfile `ENV`)

所有镜像统一预设:

```
PYTHONUNBUFFERED=1
PYTHONDONTWRITEBYTECODE=1
LANG=C.UTF-8
LC_ALL=C.UTF-8
HF_HOME=/root/.cache/huggingface
HF_HUB_ENABLE_HF_TRANSFER=1
PIP_NO_CACHE_DIR=1
```

按镜像变体额外预设:

- **CUDA 镜像**(默认选择):**不**预设 `{ENGINE}_DEVICE` / `{ENGINE}_DTYPE`,由 Settings 的 `auto` 选择——有 GPU 时走 CUDA + `float16`,无 GPU 时回退 CPU + `float32`(具体由引擎实现决定)。这让同一个镜像能同时服务 GPU 和 CPU 部署。
- **CPU 镜像**(仅当上游不支持 CUDA 时):额外预设 `{ENGINE}_DEVICE=cpu` / `{ENGINE}_DTYPE=float32`,让用户无需额外配置即可运行。

### 8.2 用户可覆盖环境变量

**通用变量**(所有项目必须支持):

| 变量 | 默认 | 说明 |
|---|---|---|
| `{ENGINE}_MODEL` | 引擎具体值 | HuggingFace model id 或本地路径 |
| `{ENGINE}_DEVICE` | `auto` | `auto` / `cuda` / `cpu`(容器内不支持 `mps`,macOS 宿主的 Metal 不可达 Linux 容器) |
| `{ENGINE}_CUDA_INDEX` | `0` | 多卡场景指定 GPU |
| `{ENGINE}_DTYPE` | 变体相关 | `float16` / `bfloat16` / `float32` |
| `VOICES_DIR` | `/voices` | 声音挂载路径(**不带引擎前缀**,服务级共享约定) |
| `HOST` | `0.0.0.0` | 绑定地址 |
| `PORT` | `8000` | 绑定端口 |
| `LOG_LEVEL` | `info` | uvicorn 日志级别 |
| `MAX_INPUT_CHARS` | `8000` | 文本长度上限 |

**按需变量**(引擎具备相应能力时必须暴露,命名固定,不要用别名):

| 变量 | 暴露条件 | 取值 | 说明 |
|---|---|---|---|
| `{ENGINE}_VERSION` | 单镜像承载多个模型版本(调用接口不同) | 引擎定义的版本号,如 `"2"`、`"3"`、`"v3.0.3"` | 用于在运行时分派到不同的加载路径。例:CosyVoice 的 v2 与 v3 API 略有不同,同一镜像需要通过此变量切换 |
| `{ENGINE}_VARIANT` | 引擎有多个功能变体,同一镜像承载全部 | 引擎定义的变体 id,如 `tts` / `ttsd` / `voicegen` / `sfx` | **只负责分派功能逻辑(启用哪些端点、用哪条推理路径),不决定模型权重**——权重由 `{ENGINE}_MODEL` 指定,两者正交。例:MOSS-TTS 的 `tts` / `ttsd` / `voicegen` / `sfx` 对应不同的功能模式,每个变体再配合 `MOSS_MODEL` 指向该变体支持的权重 |
| `{ENGINE}_QUANTIZATION` | 引擎支持权重量化 | `none` / `int8` / `int4` 等 | **仅 CUDA 镜像支持**。CPU 镜像必须忽略此变量并按 `float32` 加载(容器内无 bitsandbytes 等加速库的 CPU 后端)。缺省等同 `none`,按 `{ENGINE}_DTYPE` 加载 |

**命名约定**(其他引擎特有变量):

- 所有引擎特有的环境变量**必须**带 `{ENGINE}_` 前缀,避免与其他 TTS 项目或系统工具冲突。例如 `FISHSPEECH_COMPILE`、`QWEN3_ATTN_IMPLEMENTATION`、`CHATTERBOX_EXAGGERATION_DEFAULT`。
- 不要使用 `MODEL_VERSION`、`VARIANT` 这类无前缀命名——哪怕语义清楚,也会破坏多引擎共存的部署场景。
- 变量**全大写、下划线分词**,与 `Settings` 字段 `case_sensitive=False` 的小写蛇形形成一一映射(见 `project-layout-spec.md` §3.3)。

---

## 9. 卷挂载规范

| 挂载点 | 宿主侧建议 | 模式 | 用途 |
|---|---|---|---|
| `/voices` | `./voices` | `:ro` | 声音文件 |
| `/root/.cache` | `./cache` | 读写 | 模型缓存。**HuggingFace Hub 与 ModelScope 的缓存统一落到此目录下**,由各 SDK 自行创建 `huggingface/` / `modelscope/` 子目录 |

**模型来源规范**:所有引擎的模型权重必须走 HF Hub / ModelScope SDK 拉取,缓存到 `/root/.cache` 下。如果某个引擎确实需要从本地磁盘加载非 SDK 管理的权重文件,由用户自行通过 `-v ./my-ckpts:/any/path` 额外挂载,并在 `{ENGINE}_MODEL` 中给出该路径。规范不预留此位置。

**首次启动下载**:容器启动时若 `/root/.cache` 下未命中 `{ENGINE}_MODEL` 对应的权重,引擎会在 `__init__` 阶段**自动**触发 SDK 下载(见 `project-layout-spec.md` §3.7 第 7-12 项)。此时 `/healthz` 返回 `status="loading"`。**强烈推荐**挂载该目录(`-v ./cache:/root/.cache`)——否则每次容器重启都会重新下载数 GB 权重。离线部署场景设置 `HF_HUB_OFFLINE=1` 让 SDK 只读缓存、不尝试联网。

**不要**挂载 `/opt/api` — 应用代码应在镜像内,避免本地修改污染运行时。

---

## 10. Docker Compose 示例

`docker/docker-compose.example.yml`(CUDA 镜像版本,默认场景):

```yaml
# docker compose -f docker/docker-compose.example.yml up

services:
  {engine}:
    image: ghcr.io/{owner}/{engine}-open-tts:latest
    container_name: {engine}
    ports:
      - "8000:8000"
    volumes:
      - ./voices:/voices:ro
      - ./cache:/root/.cache
    environment:
      {ENGINE}_MODEL: "org/model-id"
      LOG_LEVEL: "info"
      # CUDA 镜像默认走 GPU;无 GPU 主机取消注释回退 CPU:
      # {ENGINE}_DEVICE: "cpu"
      # {ENGINE}_DTYPE: "float32"
    # 仅 CUDA 镜像需要 deploy 段;CPU-only 版本删除此段
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    restart: unless-stopped
```

**约束**:
- 使用 `services:` 顶层,不使用已过时的 `version:` 字段。
- 每个项目只声明**一个** service(与单镜像策略一致)。
- GPU 声明使用现代 `deploy.resources.reservations.devices` 语法,不用 `runtime: nvidia`(已弃用)。
- CPU-only 项目(`Dockerfile` 以 `python:3.10-slim-bookworm` 为基础)的 compose 示例需删去 `deploy` 段,删除 `{ENGINE}_DEVICE` / `{ENGINE}_DTYPE` 的注释(这两项已在镜像内预设)。

---

## 11. 可选高级模式

### 11.1 `uv` 替代 `pip`

适用于有复杂依赖树 + 大型 CUDA 层的项目(如 fishspeech)。

**优势**:分层缓存更细,源码改动不触发依赖重装。

**代价**:构建逻辑复杂度上升,需要 `pyproject.toml` + `uv.lock`。

模板片段:

```dockerfile
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

ENV UV_LINK_MODE=copy \
    UV_PYTHON_DOWNLOADS=never \
    UV_COMPILE_BYTECODE=1

# 仅 lock + 清单层(变动少)
WORKDIR /opt/api
COPY pyproject.toml uv.lock /opt/api/
RUN uv sync --frozen --no-install-project --extra cu128

# 源码层
COPY {UpstreamModule} /opt/api/{UpstreamModule}
COPY app /opt/api/app
RUN uv sync --frozen --extra cu128
```

### 11.2 非 root 用户

```dockerfile
RUN useradd -m -u 1000 -s /bin/bash api
USER 1000
WORKDIR /home/api
# 将 /root/.cache 替换为 /home/api/.cache
```

注意需要配合调整 `HF_HOME`、`VOLUME` 声明。

### 11.3 `HEALTHCHECK`

推荐:

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=120s --retries=3 \
    CMD curl -fsS http://localhost:8000/healthz | grep -q '"status":"ok"' || exit 1
```

`start-period=120s` 给模型加载留出时间。需要在基础镜像中安装 `curl`(默认 slim 镜像不含)。

### 11.4 多架构(`linux/arm64`)

启用 ARM64 支持:

1. 上游引擎须有 ARM wheel(大多数 PyTorch 模型满足)。
2. GitHub Actions 中使用 `docker/setup-qemu-action@v3`。
3. `docker/build-push-action` 设置 `platforms: linux/amd64,linux/arm64`。

注意:`nvidia/cuda` 基础镜像**不提供 ARM 版本**,因此 CUDA 镜像只能是 `linux/amd64`。多架构构建仅对 CPU 版本项目有效。

---

## 12. 构建与推送

### 12.1 本地构建

使用默认 ARG(即引擎要求的 CUDA / Python 版本)构建:

```bash
docker build -f docker/Dockerfile -t {engine}:local .
```

需要覆盖 CUDA / Ubuntu / Python 版本时通过 `--build-arg`:

```bash
# CUDA 版本:给老 GPU(例如 T4,仅 CUDA 12.4)生成镜像
docker build -f docker/Dockerfile \
  --build-arg CUDA_VERSION=12.4.1 \
  --build-arg TORCH_CUDA_TAG=cu124 \
  -t {engine}:cu124 .

# CPU 版本:升级 Python 到 3.12
docker build -f docker/Dockerfile \
  --build-arg PYTHON_VERSION=3.12 \
  -t {engine}:py312 .
```

**重要**:`context` 必须是仓库根(`.`),因为 Dockerfile 需要 `COPY {UpstreamModule}` 和 `COPY app`。

**ARG 覆盖边界**:ARG 只控制基础镜像与 PyTorch wheel 索引;上游引擎的 `pyproject.toml` / `requirements` 中的 torch 版本号是**写死**的(见 §3 模板中的 `torch==X.Y.Z`),覆盖 `TORCH_CUDA_TAG` 但不改 `torch` 版本可能导致 wheel 不存在。版本矩阵越界时构建会直接失败——这是符合预期的。

### 12.2 CI 推送

统一通过 `.github/workflows/build-image.yml` 推送到 GHCR。详见 `github-workflow-spec.md`。

---

## 13. 检查清单

- [ ] 仅存在一个 `docker/Dockerfile`(无 `.cpu` / `.cuda` 后缀),按引擎能力选择 CUDA(§3)或 CPU(§4)模板
- [ ] 默认走 CUDA 版本;仅当上游引擎不支持 CUDA 时选 CPU 版本
- [ ] **使用 multi-stage 构建**:builder 阶段装编译工具与依赖,runtime 阶段仅保留 Python + 音频库 + `/opt/venv`
- [ ] Runtime 阶段**不含** `build-essential`、`git`、`pip` 缓存与源码包
- [ ] 关键版本通过 ARG 暴露:CUDA 版本 `CUDA_VERSION` / `UBUNTU_VERSION` / `TORCH_CUDA_TAG`;CPU 版本 `PYTHON_VERSION` / `DEBIAN_CODENAME`
- [ ] ARG **默认值**直接满足上游引擎要求,`docker build` 不传 `--build-arg` 即可得到可用镜像
- [ ] `docker/entrypoint.sh` 按 §5 模板,`chmod +x`
- [ ] `docker/requirements.api.txt` 包含 7 行基线(见 `project-layout-spec.md` §5.2)
- [ ] 镜像预设的环境变量符合 §8.1(CUDA 镜像不预设 `_DEVICE` / `_DTYPE`,CPU 镜像预设 `cpu` / `float32`)
- [ ] 按引擎能力暴露条件变量:多版本 → `{ENGINE}_VERSION`,多变体 → `{ENGINE}_VARIANT`,支持量化(仅 CUDA) → `{ENGINE}_QUANTIZATION`;命名固定,不使用别名
- [ ] 其他引擎特有变量全部以 `{ENGINE}_` 为前缀,全大写下划线命名
- [ ] `VOLUME ["/voices", "/root/.cache"]` 声明
- [ ] `EXPOSE 8000`
- [ ] `ENTRYPOINT ["/entrypoint.sh"]` + `CMD ["uvicorn", "app.server:app", ...]`
- [ ] `docker/docker-compose.example.yml` 只含**一个** service,与镜像变体对应
- [ ] `.dockerignore` 包含 `.git/`、`cache/`、`voices/`、`*.md` 等
- [ ] Dockerfile 内层顺序(builder 内):系统包 → torch → 上游 → API 依赖;(runtime 内):系统包 → 拷贝 venv → 应用代码
