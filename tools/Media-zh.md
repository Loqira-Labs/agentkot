# Media

通过配置的提供商、经由单一且与提供商无关的接口生成媒体的工具。创建图像、视频和音频（音乐或语音合成——由模型种类决定）。

## 操作

该工具通过 `operation` 字段区分操作。允许的值：`generate_image`、`generate_video`、`generate_audio`。

所有操作共用的参数：

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| `operation` | string | yes | — | 三种操作之一 |
| `model` | string | yes | — | 媒体模型的精确标识符，原样传递 |
| `provider` | string | no | 无 | 已注册提供商的精确名称；仅当模型标识符在各提供商间唯一时才可省略 |
| `prompt` | string | yes | — | 任务文本；对于语音合成——要朗读的文本 |
| `save_to` | string | 取决于操作 | — | 输出文件或目录的路径；视频和音频必填，图像可选 |

媒体模型及其种类通过 `Agent{list_models}` 发现：媒体是 `for_chat: false` 且指定了种类（image/video/audio）的卡片。提供商名称通过 `Agent{list_providers}` 发现。

### generate_image

生成图像；参考图驱动的生成和编辑使用同一操作。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| `n` | integer | no | 支持或未声明时为1 | 正数数量提示 |
| `size` | string | no | 支持或未声明时为1024x1024 | 尺寸提示 |
| `format` | string | no | 支持或未声明时为png | png或jpeg |
| `quality` | string | no | 无 | 质量提示 |
| `style` | string | no | 无 | 风格提示 |
| `negative_prompt` | string | no | 无 | 要避免的内容 |
| `seed` | integer | no | 无 | 确定性种子提示 |
| `background` | string | no | 无 | transparent、opaque或auto |
| `images` | array | no | 空 | 原始输入引用；每项包含source和必填media_type |
| `mask` | object | no | 无 | 原始蒙版引用；至少需要一张输入图像 |

`Agent{list_models}` 发布 `image.options`。声明不支持的提示不会发送，仅显式提供的提示列入 `ignored`；不支持的images/mask在生成前返回类型化unsupported。没有声明时由适配器判断。原图不进行vision归一化；LocalPath从cwd解析，URL下载不携带提供商凭证，声明的MIME须匹配实际字节。

订阅图像id同时由openai和openai-codex发布：有歧义时必须指定provider。两个名称连接同一Codex Images后端。声明支持background和images；订阅服务器决定单张PNG及其他参数。Direct OpenAI图像行需要API密钥。

返回saved（path、bytes、mime）、usage、request_id、revised_prompt及非空ignored。未提供save_to时返回内联图像块。

### generate_video

生成、编辑和延长视频。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| `save_to` | string | yes | — | 输出文件或目录 |
| `task` | string | no | `"auto"` | `auto`、`text_to_video`、`image_to_video`、`reference_to_video`、`edit`、`extend` |
| `images` | array | no | 空 | 1–2 张图像转视频帧（首帧+末帧），或风格/主体参考 |
| `input_video` | object | no | 无 | 用于编辑或延长的原视频 |
| `continuation_id` | string | no | 无 | 继续先前的有状态结果 |
| `stateful` | boolean | no | `false` | 将新结果保存在服务端 |
| `duration_seconds` | number | no | 支持或未声明时为 `5.0` | 有限且为正的时长提示 |
| `aspect_ratio` | string | no | 无 | 宽高比提示 |
| `resolution` | string | no | 无 | 分辨率提示；声明支持时原样传递 |
| `fps` | integer | no | 无 | 每秒帧数提示，最小为 1 |
| `seed` | integer | no | 无 | 确定性种子提示 |

每张图像和输入视频都需要 `source`（带类型标签的来源对象）和 `media_type`。

`Agent{list_models}` 发布模型声明的 `video.tasks` 和 `video.options`。声明不支持的提示不会发送，其名称列在 `ignored` 中；不支持的任务或 `continuation_id` 在生成前返回类型化 `unsupported`。没有视频声明时，由适配器判断支持情况。未设置且不支持的 `duration` 不发送；不支持的 `stateful` 仅在值为 `true` 时列入 `ignored`。

返回 `saved`、可选的 `continuation_id`，以及过滤了提示时的非空 `ignored`。

