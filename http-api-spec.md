# Open TTS HTTP API 规范

> 版本: v1.0 · 最后更新: 2026-04-21
>
> 本规范定义了一套统一的、OpenAI 兼容的 TTS HTTP API,覆盖语音合成、声音克隆、声音设计、声音列表、健康检查等能力。

---

## 1. 设计目标

1. **OpenAI 兼容**:`POST /v1/audio/speech` 的核心字段必须与 OpenAI TTS API 兼容,使得任何 OpenAI SDK / Chat 客户端 / LangChain 集成可开箱即用。
2. **最小核心 + 可选扩展**:核心接口覆盖 "文本 + 声音 → 音频" 的主路径;声音克隆(multipart 上传)、声音设计、流式合成、语言列表等能力以可选扩展端点形式存在,通过 `/healthz.capabilities` 声明。
3. **文件系统驱动的声音管理**:声音以 `{id}.wav + {id}.txt` 的文件对形式存在于挂载目录,无需数据库。
4. **引擎无关**:规范不假设任何特定 TTS 引擎,所有引擎特定参数通过可选字段传递。
5. **可演进**:新的引擎能力可以以新增可选字段或可选端点的方式加入,不破坏现有客户端。

---

## 2. URL 前缀与版本

所有业务接口位于 `/v1/audio/*` 路径下,健康检查位于 `/healthz`。

规范当前版本为 v1。未来破坏性变更将使用 `/v2/audio/*`,并保留 `/v1` 至少一个大版本周期。

---

## 3. 通用约定

### 3.1 内容类型

| `response_format` | Content-Type | 编码 |
|---|---|---|
| `mp3` (默认) | `audio/mpeg` | libmp3lame (PyAV) |
| `opus` | `audio/ogg` | libopus (PyAV),OGG 容器 |
| `aac` | `audio/aac` | aac (PyAV),ADTS 容器 |
| `flac` | `audio/flac` | FLAC (SoundFile) |
| `wav` | `audio/wav` | PCM_16 WAV (SoundFile) |
| `pcm` | `application/octet-stream` | 原始小端 int16,采样率同 `/healthz.sample_rate` |

**所有实现必须支持全部 6 种格式**。这是互操作性的最低门槛。

### 3.2 声道

输出一律为**单声道**。`float32 [-1, 1]` → 按格式编码。

### 3.3 字符限制

请求文本默认上限 `8000` 字符,由 `MAX_INPUT_CHARS` 环境变量覆盖。超出返回 `413 Payload Too Large`。空字符串返回 `422 Unprocessable Entity`。

### 3.4 鉴权

默认无鉴权(本地/内网部署场景)。如需鉴权,推荐在反向代理层实现(Nginx / Traefik / Cloudflare Access)。

如果实现层提供内置鉴权,应遵循 OpenAI 风格:

```
Authorization: Bearer <API_KEY>
```

API Key 通过环境变量配置(建议 `OPEN_TTS_API_KEY`),缺失时禁用鉴权。

### 3.5 错误响应

所有错误使用 FastAPI 原生 `HTTPException` 结构:

```json
{
  "detail": "human-readable error message"
}
```

| 状态码 | 场景 |
|---|---|
| 400 | 请求结构错误 |
| 401 | 鉴权失败(启用鉴权时) |
| 404 | `voice` 未找到 |
| 413 | `input` 超长 |
| 422 | 字段校验失败(空文本、不支持的 `response_format`、互斥参数冲突等) |
| 500 | 推理或编码失败 |
| 501 | 当前模式/变体不支持该端点 |
| 503 | 引擎尚未加载完成,或并发队列满/超时(见 §3.6) |

错误消息要足以让调用方定位问题,但**不得泄露服务器路径、模型权重路径、堆栈等敏感信息**。内部异常应通过 `log.exception` 记录,对外仅返回简短描述。

### 3.6 并发控制

TTS 推理是 GPU/CPU 密集型操作,同一进程内并行多个推理会相互抢占算力、反而延长单个请求的延迟,严重时还会触发 OOM。所有实现**必须**在服务端限制同时执行的推理任务数。

**配置**(环境变量,见 project-layout §3.3):

| 变量 | 默认 | 说明 |
|---|---|---|
| `MAX_CONCURRENCY` | `1` | 全局并发上限。同一时刻最多允许 N 个推理请求**进入引擎**执行,其余排队等待 |
| `MAX_QUEUE_SIZE` | `0` | 排队上限,`0` 表示不限。超出时立即返回 `503` |
| `QUEUE_TIMEOUT` | `0` | 排队等待超时(秒),`0` 表示不限。等待超过阈值返回 `503` |

**受约束的端点**(进入引擎前必须持有并发令牌):

