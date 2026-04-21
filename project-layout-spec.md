# Open TTS 项目结构与代码规范

> 版本: v1.0 · 最后更新: 2026-04-21
>
> 本规范定义了一个 `*-open-tts` 项目的标准目录结构、`app/` 模块职责划分、配置管理和依赖规范。

---

## 1. 项目命名

项目仓库统一命名为:

```
{engine}-open-tts
```

其中 `{engine}` 为全小写,连字符或无连字符均可(如 `omnivoice`, `qwen3-tts`, `fishspeech`)。

---

## 2. 顶级目录布局

```
{engine}-open-tts/
├── app/                          # FastAPI 应用层(本规范重点)
├── docker/                       # Docker 相关
├── engine/                       # 上游引擎,作为 git submodule(目录名固定为 engine)
├── .github/
│   └── workflows/
│       └── build-image.yml
├── .gitmodules                   # 声明 submodule
├── .gitignore
├── .dockerignore
├── README.md                     # 英文文档(默认)
└── README.zh.md                  # 中文文档
```

**可选**:
- `docs/`, `scripts/`, `tests/` — 如有则在 `.dockerignore` 中排除,不进入镜像。

**禁止**:
- 在仓库根放置模型权重、缓存、音色文件(应通过挂载)。
- 在 `app/` 外放置可执行代码(`run.py`, `main.py` 等),统一入口为 `uvicorn app.server:app`。

---

## 3. `app/` 模块规范

### 3.1 标准文件清单

| 文件 | 职责 | 强制 |
|---|---|---|
| `__init__.py` | 包标识,通常为空 | 是 |
| `server.py` | FastAPI 应用、路由、lifespan | 是 |
| `config.py` | `pydantic_settings.BaseSettings` 配置 | 是 |
| `schemas.py` | 请求/响应 Pydantic 模型 | 是 |
| `engine.py` | TTS 引擎包装层 | 是 |
| `audio.py` | 音频编码工具 | 是 |
| `voices.py` | 声音目录扫描与缓存 | 是 |
| `concurrency.py` | 并发闸门(见 http-api-spec §3.6) | 是 |

**禁止**:
- 在 `app/` 外散落工具模块。如果需要更细的结构,使用子包(`app/engine/`, `app/routes/`)而非平级散文件。
- 单个文件超过 500 行时考虑拆分,但**不要过度拆分**:`server.py` 的路由应保持一处可读。

### 3.2 `server.py` 规范

职责:
1. 定义 `lifespan` 异步上下文管理器,在启动时加载设置、声音目录、引擎。
2. 创建 FastAPI `app` 实例,挂载到 `lifespan`。
3. 定义所有路由。简单项目(≤ 8 个端点)直接在本文件写;端点更多时拆分到 `app/routes/` 子包并在 `server.py` 中 `include_router`。
4. 定义 `_validate_text` / `_validate_format` 等本地辅助函数。

示例骨架:

```python
from __future__ import annotations

import asyncio
import logging
from contextlib import asynccontextmanager
from urllib.parse import quote

from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import FileResponse, Response

from .audio import CONTENT_TYPES, encode
from .config import get_settings
from .concurrency import ConcurrencyLimiter
from .engine import TTSEngine
from .schemas import Capabilities, HealthResponse, SpeechRequest, VoiceInfo, VoiceListResponse
from .voices import VoiceCatalog

log = logging.getLogger(__name__)


# 引擎能力常量:在引擎加载前就已知,由实现自己声明
CAPABILITIES = Capabilities(
    clone=True,
    streaming=False,
    design=True,
    languages=False,
    builtin_voices=False,
)


@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()
    logging.basicConfig(level=settings.log_level.upper())
    app.state.settings = settings
    app.state.catalog = VoiceCatalog(settings.voices_path)
    app.state.limiter = ConcurrencyLimiter(
        max_concurrency=settings.max_concurrency,
        max_queue_size=settings.max_queue_size,
        queue_timeout=settings.queue_timeout,
    )
    app.state.engine = None
    try:
        app.state.engine = TTSEngine(settings)
    except Exception:
        log.exception("failed to load %s model", settings.engine_name)
        raise
    yield


app = FastAPI(
    title="{Engine} Open TTS",
    version="1.0.0",
    lifespan=lifespan,
)


@app.get("/healthz", response_model=HealthResponse)
async def healthz(request: Request) -> HealthResponse:
    settings = request.app.state.settings
    limiter = request.app.state.limiter
    engine: TTSEngine | None = request.app.state.engine
    if engine is None:
        return HealthResponse(
            status="loading",
            model=settings.engine_model,
            sample_rate=0,
            capabilities=CAPABILITIES,
            concurrency=limiter.snapshot(),
        )
    return HealthResponse(
        status="ok",
        model=settings.engine_model,
        sample_rate=engine.sample_rate,
        capabilities=CAPABILITIES,
        device=engine.device,
        dtype=engine.dtype_str,
        concurrency=limiter.snapshot(),
    )


@app.post("/v1/audio/speech")
async def speech(req: SpeechRequest, request: Request):
    # 1. 入参校验先行:参数错误直接 422/404,不占用队列
    _validate_text(req.input, request.app.state.settings.max_input_chars)
    _validate_format(req.response_format)
    # ... voice 解析、file:// 与 capabilities 校验 ...

    # 2. 进入并发闸门:阻塞等待 / 队列满返回 503 / 超时返回 503
    async with request.app.state.limiter.acquire():
        samples = await request.app.state.engine.synthesize_clone(...)
        body, content_type = encode(samples, engine.sample_rate, req.response_format)
    return Response(content=body, media_type=content_type)
```