任务形态：`text_to_video` 不接受媒体输入；`image_to_video` 接受 1–2 张图像，不接受视频或 continuation；`reference_to_video` 至少需要一张图像，不接受视频或 continuation。`edit` 要求原视频和 continuation 二选一；`extend` 需要原视频且不允许 continuation。编辑和延长可附带参考图像。`auto` 不允许同时传入视频和 continuation，原样交给适配器；已声明支持时，必须至少有一个与输入兼容的任务。

### generate_audio

创建音乐或合成语音——路线由模型种类决定。

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| `save_to` | string | yes | — | 输出文件或目录的路径 |
| `duration_seconds` | integer | no | 无 | 以整秒为单位的时长提示，最小为 1；仅音乐模型 |
| `voice` | string | no | 无 | 语音标识符；语音合成模型必填 |
| `speed` | number | no | 无 | 语速；有限且为正数 |
| `format` | string | no | 语音为 `"wav"` | 音频格式；用于语音合成模型 |

返回：`saved`、`usage`、`request_id`，对于音乐还有 `description`。

对于音乐模型（`AudioGeneration`），参数 `voice`、`speed` 和 `format` 被禁止——传入它们会返回错误。对于语音合成模型（`TextToSpeech`），`voice` 参数必填，`duration_seconds` 被禁止。

## 识别工具

除了生成之外，还有两个一次性识别工具：`RecognizeImage` 和 `RecognizeAudio`。每个工具都以最小的识别提示词进行一次 LLM 调用，不使用工具、占用最少的上下文：传入一张或多张图片（或音频片段）以及任务，即可获得直接的答案。当代理需要描述图片、读取其中的文字、比较图片或转录/分析音频时使用它们——无需消耗主上下文。转换媒体的操作（编辑、转换、使用其他工具的多步骤分析）需要完整的子代理。

### RecognizeImage

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| `model` | string | yes | — | 具有图像输入和文本输出的精确模型标识符 |
| `provider` | string | no | 无 | 已注册提供商的精确名称；仅当模型标识符唯一时才可省略 |
| `prompt` | string | yes | — | 识别任务；按附加顺序用 `image 1`、`image 2`、… 引用图片；回答语言跟随任务语言 |
| `images` | array | yes | — | 一张或多张图片；顺序决定 `image N` 标签 |

每个元素：`source`（媒体来源，见下表）、可选的 `media_type`（`url` / file-id / `s3_uri` / `gcs_uri` 来源必填；本地文件或内联载荷会自动确定）、可选的 `detail`（`low` / `high` / `auto`）。

本地图片会在共享的视觉策略下进行完整验证和归一化：最长边限制在 2000 像素以内，过度拉长的图片会以原生分辨率切成小块（每块标记为 `Image N (part M of K)`）。URL 图片原样传递给提供商。

### RecognizeAudio

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| `model` | string | yes | — | 具有音频输入和文本输出的精确模型标识符 |
| `provider` | string | no | 无 | 已注册提供商的精确名称 |
| `prompt` | string | yes | — | 识别任务；用 `audio 1`、`audio 2`、… 引用片段 |
| `audio` | array | yes | — | 一个或多个音频片段 |

每个元素：`source`、可选的 `media_type`（`base64` / `url` / file-id 来源必填）。本地文件的格式由扩展名推断（mp3/wav/flac/ogg/aac/aiff/opus/m4a），片段以内联方式发送。

### 识别行为

- 模型解析：需要具有所需输入模态（图像或音频）和文本输出的聊天类模型。不合适的模型会报错并给出合适模型的列表；未知标识符仅在显式指定 `provider` 时原样传递。通过 `Agent{list_models}` 发现候选模型。
- 本地文件通过会话的虚拟文件视图读取：已暂存（尚未同步）的文件版本和暂存删除都会被遵守。
- 结果是类型化的摘要：`answer`（模型的文本）、`usage`、`request_id`，失败时还有带 `code` 的 `error`（`abort`、`rate_limit`、`truncated`、`refusal`、`empty_response`、`incomplete_stream`、…）。任何失败时都会保留部分答案和 usage。
- 识别 token 会计入会话使用量账本。

示例——一次调用处理两张图片：

```json
{
  "model": "<vision-model-id>",
  "prompt": "image 1 和 image 2 之间有什么变化？简要回答。",
  "images": [
    { "source": { "kind": "local_path", "path": "before.png" } },
    { "source": { "kind": "local_path", "path": "after.png" } }
  ]
}
```

转录本地录音：