- `POST /v1/audio/speech`
- `POST /v1/audio/clone`(§5.1)
- `POST /v1/audio/design`(§5.2)
- `POST /v1/audio/realtime`(§5.3,持有令牌直至流结束)

**不受约束的端点**:`GET /healthz`、`GET /v1/audio/voices`、`GET /v1/audio/voices/preview`、`GET /v1/audio/languages`。这些端点不能被推理请求阻塞,否则健康检查会在高负载下失败、触发容器重启。

**行为**:

1. 请求进入 handler 后,**先完成入参校验**(文本长度、格式、voice 存在性等),再去获取并发令牌。这样参数错误能立即返回 `422`/`404`,不占用队列。
2. 获取令牌 = `asyncio.Semaphore.acquire()`。默认 `MAX_CONCURRENCY=1` 时,等同于串行执行。
3. 如果 `MAX_QUEUE_SIZE > 0` 且当前等待者数量已达上限,返回 `503 engine busy: queue full`,不进入等待。
4. 如果 `QUEUE_TIMEOUT > 0` 且等待时间超过阈值,返回 `503 engine busy: queue timeout`。
5. 流式端点(`/v1/audio/realtime`)在流**完全结束**(最后一个 chunk 推送完成或连接中断)后才释放令牌。
6. 客户端断连时必须释放令牌(用 `try/finally` 或 `async with semaphore` 保证)。

**健康检查可选字段**:

`GET /healthz` 的响应可以增加 `concurrency` 字段用于观测(非必需):

```json
{
  "status": "ok",
  "...": "...",
  "concurrency": {
    "max": 1,
    "active": 1,
    "queued": 3
  }
}
```

- `max` — 与 `MAX_CONCURRENCY` 一致。
- `active` — 当前持有令牌的推理请求数,范围 `0..max`。
- `queued` — 当前排队等待的请求数。

客户端不应依赖此字段做调度决策(数值是瞬时快照);用于人工观察或监控指标导出。

### 3.7 跨域资源共享(CORS)

默认**不**启用 CORS。生产部署通常藏在反向代理或与调用方同域,不需要浏览器
跨域支持。实现**必须**提供一个统一的开关,让用户在有浏览器直连场景时一次性
打开所有端点的跨域。

**配置**(环境变量):

| 变量 | 默认 | 说明 |
|---|---|---|
| `CORS_ENABLED` | `false` | 布尔开关。`true` 时挂载 CORS 中间件,允许任意 origin、任意 method、任意 header 访问**所有**端点。`false` 时不挂载中间件,跨域请求由浏览器阻止 |

**行为**(当 `CORS_ENABLED=true`):

- `Access-Control-Allow-Origin: *` —— 允许任何来源。
- `Access-Control-Allow-Methods: *` —— 允许所有方法(含 `OPTIONS` preflight)。
- `Access-Control-Allow-Headers: *` —— 允许所有自定义请求头。
- **不**声明 `Access-Control-Allow-Credentials: true`——`origin=*` 与
  `credentials=true` 在浏览器端互斥;本规范选择简化到"全开放、不携凭证"。
  需要带 cookie / `Authorization` 走跨域的部署者自行修改实现(限定具体
  origin 列表并开启 credentials)——这属于超出本规范的高级用法。
- 应用于**全部**端点,包括核心端点(§4)和所有可选扩展端点(§5)。

**实现细节**:

- FastAPI 实现直接用 `fastapi.middleware.cors.CORSMiddleware(allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])`,在 `app` 创建之后立刻挂载。
- 中间件注册**不**受 `capabilities` 影响——开关独立于引擎能力。
- `CORS_ENABLED=false` 时**不要**注册一个"空壳" CORS 中间件,直接跳过挂载;这样响应头里不会出现任何 `Access-Control-*`,部署在反向代理后时也更干净。

---

## 4. 核心端点(所有实现必须提供)

### 4.1 `GET /healthz`

健康检查与服务元信息。

**响应 200**:

```json
{
  "status": "ok",
  "model": "org/model-id",
  "sample_rate": 24000,
  "device": "cuda:0",
  "dtype": "float16",
  "capabilities": {
    "clone": true,
    "streaming": false,
    "design": true,
    "languages": false,
    "builtin_voices": false
  }
}
```

| 字段 | 类型 | 必需 | 说明 |
|---|---|---|---|
| `status` | `"ok" \| "loading" \| "error"` | 是 | 引擎就绪状态 |
| `model` | string | 是 | 当前加载的模型标识 |
| `sample_rate` | int | 是 | 推理输出采样率(Hz) |
| `capabilities` | object | 是 | 能力发现字段集,见下表 |
| `device` | string | 否 | `"cuda:0"`, `"cpu"` 等(容器部署不会出现 `mps`) |
| `dtype` | string | 否 | `"float16"`, `"bfloat16"`, `"float32"` 等 |
| `concurrency` | object | 否 | 并发使用情况快照,见 §3.6。结构 `{ max, active, queued }` |