**并发闸门实现要点**(`app/concurrency.py`,单文件即可):

```python
import asyncio
from contextlib import asynccontextmanager

from fastapi import HTTPException


class ConcurrencyLimiter:
    def __init__(self, max_concurrency: int, max_queue_size: int, queue_timeout: float):
        self._sem = asyncio.Semaphore(max_concurrency)
        self._max = max_concurrency
        self._max_queue = max_queue_size       # 0 = 不限
        self._timeout = queue_timeout           # 0 = 不限
        self._queued = 0                        # 当前等待者数

    @asynccontextmanager
    async def acquire(self):
        if self._max_queue and self._queued >= self._max_queue:
            raise HTTPException(status_code=503, detail="engine busy: queue full")
        self._queued += 1
        try:
            coro = self._sem.acquire()
            if self._timeout:
                await asyncio.wait_for(coro, timeout=self._timeout)
            else:
                await coro
        except asyncio.TimeoutError:
            raise HTTPException(status_code=503, detail="engine busy: queue timeout")
        finally:
            self._queued -= 1
        try:
            yield
        finally:
            self._sem.release()

    def snapshot(self) -> dict:
        return {
            "max": self._max,
            "active": self._max - self._sem._value,   # 仅用于观测,允许读 _value
            "queued": self._queued,
        }
```

- `async with limiter.acquire():` 包裹所有**进入引擎**的代码,包括流式端点——流式情况下 `async with` 需要持续到流生成器彻底耗尽(见下文)。
- 客户端断连时 FastAPI 会抛 `CancelledError`,`finally` 分支释放令牌,不会泄漏。
- 流式端点(`/v1/audio/realtime`)不能直接用 `async with` 包裹 `StreamingResponse(...)` 的返回——那会在 handler return 时就释放。正确做法是让闸门覆盖生成器本身:

  ```python
  async def _gen():
      async with request.app.state.limiter.acquire():
          async for chunk in engine.synthesize_realtime(...):
              yield encoder.encode(chunk)
      # encoder flush
  return StreamingResponse(_gen(), media_type=CONTENT_TYPES[fmt])
  ```

**`capabilities` 注册原则**:

- `CAPABILITIES` 常量在模块顶层声明,值在**引擎加载前**就已知,由实现者手动填写。
- 不要在 `healthz` handler 里动态检查引擎方法存在性——这会让能力发现变得不可靠(慢、延迟绑定)。
- 按引擎能力,在 `app = FastAPI(...)` 之后通过 `if CAPABILITIES.clone: app.include_router(clone_router)` 模式**条件注册** `/v1/audio/clone` / `/v1/audio/design` / `/v1/audio/realtime` / `/v1/audio/languages`。未注册的路由由 FastAPI 自动 404。

### 3.3 `config.py` 规范

使用 `pydantic_settings.BaseSettings`,通过环境变量驱动。

