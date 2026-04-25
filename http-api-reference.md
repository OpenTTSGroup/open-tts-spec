# Open TTS HTTP API 使用指南

> 面向 API 调用方的接口文档。涵盖端点、字段、错误码、调用示例。

---

## 1. 概览

Open TTS 提供一组统一的 HTTP TTS 接口,覆盖语音合成、声音克隆、声音设计、流式合成、声音列表等能力。

| 方法 | 路径 | 必有 | 说明 |
|---|---|---|---|
| GET | `/healthz` | ✓ | 健康检查 + 能力发现 |
| GET | `/v1/audio/voices` | ✓ | 列出可用声音 |
| GET | `/v1/audio/voices/preview?id=` | ✓ | 试听声音 |
| POST | `/v1/audio/speech` | ✓ | 核心 TTS |
| POST | `/v1/audio/clone` | ○ | 一次性上传克隆(multipart) |
| POST | `/v1/audio/design` | ○ | 声音设计(自然语言描述) |
| POST | `/v1/audio/realtime` | ○ | 流式合成(chunked) |
| GET | `/v1/audio/languages` | ○ | 支持的语言列表 |

✓ = 所有部署都提供。○ = 视引擎能力,通过 `/healthz.capabilities` 判断,**不要**靠试探 404/501。

---

## 2. 基础信息

- **Base URL**:由部署方提供,示例使用 `http://localhost:8000`。
- **API 版本**:所有业务端点位于 `/v1/audio/*` 下。
- **请求体**:`POST /v1/audio/speech` / `/design` / `/realtime` 使用 `application/json`;`/clone` 使用 `multipart/form-data`。
- **响应**:音频端点返回二进制流,JSON 端点返回 UTF-8 JSON。

### 2.1 鉴权

默认无鉴权(本地/内网部署)。如果部署方启用了鉴权,使用 Bearer Token:

```http
Authorization: Bearer <API_KEY>
```

具体是否启用、Key 如何获取,请咨询部署方。

### 2.2 音频格式

`response_format` 字段决定输出格式与 `Content-Type`:

| `response_format` | Content-Type | 备注 |
|---|---|---|
| `mp3` (默认) | `audio/mpeg` | 通用度最高 |
| `opus` | `audio/ogg` | 低延迟、低带宽 |
| `aac` | `audio/aac` | iOS / Safari 友好 |
| `flac` | `audio/flac` | 无损 |
| `wav` | `audio/wav` | PCM_16 |
| `pcm` | `application/octet-stream` | 原始小端 int16,采样率见 `/healthz.sample_rate` |

**所有部署支持上述全部 6 种格式**。输出始终为单声道。

### 2.3 文本长度

`input` 默认上限 `8000` 字符(部署方可调)。超出返回 `413`,空字符串返回 `422`。

---

## 3. 快速开始

### 3.1 curl

```bash
curl -X POST http://localhost:8000/v1/audio/speech \
  -H 'Content-Type: application/json' \
  -d '{"input": "hello world", "voice": "af_heart", "response_format": "wav"}' \
  --output out.wav
```

### 3.2 Python(httpx)

```python
import httpx

resp = httpx.post(
    "http://localhost:8000/v1/audio/speech",
    json={
        "input": "你好,世界",
        "voice": "af_heart",
        "response_format": "mp3",
    },
    timeout=60.0,
)
resp.raise_for_status()
with open("out.mp3", "wb") as f:
    f.write(resp.content)
```

### 3.3 推荐的调用顺序

1. **`GET /healthz`** —— 确认服务可用,读取 `capabilities` 判断扩展端点是否支持。
2. **`GET /v1/audio/voices`** —— 获取可用声音列表,直接取返回的 `id` 字段作为 `voice` 参数。
3. **`POST /v1/audio/speech`**(或扩展端点)—— 发起合成。

---

## 4. 核心端点

### 4.1 `GET /healthz`

健康检查 + 服务能力发现。

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

| 字段 | 类型 | 说明 |
|---|---|---|
| `status` | `"ok"` / `"loading"` / `"error"` | `loading` 表示模型仍在加载,稍后重试 |
| `model` | string | 当前加载的模型标识 |
| `sample_rate` | int | 推理输出采样率(Hz);**`pcm` 格式需要这个值** |
| `device` | string | 如 `"cuda:0"` / `"cpu"` |
| `dtype` | string | 如 `"float16"` / `"bfloat16"` |
| `capabilities.clone` | bool | 是否支持 `/v1/audio/clone` |
| `capabilities.streaming` | bool | 是否支持 `/v1/audio/realtime` |
| `capabilities.design` | bool | 是否支持 `/v1/audio/design` |
| `capabilities.languages` | bool | 是否提供语言列表(**不**等于"是否支持多语言") |
| `capabilities.builtin_voices` | bool | 是否包含内置声音 |