**`capabilities` 字段**:

| 字段 | 类型 | 必需 | 说明 |
|---|---|---|---|
| `clone` | bool | 是 | 是否支持声音克隆,对应是否暴露 `POST /v1/audio/clone`、是否接受 `voice="file://..."` |
| `streaming` | bool | 是 | 是否支持流式合成,对应是否暴露 `POST /v1/audio/realtime` |
| `design` | bool | 是 | 是否支持声音设计,对应是否暴露 `POST /v1/audio/design` |
| `languages` | bool | 是 | 是否支持获取语言列表,对应是否暴露 `GET /v1/audio/languages`。**注意:此字段表示引擎是否提供可选语言列表,不代表引擎"支持多语言";有些引擎在合成时不需要显式指定语言,`languages=false`。** |
| `builtin_voices` | bool | 是 | 是否提供内置声音。`true` 时 `/v1/audio/voices` 会包含不带 `file://` 前缀的内置声音条目;`false` 时 `/v1/audio/voices` 只可能出现 `file://` 克隆声音(或为空)。Kokoro 这类仅内置声音的引擎为 `true`,纯克隆引擎(如 OmniVoice、F5-TTS)为 `false`。|

**约束关系**:至少满足 `clone=true`(必须用 `file://...` 调 speech)或 `builtin_voices=true`(必须用内置名调 speech)之一,否则 `/v1/audio/speech` 无可用 voice。两者可同时为 `true`(如 Chatterbox)。

客户端**必须**通过 `capabilities` 判断扩展端点的存在性,不要通过试探 404/501 推断。服务端 `capabilities.X = true` 时,对应端点必须可用;`false` 时,对应端点必须返回 404(FastAPI 默认行为)或 501。

**扩展字段**(引擎可选):`variant`, `quantization`, `attn_implementation`, `loaded_langs`, `mode` 等。新增字段应为可选,且不要与标准字段冲突。引擎可在 `capabilities` 中添加**自定义能力位**,但不得与本规范定义的 5 个标准能力(`clone` / `streaming` / `design` / `languages` / `builtin_voices`)冲突。

**状态码**:
- 引擎加载中返回 200 `status="loading"`,不返回 503。这样健康检查本身总是可达。首次启动若本地无权重,"加载中"包含从 HuggingFace Hub / ModelScope 下载权重的时间,可能持续数分钟到数十分钟;K8s / systemd 探针应把 `startup probe` 的超时上限设得足够宽。
- 仅在进程级致命错误时返回 500。

### 4.2 `GET /v1/audio/voices`

列出所有可用声音。**文件克隆声音以 `file://` 前缀标识,内置声音使用引擎自定义的名称**。见 §4.4 voice 语义。

**响应 200**:

```json
{
  "voices": [
    {
      "id": "file://alice",
      "preview_url": "https://host/v1/audio/voices/preview?id=alice",
      "prompt_text": "Reference transcript for alice's voice sample.",
      "metadata": {
        "name": "Alice",
        "gender": "female",
        "age": "adult",
        "accent": "british",
        "tags": ["warm", "calm"]
      }
    },
    {
      "id": "af_heart",
      "preview_url": null,
      "prompt_text": null,
      "metadata": {
        "name": "Heart",
        "gender": "female",
        "language": "en"
      }
    }
  ]
}
```

| 字段 | 类型 | 必需 | 说明 |
|---|---|---|---|
| `id` | string | 是 | 声音唯一标识,**客户端可直接将此值作为 `POST /v1/audio/speech` 的 `voice` 字段使用**。对文件克隆声音为 `file://<filename>`,对内置声音为引擎定义的名称。 |
| `preview_url` | string \| null | 是 | 声音预览 URL。**仅文件克隆声音**给出指向 `/v1/audio/voices/preview` 的链接;内置声音必须返回 `null`。 |
| `prompt_text` | string \| null | 是 | 参考音频的转录文本。**仅文件克隆声音**返回实际内容;内置声音必须返回 `null`。 |
| `metadata` | object \| null | 是 | 声音的扩展元信息(`name`、`gender`、`language`、`tags` 等)。对文件克隆声音来源于 `${VOICES_DIR}/<id>.yml`,见 §6;对内置声音由引擎填充。无元信息时返回 `null` 或空对象 `{}`。 |

**关键约定**:
- 内置声音**不支持试听**,也没有提示文本——服务端不为其预生成样本,`preview_url` 与 `prompt_text` 两个字段都返回 `null`。
- 文件克隆声音的 `preview_url` 中,查询参数 `id` 使用**去掉 `file://` 前缀**的纯名称(URL 编码处理特殊字符)。
- 客户端通过 `preview_url === null` 判断声音是否可试听,不要依赖字段缺失或空串。

