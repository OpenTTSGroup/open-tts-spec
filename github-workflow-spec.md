# Open TTS GitHub Actions 工作流规范

> 版本: v1.0 · 最后更新: 2026-04-21
>
> 本规范定义了 `*-open-tts` 项目的 CI/CD 流水线标准,覆盖构建触发、镜像命名、标签策略、缓存方案与权限配置。与 `docker-spec.md` 的单一 Dockerfile、单一镜像变体策略保持一致:每个项目只产出一个镜像(要么 CUDA 要么 CPU,不同时产出),工作流**不使用** variant 矩阵。

---

## 1. 工作流文件

每个项目在 `.github/workflows/` 下**必须**包含:

```
.github/workflows/
└── build-image.yml            # 构建单一 Docker 镜像并推送至 GHCR
```

可选工作流:

```
.github/workflows/
├── lint.yml                   # 代码风格检查(可选)
├── smoke-test.yml             # 镜像冒烟测试(可选)
└── release.yml                # GitHub Release 自动化(可选)
```

---

## 2. `build-image.yml` 完整模板

```yaml
name: build-image

on:
  push:
    branches: [main]
    tags: ['v*']
    paths-ignore:
      - '**.md'
      - '.gitignore'
      - '.dockerignore'
      - 'LICENSE'
  workflow_dispatch:

permissions:
  contents: read
  packages: write

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout (with submodules)
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 1

      - name: Ensure submodules are populated
        run: |
          git submodule sync --recursive
          git submodule update --init --recursive --force
          # 可选:验证 submodule 核心文件存在(防止 submodule 残缺)
          # ls {UpstreamModule}/pyproject.toml

      - name: Free disk space
        run: |
          sudo rm -rf /usr/share/dotnet /usr/local/lib/android /opt/ghc \
            /opt/hostedtoolcache/CodeQL /usr/local/share/boost
          sudo docker image prune -a -f || true
          df -h

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Compute image name (lowercase)
        id: imgname
        run: |
          repo="${GITHUB_REPOSITORY,,}"
          echo "image=ghcr.io/${repo}" >> "$GITHUB_OUTPUT"

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ steps.imgname.outputs.image }}
          tags: |
            type=raw,value=latest,enable={{is_default_branch}}
            type=sha,format=short
            type=semver,pattern={{version}}

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          file: docker/Dockerfile
          platforms: linux/amd64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          # 如需在 CI 中覆盖 Dockerfile 的 ARG 默认值,取消注释并填入:
          # build-args: |
          #   CUDA_VERSION=12.4.1
          #   TORCH_CUDA_TAG=cu124
          #   UBUNTU_VERSION=22.04
```

---

## 3. 触发条件

### 3.1 标准触发集

| 触发 | 原因 |
|---|---|
| `push` 到 `main` | 发布 `latest` 流动标签 |
| `push` 的 `v*` tag | 发布语义化版本标签(`v1.2.3` → `v1.2.3`) |
| `workflow_dispatch` | 手动触发,常用于调试或传 `build-args` 生成变体镜像 |

**禁止**加入的触发:
- `pull_request` — 会消耗大量 CI 时间(单次 CUDA 构建可 30+ 分钟),且不推送的构建意义有限。如需 PR 验证,使用独立的 lint/test workflow。
- `schedule` — 除非有明确理由(如每周 rebuild 拉取上游安全更新),否则不使用。

### 3.2 `paths-ignore`

以下路径变更**不**触发构建(节省 CI 资源):

```yaml
paths-ignore:
  - '**.md'
  - '.gitignore'
  - '.dockerignore'
  - 'LICENSE'
```

**注意**:`.dockerignore` 虽然影响构建上下文,但通常是一次性调整,不值得每次触发 30 分钟构建。手动 dispatch 即可。

---

## 4. 权限声明

```yaml
permissions:
  contents: read
  packages: write
```

**最小权限原则**:
- `contents: read` — checkout 代码。
- `packages: write` — 推送 GHCR 镜像。
- **不**声明 `id-token: write`(除非配置 OIDC 签名)。
- **不**声明 `pull-requests: write`(本 workflow 不评论 PR)。

GHCR 推送使用 `${{ secrets.GITHUB_TOKEN }}`,无需额外配置 PAT。镜像归属由 `GITHUB_REPOSITORY` 自动推导。

---