**约束**:
1. 环境变量前缀 = 引擎名大写下划线(如 `OMNIVOICE_*`, `QWEN3_*`, `FISHSPEECH_*`)。**不**使用 `env_prefix`,而是将字段命名为 `omnivoice_model` 等小写形式,依赖 Pydantic 的 `case_sensitive=False` 自动映射。
2. 必须提供通用字段:`host`, `port`, `log_level`, `max_input_chars`, `default_response_format`, `voices_dir`, `max_concurrency`(以及可选的 `max_queue_size` / `queue_timeout`)。所有这些字段**不带**引擎前缀,是服务级共享约定。
3. 使用 `@lru_cache(maxsize=1)` 的 `get_settings()` 单例获取器。
4. 可用 `@property` 提供派生值(如 `voices_path: Path`, `resolved_device: str`, `torch_dtype`)。
5. `torch` 相关的 import 写在 property 内部,避免模块加载期间导入 torch。

模板:

```python
from __future__ import annotations

from functools import lru_cache
from pathlib import Path
from typing import Literal, Optional

from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="",
        case_sensitive=False,
        extra="ignore",
    )

    # 引擎模型配置
    engine_model: str = Field(default="org/model-id")
    engine_device: Literal["auto", "cuda", "cpu"] = Field(default="auto")
    engine_cuda_index: int = Field(default=0)
    engine_dtype: Literal["float16", "bfloat16", "float32"] = Field(default="float16")

    # 按引擎能力选择性暴露(对应 docker-spec.md §8.2 条件变量)
    # 多版本模型时使用(例:CosyVoice 的 v2 与 v3)
    # engine_version: Optional[str] = Field(default=None)
    # 多变体时使用(例:MOSS-TTS 的 tts / ttsd / voicegen / sfx)
    # engine_variant: Optional[str] = Field(default=None)
    # 支持量化时使用(仅 CUDA 镜像生效)
    # engine_quantization: Literal["none", "int8", "int4"] = Field(default="none")

    # 引擎推理参数(可选,给出合理默认值)
    engine_num_step: int = Field(default=32, ge=1, le=64)
    engine_guidance_scale: float = Field(default=2.0, ge=0.0, le=10.0)

    # 服务通用配置(不带引擎前缀)
    voices_dir: str = Field(default="/voices")
    host: str = Field(default="0.0.0.0")
    port: int = Field(default=8000)
    log_level: str = Field(default="info")
    max_input_chars: int = Field(default=8000)
    default_response_format: Literal[
        "mp3", "opus", "aac", "flac", "wav", "pcm"
    ] = Field(default="mp3")

    # 并发控制(见 http-api-spec §3.6)
    max_concurrency: int = Field(default=1, ge=1)
    max_queue_size: int = Field(default=0, ge=0)      # 0 = 不限
    queue_timeout: float = Field(default=0.0, ge=0.0)  # 0 = 不限

    @property
    def voices_path(self) -> Path:
        return Path(self.voices_dir)

    @property
    def resolved_device(self) -> str:
        import torch
        if self.engine_device == "auto":
            if torch.cuda.is_available():
                return f"cuda:{self.engine_cuda_index}"
            return "cpu"
        if self.engine_device == "cuda":
            return f"cuda:{self.engine_cuda_index}"
        return self.engine_device

    @property
    def torch_dtype(self):
        import torch
        return {
            "float16": torch.float16,
            "bfloat16": torch.bfloat16,
            "float32": torch.float32,
        }[self.engine_dtype]


@lru_cache(maxsize=1)
def get_settings() -> Settings:
    return Settings()
```

**实际项目中将 `engine_*` 替换为具体前缀**,如 `omnivoice_model`, `omnivoice_device`。

### 3.4 `schemas.py` 规范

定义所有 HTTP 请求/响应的 Pydantic 模型。必须导出:

| 名称 | 用途 | 对应 API |
|---|---|---|
| `ResponseFormat` | 6 种格式的 `Literal` 类型别名 | §3.1 通用 |
| `Capabilities` | `clone`/`streaming`/`design`/`languages`/`builtin_voices` 5 个 `bool` 字段 | §4.1 /healthz |
| `HealthResponse` | `status` / `model` / `sample_rate` / `capabilities` 必需,`device` / `dtype` / `concurrency` 可选 | §4.1 /healthz |
| `VoiceInfo` | `id` / `preview_url: str \| None` / `prompt_text: str \| None` / `metadata: dict \| None` | §4.2 /voices |
| `VoiceListResponse` | `{ voices: list[VoiceInfo] }`(**不要**沿用 OpenAI 风格的 `object`/`data` 包装) | §4.2 /voices |
| `SpeechRequest` | `model?` / `input` / `voice` / `response_format?` / `speed?` / `instructions?` | §4.4 /speech |
| `CloneRequest` | 通过 `Form(...)` + `UploadFile` 声明的 multipart 依赖类,而非 Pydantic 模型 | §5.1 /clone |
| `DesignRequest` | `input` / `instruct: str \| None` / `response_format?`(**不含** `model`/`speed`/`instructions`) | §5.2 /design |
| `RealtimeRequest` | 字段与 `SpeechRequest` 一致(可以直接复用) | §5.3 /realtime |
| `Language` / `LanguagesResponse` | `Language(key: str, name: str)` + `LanguagesResponse(languages: list[Language])` | §5.4 /languages |

引擎私有扩展端点不在本规范内;如果项目自行实现,对应 Request 模型也放在本文件,命名不与本表冲突即可。

**字段规则**:
1. 所有 `Field` 必须带 `description`(用于自动生成 OpenAPI 文档)。
2. 数值字段必须声明 `ge`/`le`。
3. OpenAI 兼容字段 `model` 必须是 `Optional[str]`,描述为 `"Accepted for OpenAI compatibility; ignored."`。
4. `response_format` 默认为 `"mp3"`。
5. 所有 Request 都要 `model_config = ConfigDict(extra="ignore")`,以便引擎扩展字段和未知 OpenAI 字段不会导致 422。
6. `SpeechRequest.instructions` 为 `Optional[str]`,默认 `None`;不支持的引擎在 handler 中忽略该字段,**不**报错。

模板与完整字段列表见 `http-api-spec.md` §4.1 / §4.2 / §4.4 / §5。

### 3.5 `audio.py` 规范

统一的音频编码器,必须导出:

- `CONTENT_TYPES: dict[str, str]` — 6 种格式到 MIME 类型的映射。
- `encode(samples: np.ndarray, sample_rate: int, fmt: str) -> tuple[bytes, str]` — 编码函数。

**实现要求**:
1. 先归一化为单声道 `float32 [-1, 1]`。
2. `wav`/`flac` 使用 SoundFile;`mp3`/`opus`/`aac` 使用 PyAV;`pcm` 手动写 `<i2` 小端字节。
3. PyAV 版本固定为 `av==12.3.0`(跨引擎统一)。

**不要**:在本模块内做变采样、去噪、响度归一化。这些由引擎层负责。

### 3.6 `voices.py` 规范

必须导出 `Voice` dataclass 和 `VoiceCatalog` 类。`Voice.id` 存储**不含 `file://` 前缀**的纯文件名,`file://` 前缀仅在 HTTP 边界添加/剥离:

```python
FILE_VOICE_PREFIX = "file://"


@dataclass(frozen=True)
class Voice:
    id: str                     # 纯文件名,不含 file:// 前缀
    wav_path: Path
    txt_path: Path
    yml_path: Path | None       # 可选 YAML 元信息路径
    prompt_text: str
    metadata: dict | None       # 解析后的 yml 内容,缺失/解析失败时为 None
    mtime: float

    @property
    def uri(self) -> str:
        """HTTP 层暴露的完整标识,用于 VoiceInfo.id 和 voice 字段。"""
        return f"{FILE_VOICE_PREFIX}{self.id}"


class VoiceCatalog:
    def __init__(self, root: Path): ...
    def scan(self) -> Dict[str, Voice]: ...
    def get(self, vid: str) -> Voice | None:
        """接受 'file://alice' 或裸 'alice',统一剥离前缀后按 id 查找。"""
        ...
```

**扫描规则**见 `http-api-spec.md` §6。要点:
- 扫描顶层的 `.wav` / `.txt` / `.yml`,以文件名(去扩展名)聚合为 `<id>`。
- `.wav` + `.txt` 缺一不可(有则 voice 成立);`.yml` 可选(无则 `metadata=None`)。
- YAML 用 `yaml.safe_load` 解析,顶层必须是 `dict`,否则记 warning 并当作缺失处理。
- 解析失败不让整个声音目录加载失败,只跳过该 `.yml`。
- `mtime` 取 `.wav` / `.txt` / `.yml` 中最新的,用于引擎层缓存失效。