**`metadata` 字段**:

本规范为 `metadata` 定义若干**建议键**,约定语义以保证跨引擎一致:

| 键 | 类型 | 取值约定 |
|---|---|---|
| `name` | string | 人类可读的显示名(UI 选择器用);缺失时客户端回退到 `id` |
| `gender` | string | `"female"` / `"male"` / `"neutral"` / `"other"` |
| `age` | string | `"child"` / `"teen"` / `"adult"` / `"elderly"` |
| `language` | string | 主要语言标识,与 `/v1/audio/languages` 的 `key` 一致 |
| `accent` | string | 口音描述(自由文本,如 `"british"` / `"cantonese"`) |
| `tags` | list[string] | 描述性标签(如 `["warm", "calm"]`),UI 可用于筛选 |
| `description` | string | 长文字描述 |

上述键都**可选**——引擎或声音配置缺失时对应字段就不出现在 `metadata` 中。也允许添加建议键以外的**自定义键**(如 `preview_script`、`license`),客户端应容忍未知键。

**空结果**:

```json
{ "voices": [] }
```

### 4.3 `GET /v1/audio/voices/preview?id={name}`

返回**文件克隆声音**的参考音频(原始 wav)。该端点**仅服务于 `file://` 形式的声音**,内置声音不支持试听。

**查询参数**:
- `id` — 去掉 `file://` 前缀的纯声音名,对应 `${VOICES_DIR}/<id>.wav`。

**行为**:
1. 服务端加载 `${VOICES_DIR}/<id>.wav` 并原样返回(不重采样、不转码)。
2. 如果文件不存在,返回 404。
3. 如果请求的声音是内置声音,返回 404(内置声音无 wav 源文件)。

**响应 200**: `audio/wav` 二进制
**响应 404**: `voice '<id>' not found`

### 4.4 `POST /v1/audio/speech`

核心 TTS 接口,OpenAI 兼容。

**请求体**:

```json
{
  "model": "ignored-for-compatibility",
  "input": "要合成的文本",
  "voice": "file://alice",
  "response_format": "mp3",
  "speed": 1.0,
  "instructions": "speak in a cheerful tone"
}
```

| 字段 | 类型 | 必需 | 默认 | 说明 |
|---|---|---|---|---|
| `model` | string | 否 | null | OpenAI 兼容字段,实现应接受但忽略;可用于 A/B 测试路由 |
| `input` | string | 是 | - | 合成文本,1..`MAX_INPUT_CHARS` |
| `voice` | string | 是 | - | 声音标识,语义见下文"Voice 语义" |
| `response_format` | enum | 否 | `"mp3"` | 见 §3.1 |
| `speed` | float | 否 | `1.0` | 范围 `[0.25, 4.0]`;部分引擎可能忽略,但不应因此报错 |
| `instructions` | string | 否 | null | OpenAI 兼容字段(用于控制语气、风格等)。所有实现必须接受该字段;不支持的引擎应忽略而非报错 |

#### Voice 语义

`voice` 字段根据引擎能力采用三种形态,**客户端无需关心引擎类型**,可直接使用 `/v1/audio/voices` 返回的 `id` 字段填入:

| 引擎类型 | `voice` 形态 | 行为 |
|---|---|---|
| 仅内置声音(如 Kokoro) | `<name>`(普通字符串) | 引擎按名称查找内置声音,例如 `voice="af_heart"` |
| 仅声音克隆(如 OmniVoice、F5-TTS) | `file://<id>` | 引擎加载 `${VOICES_DIR}/<id>.wav` + `${VOICES_DIR}/<id>.txt`,以零样本克隆合成 |
| 同时支持克隆与内置(如 Chatterbox) | 两者并存 | `file://<id>` 走克隆路径;其他普通字符串走内置声音路径 |

**规则**:

1. `file://` 前缀**保留给文件克隆**,服务端看到该前缀时必须将剥离后的部分作为 `{id}` 在 `${VOICES_DIR}` 下查找 `.wav` + `.txt`。
2. 不含 `file://` 前缀的 `voice` 由引擎自行解释(内置声音名、预设说话人 id、或其他协议)。
3. 若 `file://<id>` 对应文件不存在,返回 `404 voice 'file://<id>' not found`。
4. 若普通名称不在内置声音列表中,返回 `404 voice '<name>' not found`。
5. 仅支持克隆的引擎收到不含 `file://` 前缀的 `voice` 时,返回 `422 voice must use 'file://' prefix for clone-only engines`。
6. 仅支持内置声音的引擎收到带 `file://` 前缀的 `voice` 时,返回 `422 file:// voices not supported by this engine`。

其他 URI 协议(`http://`、`https://`、`s3://` 等)**保留**,当前版本未定义行为,引擎可在未来扩展。**不**支持时建议返回 `501 Not Implemented`。