## 5. 镜像命名与标签策略

### 5.1 命名规则

镜像完整路径:

```
ghcr.io/{owner_lowercase}/{repo_lowercase}
```

GitHub 的 `REPOSITORY` 变量通常是 `Owner/Repo` 大小写混合,GHCR 要求小写。通过:

```bash
repo="${GITHUB_REPOSITORY,,}"
```

的 bash 小写展开完成转换。

### 5.2 标签清单

单镜像策略下,所有标签均**无前缀**:

| 触发 | 标签示例 |
|---|---|
| main 分支推送 | `latest` |
| 任意 commit push | `<sha-7>`(如 `abc1234`) |
| git tag `vX.Y.Z` | `vX.Y.Z`(如 `v1.2.3`) |

**设计说明**:
- 单镜像不需要区分 CPU / CUDA,`latest` 指向当前 main 分支的唯一镜像。
- 每次 commit 都有独立 `<sha-7>` tag,方便 rollback。
- 发布版本走 git tag `vX.Y.Z`,由 `metadata-action` 自动转成镜像标签。
- `ghcr.io/{owner}/{repo}` 直接 pull 即可,无需关心变体。

### 5.3 `metadata-action` 配置

```yaml
- name: Docker meta
  id: meta
  uses: docker/metadata-action@v5
  with:
    images: ${{ steps.imgname.outputs.image }}
    tags: |
      type=raw,value=latest,enable={{is_default_branch}}
      type=sha,format=short
      type=semver,pattern={{version}}
```

关键点:
- `type=raw,value=latest,enable={{is_default_branch}}` — `latest` 只在默认分支更新,feature branch push 不会覆盖。
- `type=sha,format=short` — 生成 7 位 SHA 短标签。
- `type=semver,pattern={{version}}` — 从 git tag 自动推导语义版本(如 tag `v1.2.3` → 镜像标签 `1.2.3`,`metadata-action` 会自动剥离 `v` 前缀)。

---

## 6. 构建缓存策略

### 6.1 默认:GitHub Actions 缓存

```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```

**原理**:
- `type=gha` — 使用 GitHub Actions Cache 后端。
- `mode=max` — 缓存所有中间层(不仅最终层),最大化命中率。
- 单镜像策略下不需要 `scope=${variant}` 隔离,单一 scope 即可。

**限制**:GHA Cache 配额为 10GB/repo。CPU 镜像通常 3-5 GB、CUDA 镜像通常 6-9 GB,单镜像策略下一般在配额内;multi-stage 的 builder 层容易因 torch + 模型依赖而超过 8 GB,此时改用 registry 缓存。

### 6.2 超限方案:Registry 缓存

当镜像构建缓存 > 8 GB 或配额压力大时改用 registry 缓存:

```yaml
cache-from: type=registry,ref=${{ steps.imgname.outputs.image }}:buildcache
cache-to: type=registry,ref=${{ steps.imgname.outputs.image }}:buildcache,mode=max
```

**原理**:将缓存存为 GHCR 同仓库的隐藏 tag `buildcache`,使用仓库存储空间而非 Actions Cache 配额。

**代价**:GHCR 会显示 `buildcache` 这个 tag(可在 package settings 中隐藏),推送/拉取缓存消耗网络带宽(对大型 CUDA 镜像明显)。

**何时启用**:构建日志中出现 GHA Cache 命中率下降或 `cache-to` 因配额限制截断时立即启用。

---

## 7. Submodule 处理

### 7.1 Checkout 配置

```yaml
- uses: actions/checkout@v4
  with:
    submodules: recursive    # 递归拉取所有嵌套 submodule
    fetch-depth: 1           # 只拉最新提交,节省时间
```

### 7.2 显式验证步骤

checkout 的 `submodules: recursive` 有时在 submodule 为空或未初始化时静默跳过。显式验证:

```yaml
- name: Ensure submodules are populated
  run: |
    git submodule sync --recursive
    git submodule update --init --recursive --force
    # 强烈推荐:验证核心文件存在,发现残缺时立刻失败而非在 docker build 阶段
    ls {UpstreamModule}/pyproject.toml
```

将 `{UpstreamModule}` 替换为实际 submodule 目录名。`ls` 一个关键文件即可,不需要遍历全部。

---

## 8. 磁盘清理