```json
{
  "model": "<audio-input-model-id>",
  "provider": "gemini",
  "prompt": "这段录音里说了什么？简要回答。",
  "audio": [
    { "source": { "kind": "local_path", "path": "note.wav" } }
  ]
}
```

## 行为与限制

- 模型解析：给定 `provider` 时，未知名称返回错误并附带已注册提供商列表；在任何提供商处都找不到模型时返回错误并附带合适的模型列表；模型出现在多个提供商处时返回错误，要求指定 `provider`；模型种类错误时返回错误并附带该提供商合适的模型列表。
- 媒体来源（`source`）是一个带有 `kind` 字段和每种变体字段的对象：

| kind | 字段 | 值 |
|---|---|---|
| `base64` | `data` | base64 编码的内容 |
| `url` | `url` | 文件地址 |
| `anthropic_file` | `id` | Anthropic 一侧的文件标识符 |
| `openai_file` | `id` | OpenAI 一侧的文件标识符 |
| `google_file` | `uri`, `mime_type` | 对 Google 文件的引用及其类型 |
| `s3_uri` | `uri` | S3 中的对象地址 |
| `gcs_uri` | `uri` | Google Cloud Storage 中的对象地址 |
| `local_path` | `path` | 磁盘上的路径；相对路径从会话工作目录解析 |
| `bytes` | `data`, `mime` | base64 内容及其类型 |
- 声明的图像类型：`png`、`jpeg`、`gif`、`webp`、`heic`、`heif`、`avif`、`bmp`。声明的视频类型：`mp4`、`mov`、`avi`、`mpeg`、`mpg`、`webm`、`flv`、`wmv`、`3gpp`。音频格式：`mp3`、`wav`、`flac`、`ogg`、`aac`、`aiff`、`opus`、`pcm16`、`mp4audio`、`m4a`。
- `save_to` 相对于会话工作目录解析。目录（或无扩展名的路径）会获得形如 `image-1.png`、`video-1.mp4`、`audio-1.wav` 的生成名称；多张图像且显式指定名称基时，会添加索引：`art-1.png`、`art-2.png`。
- 文件写入是原子的（带 fsync），父目录会自动创建。
- 没有 `save_to` 时，图像以内联方式返回。内联图像预算为每次调用 8 个块；超出预算的图像会写入磁盘并以路径返回，绝不会被丢弃。内联图像以最长边不超过 2000 像素的原样传递；过度拉长的图像会在原生分辨率下切成小块，其余部分被压缩到边界以内。
- 声明的图像/视频过滤之外的提供商限制会返回类型化错误。`error.code` 中的提供商错误类别：`unsupported`、`unauthorized`、`rate_limit`、`overloaded`、`context_overflow`、`invalid_request`、`http`、`provider_error`、`agent_loop`。
- 中断请求会返回 `media generation interrupted` 错误；不会产生调用结果，而已成功写入磁盘的文件会保留。
- 如果当前配置中未配置媒体提供商，调用会返回关于媒体生成不可用的错误。

## 示例

创建一张图像并保存到文件：

```json
{
  "operation": "generate_image",
  "model": "<image-model-id>",
  "prompt": "暖光下的静物",
  "save_to": "art.png",
  "size": "1024x1024"
}
```

以内联方式创建两张图像，不保存到磁盘：

```json
{
  "operation": "generate_image",
  "model": "<image-model-id>",
  "prompt": "标志草图",
  "n": 2
}
```

从文本创建视频：

```json
{
  "operation": "generate_video",
  "model": "<video-model-id>",
  "prompt": "镜头在黎明时分缓慢环绕山脊",
  "save_to": "clip.mp4",
  "duration_seconds": 5
}
```

编辑现有视频：

```json
{
  "operation": "generate_video",
  "model": "<video-edit-model-id>",
  "prompt": "将背景替换为夜晚的城市",
  "save_to": "edited.mp4",
  "task": "edit",
  "input_video": { "source": { "kind": "local_path", "path": "input.mp4" }, "media_type": "mp4" }
}
```

创建音乐和合成语音：

```json
{
  "operation": "generate_audio",
  "model": "<music-model-id>",
  "prompt": "平静的氛围主题",
  "save_to": "track.wav",
  "duration_seconds": 30
}
```

```json
{
  "operation": "generate_audio",
  "model": "<tts-model-id>",
  "prompt": "你好，这是一次语音合成检查。",
  "save_to": "speech.wav",
  "voice": "<voice-id>"
}
```