**注意**:
- 服务启动期间可能首次下载模型权重,持续数分钟;此期间 `status="loading"` 且仍返回 200。
- 客户端应**始终**通过 `capabilities` 判断扩展端点支持情况。
- 引擎可能附加自定义字段(如 `variant`、`quantization`),客户端应容忍未知字段。

### 4.2 `GET /v1/audio/voices`

列出所有可用声音。

**响应 200**:

```json
{
  "voices": [
    {
      "id": "file://alice",
      "preview_url": "http://host/v1/audio/voices/preview?id=alice",
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

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | string | 声音唯一标识。**直接作为 `voice` 字段填入合成请求**,客户端不需要解析或转换 |
| `preview_url` | string \| null | 试听 URL,客户端可直接 GET。无试听样本时为 `null` |
| `prompt_text` | string \| null | 参考音频转录文本。无参考转录时为 `null` |
| `metadata` | object \| null | 扩展元信息(`name`/`gender`/`age`/`language`/`accent`/`tags`/`description` 等),所有键可选,客户端应容忍未知键 |

通过 `preview_url === null` 判断该声音是否可试听。

### 4.3 `GET /v1/audio/voices/preview?id={name}`

返回声音的参考音频(原始 wav)。

**调用方通常无需手动构造此 URL** —— 直接使用 `/v1/audio/voices` 返回的 `preview_url` 字段 GET 即可,服务端已经填好了正确的查询参数。

**响应 200**:`audio/wav` 二进制
**响应 404**:`voice '<id>' not found`(声音不存在或不支持试听,见 §4.2 `preview_url === null`)

### 4.4 `POST /v1/audio/speech`

核心 TTS 接口。

**请求体**(`application/json`):

```json
{
  "input": "要合成的文本",
  "voice": "af_heart",
  "response_format": "mp3",
  "speed": 1.0,
  "instructions": "speak in a cheerful tone"
}
```

| 字段 | 类型 | 必需 | 默认 | 说明 |
|---|---|---|---|---|
| `input` | string | 是 | - | 合成文本,1..`MAX_INPUT_CHARS` |
| `voice` | string | 是 | - | 声音标识,直接使用 `/v1/audio/voices` 返回的 `id` |
| `response_format` | enum | 否 | `"mp3"` | 见 §2.2 |
| `speed` | float | 否 | `1.0` | `[0.25, 4.0]`;部分引擎不支持时静默忽略,不会报错 |
| `instructions` | string | 否 | null | 用于控制语气、风格等自由文本提示;部分引擎不支持时静默忽略 |
| `model` | string | 否 | null | 兼容字段,服务端接受但忽略;可用于 A/B 路由或日志区分 |

#### Voice 字段使用

`voice` 字段直接填 `/v1/audio/voices` 返回的 `id` 即可。服务端会按声音类型自动路由,客户端不需要做任何解析或拼接。

| 错误 | 原因 |
|---|---|
| `404 voice '<voice>' not found` | 声音不存在(`id` 拼错,或对应声音已被部署方移除) |
| `422 voice 不被当前引擎接受` | 客户端使用了不在 `/v1/audio/voices` 返回列表中的 `voice` 值。**先通过 `/v1/audio/voices` 取 `id` 即可避免** |

#### 引擎特定可选字段

部分引擎接受额外参数(如 `temperature`、`top_p`、`top_k`、`seed`、`num_step`、`guidance_scale` 等)。这些字段:

- 全部为可选,缺省时使用引擎内部默认。
- 服务端按 `extra="ignore"` 处理未知字段,不会因为字段不被识别而报错。
- 数值范围参考 §7。

具体支持哪些字段以及对应取值,请查阅部署方使用的引擎文档。

**响应 200**:音频二进制,`Content-Type` 由 `response_format` 决定。

---

## 5. 扩展端点

调用前先通过 `GET /healthz.capabilities` 判断对应能力是否为 `true`。

### 5.1 `POST /v1/audio/clone` —— 一次性上传克隆

适用于:声音文件不在服务端目录、每次合成使用不同的临时参考音频、不希望服务端持久化音源。

仅当 `capabilities.clone === true` 时可用。

**请求**:`multipart/form-data`

| 字段 | 类型 | 必需 | 默认 | 说明 |
|---|---|---|---|---|
| `audio` | file | 是 | - | 参考音频。推荐 WAV(PCM 16-bit,16kHz+,5–15 秒);也接受 MP3 / FLAC / Opus / M4A,服务端内部重采样 |
| `prompt_text` | string | 是 | - | 参考音频对应转录;若引擎支持文本无关模式可传空 |
| `input` | string | 是 | - | 合成文本,1..`MAX_INPUT_CHARS` |
| `response_format` | string | 否 | `"mp3"` | 见 §2.2 |
| `speed` | float | 否 | `1.0` | `[0.25, 4.0]` |
| `instructions` | string | 否 | null | 自由文本风格提示,部分引擎不支持时静默忽略 |
| `model` | string | 否 | null | 兼容字段,服务端忽略 |

引擎特定字段以 form 字段形式追加(规则同 §4.4)。

**示例**:

```bash
curl -X POST http://localhost:8000/v1/audio/clone \
  -F 'audio=@./reference.wav' \
  -F 'prompt_text=This is the transcript of the reference clip.' \
  -F 'input=你好,这是要合成的文本。' \
  -F 'response_format=mp3' \
  --output out.mp3