GitHub runner 默认磁盘 ~14GB 可用,CUDA 镜像构建可能耗尽:

```yaml
- name: Free disk space
  run: |
    sudo rm -rf /usr/share/dotnet /usr/local/lib/android /opt/ghc \
      /opt/hostedtoolcache/CodeQL /usr/local/share/boost
    sudo docker image prune -a -f || true
    df -h
```

清理项:
- `/usr/share/dotnet` — .NET SDK(~2GB)
- `/usr/local/lib/android` — Android SDK(~10GB)
- `/opt/ghc` — Haskell toolchain
- `/opt/hostedtoolcache/CodeQL` — CodeQL 分析器(~5GB)
- `/usr/local/share/boost` — Boost C++ 库

共可释放 15-20GB。CUDA 项目(绝大多数)必须执行;CPU 项目可选,但保留此步骤不会带来副作用。

---

## 9. Job 策略

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
```

**关键决定**:
- `runs-on: ubuntu-latest` — 使用 Ubuntu latest,自动升级。如果需要锁定版本,使用 `ubuntu-22.04`。
- **不使用 matrix**:单镜像策略下只有一个 job,不需要 `strategy.matrix.variant`,也不需要 `fail-fast`。
- **不使用 `concurrency`**(默认):允许多个 push 触发的构建并行,通过 Docker buildx 缓存复用。如需节省资源可按 §11.1 启用。
- **不使用 `timeout-minutes`**:GitHub 默认 360 分钟足够,除非想更早失败。

---

## 10. 多架构构建(可选)

默认只构建 `linux/amd64`。

**是否可启用 ARM64 取决于项目选用的基础镜像**:

| 项目类型 | 基础镜像 | ARM64 可用? |
|---|---|---|
| CUDA 项目(绝大多数) | `nvidia/cuda:*-ubuntu*` | ✗ 不可,`nvidia/cuda` 无 ARM 发行 |
| CPU 项目(少数) | `python:*-slim-*` | ✓ 可 |

CPU 项目启用 ARM64:

```yaml
- name: Set up QEMU
  uses: docker/setup-qemu-action@v3

- name: Build and push
  uses: docker/build-push-action@v6
  with:
    platforms: linux/amd64,linux/arm64
    ...
```

**代价**:ARM64 构建通过 QEMU 仿真,比原生 amd64 慢 3-5 倍。除非有明确需求(Apple Silicon 生产部署、AWS Graviton),否则先不启用。

---

## 11. 检查清单

- [ ] `.github/workflows/build-image.yml` 按 §2 完整模板
- [ ] `permissions` 只声明 `contents: read` + `packages: write`
- [ ] 触发条件含 `push main`、`push tag v*`、`workflow_dispatch`
- [ ] `paths-ignore` 排除 `**.md`、`.gitignore`、`.dockerignore`、`LICENSE`
- [ ] 单一 build job,无 `strategy.matrix`
- [ ] Checkout 使用 `submodules: recursive` + `fetch-depth: 1`
- [ ] 有显式 submodule 验证步骤
- [ ] 有 `Free disk space` 步骤
- [ ] 登录 `ghcr.io`,用户为 `github.actor`,密码为 `GITHUB_TOKEN`
- [ ] 镜像名通过 `${GITHUB_REPOSITORY,,}` 小写化
- [ ] 标签 `latest` / `<sha-7>` / `vX.Y.Z`,无前缀
- [ ] `file: docker/Dockerfile`
- [ ] 缓存 `type=gha,mode=max`;超配额时改 registry 缓存(§6.2)
- [ ] `platforms: linux/amd64`(CUDA 项目只能 amd64;CPU 项目可选 arm64)
- [ ] 需要覆盖 `CUDA_VERSION` / `TORCH_CUDA_TAG` / `PYTHON_VERSION` 等时在 `build-args` 中传

---

## 12. 版本升级建议

定期审视以下 Action 版本更新(按 semver 主版本升级):

| Action | 当前稳定主版本 |
|---|---|
| `actions/checkout` | v4 |
| `docker/setup-buildx-action` | v3 |
| `docker/login-action` | v3 |
| `docker/metadata-action` | v5 |
| `docker/build-push-action` | v6 |
| `docker/setup-qemu-action` | v3 |

升级策略:在非高峰期(非发版窗口)统一 bump,观察 1 周无回归后合入所有 TTS 项目。