**引擎特定可选字段**:

实现可以扩展额外字段,但必须满足:
1. 所有扩展字段必须为可选(`Optional`),缺失时引擎使用内部默认。
2. 字段命名使用小写蛇形,语义清晰(如 `temperature`, `top_p`, `top_k`, `max_new_tokens`, `repetition_penalty`, `seed`, `num_step`, `guidance_scale`, `cfg_value`, `inference_timesteps`, `denoise`, `normalize`)。
3. 数值字段必须带合理的 `ge`/`le` 校验。
4. 不认识的字段由 Pydantic 按 `extra="ignore"` 丢弃,不报错。

**响应 200**: 音频二进制,`Content-Type` 由 `response_format` 决定。

**响应 404**: `voice '{voice}' not found`
**响应 413**: `input exceeds {N} chars`
**响应 422**: 字段校验失败
**响应 500**: `inference failed: {msg}` 或 `encoding failed: {msg}`

**行为**:

1. 校验 `input` 非空且 ≤ `MAX_INPUT_CHARS`。
2. 校验 `response_format` 在支持列表内。
3. 从声音目录加载 `voice` 对应的 `{id}.wav` + `{id}.txt`。
4. 调用引擎进行零样本克隆合成(以参考音频 + 参考文本为条件)。
5. 编码为目标格式并返回。

---

## 5. 可选扩展端点

引擎可以根据自身能力选择实现以下端点。每个端点的存在性应在文档中声明,客户端可通过 `GET /v1/audio/voices` 或 `/healthz.mode` 等方式推断支持情况,或直接请求并在 501 时降级。

### 音频返回的通用要求

凡是响应体为**音频内容**的扩展端点(`clone`、`design`、`realtime`)都必须遵守以下通用约定,不在各小节中重复:

| 要求 | 说明 |
|---|---|
| `response_format` 参数 | 必须作为请求字段支持,取值 `"mp3"` / `"opus"` / `"aac"` / `"flac"` / `"wav"` / `"pcm"`,默认 `"mp3"`,校验与 §3.1 一致 |
| `Content-Type` | 按 `response_format` 映射,见 §3.1 的 `CONTENT_TYPES` 表 |
| 采样率 | 与 `/healthz.sample_rate` 一致,由引擎在 `audio.encode()` 中注入 |
| 声道 | 单声道,`float32 [-1, 1]` → 按格式编码(§3.2) |
| 不支持的格式 | 返回 `422 unsupported response_format: {fmt}` |

流式端点(`/v1/audio/realtime`)还需满足 §5.3 "格式的流式特性"表的要求:不是所有格式都能流式,引擎可拒绝不可流式的 `response_format` 并返回 422。

非音频返回的端点(如 `/v1/audio/languages` 返回 JSON)不适用本要求。

---

### 5.1 `POST /v1/audio/clone` — 一次性上传克隆

**仅支持声音克隆的引擎才暴露此端点**。与 `/v1/audio/speech + voice="file://..."` 不同,本端点允许客户端在单个请求中上传参考音频、提示词与合成文本,无需预先将音频放入 `${VOICES_DIR}`。适用于:

- 声音目录不在客户端控制范围(如托管场景)。
- 每次合成都使用不同的一次性参考音频(如用户上传的片段)。
- 不希望在服务端持久化客户提供的音源。

**请求**:`multipart/form-data`

| 字段 | 类型 | 必需 | 默认 | 说明 |
|---|---|---|---|---|
| `audio` | file | 是 | - | 参考音频文件。推荐 WAV(PCM 16-bit,16kHz+,5-15 秒);引擎应同时接受 MP3/FLAC/Opus 等常见格式,内部重采样 |
| `prompt_text` | string | 是 | - | 参考音频对应的转录文本。若引擎支持文本无关模式,可传空串(引擎视为允许则忽略,否则返回 422) |
| `input` | string | 是 | - | 合成文本,1..`MAX_INPUT_CHARS` |
| `response_format` | string | 否 | `"mp3"` | 见 §3.1 |
| `speed` | float | 否 | `1.0` | 范围 `[0.25, 4.0]` |
| `instructions` | string | 否 | null | OpenAI 兼容,引擎可忽略 |
| `model` | string | 否 | null | OpenAI 兼容,引擎忽略 |

**引擎特定可选字段**:与 §4.4 相同规则,以 form 字段形式传递(Pydantic 会按字段类型解析)。字符串、数字、布尔都可以直接作为 form 值,数组/对象类型字段应使用嵌套 form 表示法或**单独提供 JSON 字符串字段**。

**请求示例**(curl):