```

**响应 200**:音频二进制。

**特有错误码**:

| 状态 | 场景 |
|---|---|
| 400 | multipart 解析失败 / `audio` 缺失 |
| 413 | `input` 超长 或 `audio` 超过部署方设置的字节上限(默认 20 MB) |
| 415 | `audio` 文件格式无法识别 |

### 5.2 `POST /v1/audio/design` —— 声音设计

通过自然语言描述生成声音,不需要参考音频。仅当 `capabilities.design === true` 时可用。

**请求**(`application/json`):

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
| `instruct` | string \| null | 否 | null | 声音描述;**传 null 或空字符串时服务端使用引擎默认声音,不报错** |
| `response_format` | string | 否 | `"mp3"` | 见 §2.2 |

引擎可能接受额外字段(如 `language`、`gender`、`pitch`、`temperature`、`num_step`、`cfg_value`),具体见引擎文档。

**响应 200**:音频二进制。

### 5.3 `POST /v1/audio/realtime` —— 流式合成

请求字段与 `/v1/audio/speech` **完全一致**(§4.4)。响应通过 HTTP `Transfer-Encoding: chunked` 边推理边推送,客户端可边接收边播放。

仅当 `capabilities.streaming === true` 时可用。

#### 响应

| Header | 值 |
|---|---|
| `Content-Type` | 由 `response_format` 决定 |
| `Transfer-Encoding` | `chunked` |
| `Content-Length` | **不**设置 |

#### 各格式的流式支持

| `response_format` | 流式 | 备注 |
|---|---|---|
| `pcm` | ★ | 原始 int16 小端,客户端需要从 `/healthz.sample_rate` 读取采样率 |
| `mp3` | ★ | 推荐用于浏览器端 MSE |
| `opus` | ★ | 低延迟低带宽 |
| `aac` | ★ | iOS / Safari 友好 |
| `wav` | △ | header 中 size 字段不准,按字节流解码即可 |
| `flac` | ✗ | 多数引擎不支持流式 FLAC,可能返回 `422 flac not supported in realtime` |

**`pcm` 与 `mp3` 在所有支持流式的部署上保证可用**;其他格式按引擎能力。

#### 调用注意

```bash
curl -N -X POST http://localhost:8000/v1/audio/realtime \
  -H 'Content-Type: application/json' \
  -d '{"input": "要流式合成的文本", "voice": "af_heart", "response_format": "mp3"}' \
  --output out.mp3