**缓存策略**:`scan()` 可以每次实时扫描,也可以按 `mtime` 缓存。对于大声音库(>100 个)建议缓存;否则每次扫描更简单。

**HTTP 边界处理**:
- `/v1/audio/voices` 的响应体为 `{ "voices": [...] }`(见 http-api-spec §4.2),**不使用** `object`/`data` 包装。每个 `VoiceInfo.id` 使用 `voice.uri`(带 `file://`),`preview_url` 与 `prompt_text` 对内置声音返回 `null`。
- `/v1/audio/voices/preview?id=<plain>` 的 `id` 查询参数使用**去掉 `file://` 前缀**的纯文件名。handler 拿到 `id` 后直接拼 `${VOICES_DIR}/<id>.wav` 返回。若引擎只提供内置声音,不注册该路由(或直接 404)。
- `/v1/audio/speech` 的 handler 在分发前判断:
  - `voice.startswith("file://")` → 走克隆路径,`catalog.get(voice)` 加载文件。
  - 否则 → 走内置声音路径(由引擎解释)。
- 引擎能力与 `voice` 形态必须匹配(见 http-api-spec §4.4 规则 5/6):
  - `capabilities.clone=false` 但收到 `file://...` → 422。
  - `capabilities.builtin_voices=false` 但收到普通名 → 422。
- 仅支持内置声音的引擎(如 Kokoro)可以不提供 `VoiceCatalog`,或让 `VoiceCatalog.scan()` 遍历引擎内置声音列表并返回不含 `file://` 前缀的 `VoiceInfo`(此时 `preview_url`/`prompt_text` 为 `null`)。

### 3.7 `engine.py` 规范

TTS 引擎的薄包装,不定义抽象基类(过度工程),但所有实现需暴露一致的属性与方法签名:

**必需属性**:
- `device: str` — 实际运行设备。
- `dtype_str: str` — dtype 字符串(`"float16"` 等)。
- `sample_rate: int` — 推理输出采样率。

**规范端点对应的方法**(按 `CAPABILITIES` 选择性实现):

```python
# 对应 POST /v1/audio/speech + voice="file://..."(capabilities.clone)
async def synthesize_clone(
    self,
    text: str,
    *,
    ref_audio: str,          # 本地文件路径,可以是 VOICES_DIR 下的文件或 /tmp 中的上传
    ref_text: str,           # 参考文本;支持文本无关模式时可为空串
    ref_mtime: float | None = None,  # 用于 prompt embedding 缓存失效;临时文件传 None
    instructions: str | None = None, # OpenAI 的 instructions,引擎可忽略
    speed: float = 1.0,
    **engine_specific,
) -> np.ndarray: ...

# 对应 POST /v1/audio/speech + voice=<builtin>(capabilities.builtin_voices)
async def synthesize_builtin(
    self,
    text: str,
    *,
    voice: str,              # 内置声音名,不含 file:// 前缀
    instructions: str | None = None,
    speed: float = 1.0,
    **engine_specific,
) -> np.ndarray: ...

# 对应 POST /v1/audio/design(capabilities.design)
async def synthesize_design(
    self,
    text: str,
    *,
    instruct: str | None,    # null/空串表示使用引擎内部默认声音
    **engine_specific,
) -> np.ndarray: ...

# 对应 POST /v1/audio/realtime(capabilities.streaming)
async def synthesize_realtime(
    self,
    text: str,
    *,
    voice: str,              # 形态同 speech(file://... 或内置名)
    response_format: str,    # 引擎用于选择流式编码器
    speed: float = 1.0,
    instructions: str | None = None,
    **engine_specific,
) -> AsyncIterator[np.ndarray]: ...

# 对应 GET /v1/audio/languages(capabilities.languages)
def list_languages(self) -> list[tuple[str, str]]:
    """返回 [(key, name), ...],key 按引擎原生形式保留(不做规范化)。"""
    ...
```

**`POST /v1/audio/clone`**(multipart 一次性上传克隆)复用 `synthesize_clone`:handler 把上传文件落到 `/tmp/<uuid>.<ext>`,调用 `synthesize_clone(ref_audio=tmp_path, ref_text=form.prompt_text, ref_mtime=None, ...)`,完成后删除。引擎不需要知道是文件目录还是临时文件。