```bash
curl -X POST http://localhost:8000/v1/audio/clone \
  -F 'audio=@./reference.wav' \
  -F 'prompt_text=This is the transcript of the reference clip.' \
  -F 'input=你好,这是要合成的文本。' \
  -F 'response_format=mp3' \
  -F 'speed=1.0' \
  --output out.mp3
```

**行为**:

1. 校验 `audio` 存在且 content-type 为 `audio/*` 或扩展名合法(`.wav`/`.mp3`/`.flac`/`.ogg`/`.opus`/`.m4a`)。
2. 校验 `input` 非空且 ≤ `MAX_INPUT_CHARS`。
3. 校验 `response_format` 在支持列表内。
4. 将上传文件写入临时位置或流式解码为波形。**不得**持久化到 `${VOICES_DIR}`。
5. 调用引擎执行零样本克隆合成。
6. 编码为目标格式并返回。
7. 处理完成后清理临时文件。

**响应 200**: 音频二进制,`Content-Type` 由 `response_format` 决定。

**错误响应**:

| 状态 | 场景 |
|---|---|
| 400 | multipart 解析失败 / `audio` 缺失 |
| 413 | `input` 超长或 `audio` 超过 `MAX_AUDIO_BYTES`(默认 20 MB,可通过 env 覆盖) |
| 415 | `audio` 文件格式无法识别 |
| 422 | `input`/`prompt_text` 为空(且引擎不支持文本无关模式) / 不支持的 `response_format` / 引擎特定字段越界 |
| 500 | `inference failed: {msg}` / `audio decode failed: {msg}` / `encoding failed: {msg}` |
| 501 | 引擎不支持克隆(**该端点在此类引擎上不应暴露,若意外收到应返回 501**) |

**能力声明**:

引擎通过 `/healthz.capabilities.clone` 声明是否支持此端点。Kokoro 等仅内置声音的引擎**不实现**此端点,`capabilities.clone=false`,请求时由 FastAPI 默认返回 404。

### 5.2 `POST /v1/audio/design` — 声音设计

基于自然语言描述生成声音,不依赖参考音频。

**请求**(通用字段):

```json
{
  "input": "要合成的文本",
  "instruct": "female, british accent, warm tone",
  "response_format": "mp3"
}
```

| 字段 | 类型 | 必需 | 默认 | 说明 |
|---|---|---|---|---|
| `input` | string | 是 | - | 合成文本,1..`MAX_INPUT_CHARS` |
| `instruct` | string \| null | 否 | null | 声音设计描述(如 "female, british accent, warm tone")。**传 `null` 或空字符串时服务端必须接受不报错**,由引擎按内部默认声音合成 |
| `response_format` | string | 否 | `"mp3"` | 见 §3.1 |

**引擎扩展**:引擎可在上述通用字段之上增加自有参数(如 `language`、`gender`、`pitch`、`speed`、`temperature`、`top_p`、`num_step`、`cfg_value` 等),遵循 §4.4 "引擎特定可选字段" 的约束:必须是 `Optional`、使用小写蛇形、数值字段带 `ge`/`le` 校验、未知字段按 `extra="ignore"` 丢弃。

客户端在不知道具体引擎的情况下,只依赖上表的通用字段即可获得可用的合成结果;有引擎特定需求时再按该引擎文档附加字段。

**能力声明**:引擎通过 `/healthz.capabilities.design` 声明是否支持此端点。`false` 时不暴露该路由。

### 5.3 `POST /v1/audio/realtime` — 流式合成

**仅支持流式生成的引擎暴露此端点**。请求字段与 `/v1/audio/speech` 完全一致(§4.4 的所有必需与可选字段、voice 语义、校验规则都适用),区别仅在响应:本端点使用 HTTP 分块传输(`Transfer-Encoding: chunked`),边推理边推送音频字节,客户端可边接收边播放。

**请求**: `application/json`,字段见 §4.4(`model` / `input` / `voice` / `response_format` / `speed` / `instructions` / 引擎特定字段)。

**响应**:

| Header | 值 |
|---|---|
| `Content-Type` | 由 `response_format` 决定,见 §3.1 |
| `Transfer-Encoding` | `chunked` |
| `Content-Length` | **不**设置(不能预知长度) |

响应体是一个连续的音频字节流,格式与 `/v1/audio/speech` 的非流式响应二进制一致,可以直接保存成文件播放。

**格式的流式特性**:

| `response_format` | 流式支持 | 说明 |
|---|---|---|
| `pcm` | ★ 天然支持 | 原始 int16 小端,最易流式 |
| `mp3` | ★ 天然支持 | MP3 frame 可独立解码 |
| `opus` | ★ 天然支持 | OGG page 流式封装 |
| `aac` | ★ 天然支持 | ADTS 每帧独立 |
| `wav` | △ 有限支持 | 服务端写入时将 `data` size 字段置 `0xFFFFFFFF` 或 `0`,客户端按字节流解码 |
| `flac` | ✗ 不推荐 | 需要完整 frame/stream info;引擎可返回 `422 flac not supported in realtime` 或退化为生成完再整体发送 |