```

`curl -N` 禁用客户端缓冲,模拟实时播放。

- HTTP 错误码(404/413/422/500/501)只在**流开始前**返回。
- 流开始后再发生错误,服务端会**直接断开连接**,客户端通过截断的流感知;请准备处理不完整字节流。
- 浏览器播放推荐:HTML5 `<audio>` + MediaSource Extensions(mp3/opus/aac);Web Audio API(pcm,需要预知采样率)。
- **不要**等待连接关闭再解码,否则等同非流式。

### 5.4 `GET /v1/audio/languages` —— 支持的语言

仅当 `capabilities.languages === true` 时可用。

**响应 200**:

```json
{
  "languages": [
    { "key": "en", "name": "English" },
    { "key": "zh", "name": "Chinese" },
    { "key": "ja", "name": "Japanese" }
  ]
}
```

| 字段 | 说明 |
|---|---|
| `key` | 引擎接受的语言标识原值。**原样**回传到引擎扩展字段(如 `language`),不要规范化 |
| `name` | 人类可读的语言名,UI 展示用 |

---

## 6. 错误响应

所有错误使用统一 JSON 结构:

```json
{ "detail": "human-readable error message" }
```

| 状态码 | 含义 |
|---|---|
| 400 | 请求结构错误(JSON 解析失败、multipart 解析失败) |
| 401 | 鉴权失败(部署启用了 Bearer Token 鉴权) |
| 404 | `voice` 未找到 |
| 413 | `input` 超长,或上传 `audio` 超过字节上限 |
| 415 | `/v1/audio/clone` 上传文件格式无法识别 |
| 422 | 字段校验失败(空文本、不支持的 `response_format`、引擎特定字段越界、voice 前缀与引擎能力不匹配) |
| 500 | `inference failed: ...` 或 `encoding failed: ...` |
| 501 | 该端点在当前引擎/模式下不可用(应优先通过 `capabilities` 避免) |
| 503 | 引擎尚未加载完成,或并发队列已满/排队超时 |

### 6.1 503 与并发

部署方会限制服务端同时执行的推理数。当队列已满或排队超时,会立即返回:

- `503 engine busy: queue full`
- `503 engine busy: queue timeout`

**客户端建议**:对 503 实现**指数退避重试**(50ms → 100ms → 200ms…)。GET 类端点(`/healthz`、`/v1/audio/voices`、`/v1/audio/voices/preview`、`/v1/audio/languages`)不会受并发限制影响。

### 6.2 错误信息

错误消息为简短的人类可读字符串,不包含服务器路径、堆栈等敏感信息。生产环境调试请联系部署方查看服务端日志。

---

## 7. 字段范围参考

| 字段 | 范围 | 备注 |
|---|---|---|
| `input` | `1..MAX_INPUT_CHARS`(默认 8000) | 部署方可调 |
| `speed` | `0.25..4.0` | 语速倍率,1.0 为原速 |
| `temperature` | `0.0..2.0` | 引擎特定;0 等价贪心 |
| `top_p` | `0.0..1.0` | 引擎特定 |
| `top_k` | `0..∞` | 引擎特定;0 表示禁用 |
| `max_new_tokens` | `1..16384` | 引擎特定 |
| `repetition_penalty` | `0.5..2.0` | 引擎特定 |
| `num_step` | `1..64` | 引擎特定(扩散步数) |
| `guidance_scale` | `0.0..10.0` | 引擎特定(CFG) |
| `cfg_value` | `0.1..10.0` | 引擎特定 |
| `inference_timesteps` | `1..100` | 引擎特定 |
| `duration` | `0.5..120.0` 秒 | 引擎特定 |

引擎特定字段是否被该部署支持以及具体含义,以引擎文档为准。

---

## 8. 客户端 FAQ

**Q:我怎么知道服务端支持克隆/流式?**
调用 `GET /healthz`,读取 `capabilities`。**不要**靠 404/501 试探。

**Q:`voice` 字段需要做什么处理吗?**
**不需要任何处理**。把 `/v1/audio/voices` 返回的 `id` 原样填进去即可,服务端会自动路由。

**Q:`model` 字段是必需的吗?**
不是必需的,服务端忽略该字段。如果你的 HTTP 客户端框架强制要求填,任意字符串都可以。

**Q:`pcm` 格式没有 header,我怎么播放?**
读取 `/healthz.sample_rate` 得到采样率(常见为 24000Hz),格式固定为单声道、小端 int16。

**Q:`flac` 流式可以用吗?**
不推荐,大多数部署会返回 `422`。流式场景请用 `mp3` 或 `pcm`。

**Q:遇到 503 怎么办?**
指数退避重试。这是正常的并发保护。如果频繁 503,联系部署方调整 `MAX_CONCURRENCY` 或扩容。

**Q:`instructions` / `speed` 没生效?**
部分引擎不支持这两个字段,服务端**静默忽略**,不会报错。具体支持情况见引擎文档。

**Q:声音目录我能直接修改吗?**
不能,目录由部署方挂载。需要新增声音请联系部署方,或使用 `POST /v1/audio/clone` 上传一次性参考音频。

**Q:怎么处理 `status="loading"`?**
服务首次启动可能下载权重(数分钟到数十分钟)。客户端应等待 `status === "ok"` 再开始合成请求。

---

## 9. 兼容性承诺

- 核心端点(§4)、6 种音频格式、`VoiceInfo` 的 `id`/`preview_url`/`prompt_text` 三个字段、以及 "`voice` 字段直接复用 `/v1/audio/voices` 返回的 `id`" 这一调用约定在 v1 内**保持稳定**。
- 服务端可能新增可选字段或新增扩展端点,客户端需**容忍未知字段**。