引擎私有扩展(`synthesize_emotion`、`synthesize_dialogue`、`synthesize_effects`、`synthesize_direct` 等)不在本规范内,项目可按需实现,命名风格与上一致即可。

**实现要求**:
1. 非流式方法返回 `np.ndarray` 单声道,`float32`,`[-1, 1]`。
2. 流式方法返回 `AsyncIterator[np.ndarray]`,每个 chunk 同样单声道 `float32`;增量编码由 server 层的 `StreamingResponse` 配合 PyAV 增量 encoder 完成。
3. 阻塞推理必须在 `asyncio.to_thread` 或 `run_in_executor` 中执行,避免阻塞事件循环;流式方法用 `asyncio.Queue` + 后台 thread 桥接同步 generator。
4. 模型加载放在 `__init__`,同步执行;加载失败让 lifespan 异常终止而非静默 fallback。
5. 引擎特定的缓存(如 prompt embeddings)以 LRU 形式实现,以 `ref_audio` 路径 + `ref_mtime` 为 key;临时文件 `ref_mtime=None` 时跳过缓存。
6. 引擎方法签名中**不使用** `language` 作为规范字段——语言是引擎私有扩展,通过 `**engine_specific` 传递;仅当 `capabilities.languages=true` 时客户端才会知道该字段的取值。

**模型权重自动下载**:

7. `__init__` 首次加载模型时,**必须**自动从 HuggingFace Hub / ModelScope 拉取权重到 `HF_HOME`(默认 `/root/.cache/huggingface`)。优先走各 SDK 原生的"缓存未命中则下载"路径(`from_pretrained(...)`、`snapshot_download(...)` 等),**不要**自己实现 HTTP 下载逻辑。
8. 首次启动时下载可能耗时数分钟到数十分钟,此期间 `/healthz` 返回 `status="loading"`(模型未就绪,但服务端口已在监听)。下载完成后 `status` 转为 `"ok"`。
9. 下载失败(网络中断、权重私有未认证、盘满等)必须抛异常让 lifespan 终止并退出容器(而非静默回退到假加载)。Kubernetes / systemd 会自动重启容器,下次尝试会从已下载的分片继续(SDK 原生支持断点续传)。
10. **持久化**:`/root/.cache` 是 Docker `VOLUME`(见 `docker-spec.md` §9),用户用 `-v ./cache:/root/.cache` 挂载后重启容器不会重复下载。
11. **离线部署**:用户设置 `HF_HUB_OFFLINE=1` 时,SDK 只读缓存不触发网络请求;若缓存缺失则直接抛错。引擎**不需要**为此写特殊代码,走标准 SDK 行为即可。
12. **不要**在 Dockerfile 中预下载权重:这会使镜像膨胀(单权重常见 2-10 GB)且与用户按需升级模型解耦。权重管理交给运行时的卷挂载。

---

## 4. 上游引擎集成

上游引擎作为 **git submodule** 集成,目录和 submodule name **固定为 `engine`**:

```ini
# .gitmodules
[submodule "engine"]
    path = engine
    url = https://github.com/<upstream-owner>/<upstream-repo>.git
```

**约束**:
1. submodule 路径固定为 `engine`,submodule name 同样为 `engine`。不再使用上游仓库的原大小写名称——跨项目统一的目录名让 Dockerfile、CI 步骤、`PYTHONPATH` 等可直接复用模板。
2. pin 到具体 commit,不跟 branch。版本升级通过 `git submodule update --remote` + 显式 commit。
3. 在 Dockerfile 中通过 `COPY engine /opt/api/engine` + `pip install /opt/api/engine` 安装。如果上游没有 `pyproject.toml`/`setup.py`,则通过 `PYTHONPATH=/opt/api/engine` 暴露(若还有嵌套 third_party 依赖,一并加入,例如 `PYTHONPATH=/opt/api/engine:/opt/api/engine/third_party/Matcha-TTS`)。
4. 上游仓库**不**打 patch。如果需要修改上游行为,在 `app/` 下写 monkey patch 模块(如 fishspeech 的 `_max_seq_len_patch.py`),并在 `engine.py` 的 `__init__` 头部 import。

---

## 5. 依赖管理

### 5.1 依赖文件位置