**实现最低要求**:`pcm` 和 `mp3` 必须可流式;其他格式按引擎能力选择性支持。

**请求示例**:

```bash
curl -N -X POST http://localhost:8000/v1/audio/realtime \
  -H 'Content-Type: application/json' \
  -d '{"input": "要流式合成的文本", "voice": "file://alice", "response_format": "mp3"}' \
  --output out.mp3
```

`curl -N` 禁用客户端缓冲,模拟实时播放。

**行为**:

1. 请求入参校验与 `/v1/audio/speech` 相同(文本长度、格式、voice 存在性、前缀与引擎能力匹配)。
2. 引擎的合成方法必须返回异步迭代器/生成器(`AsyncIterator[np.ndarray]` 或 `AsyncIterator[bytes]`)。
3. 服务端逐块编码并通过 FastAPI `StreamingResponse` 推送。编码器需要**增量状态**(如 MP3 的 PyAV encoder),不能对每 chunk 独立调用 `encode()` 后拼接——那会在 chunk 之间插入格式头导致播放中断。
4. 推理期间若发生不可恢复错误:断开连接,客户端通过截断的流感知;服务端 `log.exception`。
5. 首字节延迟(TTFB)目标:模型首 chunk 生成即推送,不要为减少 chunk 数而积攒。

**错误响应**:

| 状态 | 场景 | 说明 |
|---|---|---|
| 200 | 流开始 | 正常路径 |
| 404 | voice 未找到 | 流开始前返回 JSON body |
| 413 | `input` 过长 | 同上 |
| 422 | 参数错误 / 请求的 `response_format` 不能流式 | 同上 |
| 500 | 流开始前的推理初始化失败 | 同上 |
| 501 | 引擎不支持流式 | 该端点不应在此类引擎上暴露,若意外收到返回 JSON body |

流**开始后**不能再返回 HTTP 错误码。错误只能通过连接中止暴露。

**能力声明**:

引擎通过 `/healthz.capabilities.streaming` 声明是否支持此端点。`false` 时不暴露该路由。

**客户端注意事项**:

- 播放器应使用增量解码器(如 HTML5 `<audio>` + MediaSource Extensions 对应 mp3,或 Web Audio API 对 pcm)。
- 不要等待连接关闭后再解码,否则等同非流式。
- 对 `pcm` 格式,客户端需要预先知道采样率(通过 `/healthz.sample_rate`),因为 raw PCM 无自描述头。

### 5.4 `GET /v1/audio/languages` — 支持的语言

```json
{
  "languages": [
    { "key": "en", "name": "English" },
    { "key": "zh", "name": "Chinese" },
    { "key": "ja", "name": "Japanese" }
  ]
}
```

| 字段 | 类型 | 必需 | 说明 |
|---|---|---|---|
| `key` | string | 是 | **引擎接受的语言标识原值**,具体格式由引擎决定。可能是 ISO 639-1(`en`)、BCP 47(`zh-CN`)、语言名全称(`English`)或引擎内部码。客户端应原样在引擎扩展字段(如 `language`)中回传,不做规范化。 |
| `name` | string | 是 | 人类可读的语言名称,用于 UI 展示 |

**空结果**:`{ "languages": [] }`

用于需要显式指定语言的多语言模型(如 Chatterbox multilingual)。

**能力声明**:引擎通过 `/healthz.capabilities.languages` 声明是否支持此端点。`false` 时不暴露该路由。注意此字段**只**表示引擎是否提供语言列表,不代表引擎是否"支持多语言"——有些引擎在合成时不需要显式指定语言,`capabilities.languages=false` 但仍可能合成多种语言。

---

## 6. 声音目录约定

所有支持声音克隆的引擎遵循同一目录规范:

```
${VOICES_DIR}/
├── alice.wav          # 必需,参考音频(PCM WAV 推荐,16kHz+,5-15 秒为佳)
├── alice.txt          # 必需,UTF-8 文本,支持 BOM;参考音频对应的转录
├── alice.yml          # 可选,YAML 元信息(见下文 schema)
├── bob.wav
├── bob.txt
└── ...
```

**扫描规则**:

1. 仅扫描顶层目录的 `.wav`、`.txt` 和 `.yml` 文件(不递归)。
2. 文件名去扩展名后得到 `<id>`,`/v1/audio/voices` 返回的 `VoiceInfo.id` 为 `file://<id>`,`POST /v1/audio/speech` 的 `voice` 字段使用同样形式。
3. `<id>.wav` 与 `<id>.txt` **必须同时存在且非空**,否则跳过该声音并记录 warning。
4. `<id>.yml` 可选;缺失时 `VoiceInfo.metadata` 为 `null`。存在但解析失败(非法 YAML)时跳过该元信息文件(不影响声音本身加载),并记录 warning。
5. `.txt` 允许 UTF-8 BOM,读取时自动剥离。
6. `mtime` 取存在文件(`.wav` / `.txt` / `.yml`)中最新的时间,用于缓存失效。
7. `<id>` 可包含 URL-safe 字符以外的字符(中文、空格等),在 `preview_url` 中需做 URL 编码;文件系统上保持原样。

**`<id>.yml` 元信息 schema**:

扁平 key-value 结构,所有字段**可选**。建议键集合与 §4.2 `metadata` 字段表一致,示例:

```yaml
# alice.yml
name: "Alice"
gender: female
age: adult
language: en
accent: british
tags:
  - warm
  - calm
description: "A warm female voice with British accent."
```

- 值类型按 YAML 自然推断:字符串、数字、布尔、列表、嵌套对象皆可。
- 允许添加建议键以外的**自定义键**,服务端原样保留并通过 `VoiceInfo.metadata` 暴露。
- 顶层**必须**是 mapping(对象),不接受列表或标量作为根节点。
- 文件编码 UTF-8,支持 BOM;超过 64 KB 时建议拆分(声音元信息本身不应承载大段文本)。

**挂载**:推荐以只读方式挂载(`:ro`),避免容器写入污染宿主目录:

```
docker run -v ./voices:/voices:ro ...
```

---

## 7. 客户端契约

### 7.1 OpenAI SDK 兼容示例

```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8000/v1", api_key="not-needed")

# 文件克隆(引擎支持 clone)
resp = client.audio.speech.create(
    model="any",
    voice="file://alice",
    input="你好,世界",
    response_format="mp3",
)
resp.stream_to_file("out.mp3")

# 内置声音(如 Kokoro)
resp = client.audio.speech.create(
    model="any",
    voice="af_heart",
    input="你好,世界",
    response_format="mp3",
)
resp.stream_to_file("out.mp3")
```

### 7.2 curl 示例

```bash
# 文件克隆
curl -X POST http://localhost:8000/v1/audio/speech \
  -H 'Content-Type: application/json' \
  -d '{"input": "hello world", "voice": "file://alice", "response_format": "wav"}' \
  --output out.wav

# 内置声音
curl -X POST http://localhost:8000/v1/audio/speech \
  -H 'Content-Type: application/json' \
  -d '{"input": "hello world", "voice": "af_heart", "response_format": "wav"}' \
  --output out.wav
```

### 7.3 能力发现

客户端推荐的发现顺序:

1. `GET /healthz` 确认服务可用,并读取 `capabilities` 字段判断引擎支持的扩展端点(`clone` / `streaming` / `design` / `languages` / `builtin_voices`)。
2. `GET /v1/audio/voices` 获取可用声音,**直接取返回的 `id` 字段作为 `voice` 使用**,不需要判断引擎类型或自己拼接 `file://` 前缀。
3. 根据 `capabilities` 决定是否调用扩展端点,不要靠试探 404/501。

---

## 8. 版本兼容与演进

- **v1 保证**:§4 的核心端点、§3 的 6 种音频格式、`VoiceInfo` 的 3 个必需字段(`id`/`preview_url`/`prompt_text`)、§4.4 的 voice 语义(`file://` 前缀保留给文件克隆)、§3.6 的并发控制行为(推理类端点受令牌约束,观测类端点不受限)。
- **新增可选字段**不算破坏性变更,客户端必须容忍未知字段。
- **修改字段语义**或**删除字段**需要 bump 到 `/v2`,并维护 `/v1` 至少 6 个月。
- **扩展端点**(§5)可在任何小版本加入或移除,客户端应基于 404/501 降级。

---

## 9. 附录:字段校验范围建议

| 字段 | 范围 | 说明 |
|---|---|---|
| `input` | `1..MAX_INPUT_CHARS` | 默认 8000 |
| `speed` | `0.25..4.0` | 与 OpenAI 一致 |
| `temperature` | `0.0..2.0` | 0 等价贪心 |
| `top_p` | `0.0..1.0` | |
| `top_k` | `0..∞` | 0 表示禁用 |
| `max_new_tokens` | `1..16384` | |
| `repetition_penalty` | `0.5..2.0` | |
| `num_step` | `1..64` | 扩散步数 |
| `guidance_scale` | `0.0..10.0` | 分类器自由指导 |
| `cfg_value` | `0.1..10.0` | |
| `inference_timesteps` | `1..100` | |
| `duration` | `0.5..120.0` (秒) | |

所有数值参数必须在 schemas 中声明 `Field(ge=..., le=...)`,由 Pydantic 在入参阶段校验,引擎层不再重复检查。
