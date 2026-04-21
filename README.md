# Open TTS Spec

面向 TTS 引擎的**统一 HTTP API + 项目结构 + Docker + CI/CD** 规范。

---

## 规范文档

| 文档 | 覆盖范围 |
|---|---|
| [`http-api-spec.md`](./http-api-spec.md) | HTTP 接口:语音合成、声音克隆、声音设计、流式合成、声音列表、健康检查 |
| [`project-layout-spec.md`](./project-layout-spec.md) | 仓库目录布局、`app/` 模块职责、配置管理、依赖规范 |
| [`docker-spec.md`](./docker-spec.md) | 单镜像 multi-stage Dockerfile 模板、基础镜像、entrypoint、卷挂载、Compose |
| [`github-workflow-spec.md`](./github-workflow-spec.md) | GitHub Actions CI/CD、镜像命名、标签策略、缓存方案 |

四份文档**互相独立**,可单独阅读;但在实际落地新项目时建议按 `project-layout → http-api → docker → github-workflow` 的顺序实现。

---

## 设计原则

1. **OpenAI 兼容**:核心端点 `POST /v1/audio/speech` 与 OpenAI TTS API 字段一致,任何 OpenAI SDK 可开箱即用。
2. **最小核心 + 可选扩展**:只有 4 个端点(`/healthz`、`/v1/audio/voices`、`/v1/audio/voices/preview`、`/v1/audio/speech`)是强制的;`/v1/audio/clone`、`/v1/audio/design`、`/v1/audio/realtime`、`/v1/audio/languages` 按引擎能力可选暴露,通过 `/healthz.capabilities` 声明。
3. **文件系统驱动的声音库**:声音以 `{id}.wav + {id}.txt` 文件对存在于挂载目录,无需数据库、无需管理接口。声音标识通过 `file://` 前缀区分克隆源与内置声音。
4. **引擎无关**:规范层不假设任何具体 TTS 引擎,扩展参数以 `Optional` 字段形式传递。
5. **单镜像发布**:每个项目只产出一个镜像(CUDA 或 CPU 二选一),发布到 GHCR,标签 `latest` / `<sha-7>` / `vX.Y.Z`,无前缀。

---

## 核心端点一览

| 方法 | 路径 | 强制 | 说明 |
|---|---|---|---|
| GET | `/healthz` | ✓ | 服务与模型状态 |
| GET | `/v1/audio/voices` | ✓ | 列出可用声音 |
| GET | `/v1/audio/voices/preview?id=` | ✓ | 声音样本播放 |
| POST | `/v1/audio/speech` | ✓ | 核心 TTS(OpenAI 兼容) |
| POST | `/v1/audio/clone` | ○ | 一次性上传克隆(multipart,仅克隆引擎) |
| POST | `/v1/audio/design` | ○ | 声音设计(自然语言描述) |
| POST | `/v1/audio/realtime` | ○ | 流式合成(chunked,仅流式引擎) |
| GET | `/v1/audio/languages` | ○ | 支持的语言列表 |

✓ = 强制,○ = 可选(按引擎能力)。

---

## 新项目快速起步

1. 阅读 [`project-layout-spec.md`](./project-layout-spec.md) §2-§3,创建仓库骨架:

   ```
   app/__init__.py
   app/server.py
   app/config.py
   app/schemas.py
   app/engine.py
   app/audio.py
   app/voices.py
   docker/Dockerfile
   docker/entrypoint.sh
   docker/requirements.api.txt
   docker/requirements.engine.txt
   docker/docker-compose.example.yml
   .github/workflows/build-image.yml
   .gitmodules
   .gitignore
   .dockerignore
   README.md
   README.zh.md
   ```

2. 将上游 TTS 引擎作为 `git submodule add` 到仓库根。

3. 按 [`http-api-spec.md`](./http-api-spec.md) §4 实现 4 个核心端点,根据引擎能力在 `/healthz.capabilities` 中声明扩展能力并注册对应路由。

4. 按 [`docker-spec.md`](./docker-spec.md) §3(CUDA,默认)或 §4(CPU,仅上游不支持 CUDA 时)填写单一 Dockerfile,使用 multi-stage 隔离构建与运行时。

5. 复制 [`github-workflow-spec.md`](./github-workflow-spec.md) §2 的完整模板作为 `build-image.yml`。

6. 按各规范的"检查清单"逐项核对。