```
docker/
├── requirements.api.txt          # FastAPI 应用层依赖(强制)
└── requirements.engine.txt     # 引擎特定额外依赖(可选)
```

**禁止**:
- 在仓库根放 `requirements.txt`(会与上游 submodule 的文件混淆)。
- 在 `app/` 下放依赖文件。

### 5.2 `requirements.api.txt` 基线

所有项目统一使用:

```
fastapi>=0.115
uvicorn[standard]>=0.30
pydantic>=2.8
pydantic-settings>=2.5
soundfile>=0.12
av==12.3.0
pyyaml>=6.0
```

**版本策略**:
- FastAPI/Pydantic 族:`>=` 松约束,跟随上游 bug 修复。
- `av`:`==` 严约束,跨项目固定 12.3.0,避免编码器回归。
- 其他工具库:可选加入,遵循"需要就加,能宽松就宽松"。

### 5.3 PyTorch 安装

**独立于 `requirements.*.txt`**,在 Dockerfile 中用官方索引安装:

```dockerfile
# CPU
RUN pip install --extra-index-url https://download.pytorch.org/whl/cpu \
        torch==X.Y.Z+cpu torchaudio==X.Y.Z+cpu

# CUDA (示例为 CUDA 12.8)
RUN pip install --index-url https://download.pytorch.org/whl/cu128 \
        torch==X.Y.Z torchaudio==X.Y.Z
```

版本选择:优先匹配上游引擎对 torch 的约束;CUDA 版本与 `nvidia/cuda` 基础镜像对齐。

### 5.4 依赖安装顺序(Dockerfile)

正确顺序对构建缓存至关重要:

1. 系统包(`apt-get install`)
2. `pip install torch` — 单独一层,体积大,变动少
3. `COPY engine /opt/api/engine` + `pip install /opt/api/engine`(或通过 `PYTHONPATH=/opt/api/engine` 暴露) — 上游引擎层
4. `COPY docker/requirements.api.txt` + `pip install -r` — API 层依赖
5. `COPY app /opt/api/app` — 应用代码(变动最频繁,放最后)

这样,应用代码修改只触发最后一层重建。

---

## 6. `.gitignore` 与 `.dockerignore` 规范

### 6.1 `.gitignore`

除了语言标准条目,至少忽略:

```
__pycache__/
*.pyc
.venv/
venv/
.pytest_cache/
.ruff_cache/
.mypy_cache/
.idea/
.vscode/
.DS_Store

cache/
voices/
*.log
```

**不要**忽略 `.github/`, `.gitmodules`, `docker/`, `app/`。

### 6.2 `.dockerignore`

镜像构建只需要 `app/`、`docker/` 和 submodule 目录,其他一律忽略:

```
.git/
.gitignore
.github/
.venv/
venv/
__pycache__/
*.pyc
.pytest_cache/
.ruff_cache/
.mypy_cache/
.idea/
.vscode/
.DS_Store

cache/
voices/
tests/
docs/
scripts/

README.md
README.zh.md
*.md
*.log
```

忽略 `.git/` 很关键:submodule 的 `.git` 目录会显著膨胀构建上下文。

---

## 7. README 规范

### 7.1 结构

两份 README:`README.md` (英文,默认) + `README.zh.md` (中文)。内容等价。

推荐章节:

```
# {Engine} Open TTS

One-paragraph description

## Features
## Quick Start (Docker)
## Configuration (env vars table)
## API Reference
  ### POST /v1/audio/speech
  ### POST /v1/audio/design  (if supported)
  ...
## Voice Directory
## Development
```

### 7.2 必须包含的内容

1. GHCR 镜像 pull 命令(单镜像)。
2. 完整的 `docker run` 示例,含卷挂载和 GPU 声明。
3. 配置环境变量表(变量名、默认值、说明)。
4. API 端点表(方法、路径、简述)。
5. 指向规范文档的链接(`https://github.com/seancheung/open-tts-spec`)。

---

## 8. 代码风格

1. **Python 3.10+**。使用 `from __future__ import annotations` 以启用 PEP 604 `X | Y` 语法。
2. **类型注解**:公开函数必须有完整注解;内部辅助可省略。
3. **异步**:所有 HTTP handler 使用 `async def`;阻塞调用用 `asyncio.to_thread` 包装。
4. **日志**:`logging.getLogger(__name__)`,不 `print`。异常处理必须 `log.exception(...)` 记录堆栈。
5. **不写文档字符串堆山**:一个清晰的类型签名 + 1 行 description 胜过三段 docstring。
6. **不引入 formatter 锁定**:不强制 black/ruff 配置,项目自行约定。

---

## 9. 测试建议

规范**不强制**单元测试。如果编写:

- 测试目录:`tests/`,不进入镜像(在 `.dockerignore`)。
- 不在 CI 中跑需要模型权重的测试(会使 CI 超时)。
- 音频编解码可以用 1 秒的合成波形做 smoke test。
- 端点测试用 FastAPI `TestClient`,mock 引擎层。

---

## 10. 检查清单

新项目初始化后按此清单自检:

**核心必须**:

- [ ] 目录结构符合 §2,顶层命名为 `{engine}-open-tts/`
- [ ] `app/` 下 7 个标准文件齐全(§3.1)
- [ ] `Settings` 有 `host`/`port`/`log_level`/`max_input_chars`/`default_response_format`/`voices_dir`/`max_concurrency`(默认 1);`max_queue_size`/`queue_timeout` 可选
- [ ] `/healthz` 返回 `status`/`model`/`sample_rate`/`capabilities` 全部必需字段,`capabilities` 5 个 bool 齐全
- [ ] `capabilities` 声明与实际暴露的路由一致(`true` 则路由注册,`false` 则不注册)
- [ ] 至少 `capabilities.clone` 或 `capabilities.builtin_voices` 之一为 `true`
- [ ] `/v1/audio/voices` 响应为 `{ "voices": [...] }`
- [ ] `VoiceInfo`:`file://` 前缀对克隆声音,内置声音无前缀;内置声音的 `preview_url`/`prompt_text` 为 `null`
- [ ] `VoiceInfo.metadata` 字段已实现:文件克隆声音解析可选的 `${VOICES_DIR}/<id>.yml`(YAML 对象),内置声音由引擎填充或返回 `null`
- [ ] `voices.py` 使用 `yaml.safe_load` 解析;解析失败不影响声音本身加载
- [ ] `/v1/audio/voices/preview` 的 `id` 查询参数使用**去掉 `file://` 前缀**的纯文件名,仅服务文件克隆声音
- [ ] `/v1/audio/speech` 接受 `input`/`voice`/`response_format`/`speed`/`model`/`instructions`,`instructions` 不支持时忽略不报错
- [ ] `voice` 形态与 `capabilities` 不匹配时返回 422(见 http-api-spec §4.4 规则 5/6)
- [ ] 6 种音频格式(`mp3`/`opus`/`aac`/`flac`/`wav`/`pcm`)全部返回正确的 MIME 类型
- [ ] 所有推理类端点(`/v1/audio/speech`、`/clone`、`/design`、`/realtime`)通过 `ConcurrencyLimiter` 持有并发令牌;`/healthz`、`/v1/audio/voices*`、`/v1/audio/languages` **不**持有(见 http-api-spec §3.6)
- [ ] 参数校验在获取令牌**之前**完成,避免错误请求占队
- [ ] 流式端点在 `StreamingResponse` 的生成器内持有令牌,流结束或连接断开时释放

**可选扩展(按 capabilities)**:

- [ ] `capabilities.clone=true` 时,`POST /v1/audio/clone` 以 `multipart/form-data` 接受 `audio`/`prompt_text`/`input`(+ `response_format`/`speed`/`instructions`/`model`),上传音频不持久化到 `${VOICES_DIR}`
- [ ] `capabilities.design=true` 时,`POST /v1/audio/design` 接受 `input`/`instruct?`/`response_format?`,`instruct` 为 `null` 或空串不报错
- [ ] `capabilities.streaming=true` 时,`POST /v1/audio/realtime` 使用 `Transfer-Encoding: chunked`,至少支持 `mp3` 和 `pcm` 流式
- [ ] `capabilities.languages=true` 时,`GET /v1/audio/languages` 返回 `{ "languages": [{ "key", "name" }] }`

**工程规范**:

- [ ] 上游引擎作为 submodule 集成,目录和 submodule name 固定为 `engine`
- [ ] `.gitignore` 与 `.dockerignore` 按规范
- [ ] `docker/requirements.api.txt` 使用基线版本
- [ ] README (中英文) 含配置表和 API 表
