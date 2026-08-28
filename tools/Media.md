# Media

A media generation tool through configured providers via a single, provider-agnostic interface. Creates images, video, and audio (music or speech synthesis — by model kind).

## Operations

The tool distinguishes operations by the `operation` field. Allowed values: `generate_image`, `generate_video`, `generate_audio`.

Parameters common to all operations:

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| `operation` | string | yes | — | one of the three operations |
| `model` | string | yes | — | the exact media-model identifier, unchanged |
| `provider` | string | no | absent | the exact name of a registered provider; omitted only when the model identifier is unique across providers |
| `prompt` | string | yes | — | the task text; for speech synthesis — the text to speak |
| `save_to` | string | depends on operation | — | the path to the output file or directory; required for video and audio, optional for images |

Media models and their kind are discovered through `Agent{list_models}`: media are the cards with `for_chat: false` and a specified kind (image/video/audio). Provider names are discovered through `Agent{list_providers}`.

### generate_image

Creates images.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| `n` | integer | no | `1` | the number of images, minimum 1 |
| `size` | string | no | `"1024x1024"` | the requested size |
| `format` | string | no | `"png"` | the format: only `png` or `jpeg` |
| `quality` | string | no | absent | a neutral quality hint |
| `style` | string | no | absent | a neutral style hint |
| `negative_prompt` | string | no | absent | content to avoid |
| `seed` | integer | no | absent | a hint for a deterministic result, minimum 0 |

Returns: `saved` (for each file — `path`, `bytes`, `mime`), `usage`, `request_id`, `revised_prompt`. Without `save_to`, the image is returned as an inline base64 block in the context.

### generate_video

Creates and edits video.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| `save_to` | string | yes | — | the path to the output file or directory |
| `task` | string | no | `"auto"` | what to do: `auto`, `text_to_video`, `image_to_video`, `reference_to_video`, `edit` |
| `images` | array | no | empty | input images; one image drives image-to-video, several are style/subject references |
| `input_video` | object | no | absent | the video being edited |
| `continuation_id` | string | no | absent | continue editing a past stateful result |
| `stateful` | boolean | no | `false` | save the new result server-side for continuation |
| `duration_seconds` | number | no | `5.0` | a duration hint; finite and positive |
| `aspect_ratio` | string | no | absent | an aspect-ratio hint |
| `fps` | integer | no | absent | a frames-per-second hint, minimum 1 |
| `seed` | integer | no | absent | a hint for a deterministic result |

Each `images` element and the `input_video` field describe a source: `source` (an object with a kind tag) and a mandatory `media_type` (the declared type).

Returns: `saved` with the saved video and, for a stateful result, `continuation_id` for continuation.

Task-shape limits (checked before calling the provider): `text_to_video` does not accept images, video, or continuation; `image_to_video` requires exactly one image and does not accept video or continuation; `reference_to_video` requires at least one image and does not accept video or continuation; `edit` does not accept images and requires exactly one of `input_video` or `continuation_id`; `auto` accepts at most one input category (images, video, or continuation).

### generate_audio

Creates music or synthesizes speech — the route is chosen by the model kind.

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| `save_to` | string | yes | — | the path to the output file or directory |
| `duration_seconds` | integer | no | absent | a duration hint in whole seconds, minimum 1; music models only |
| `voice` | string | no | absent | the voice identifier; required for speech-synthesis models |
| `speed` | number | no | absent | the speech speed; finite and positive |
| `format` | string | no | `"wav"` for speech | the audio format; for speech-synthesis models |

Returns: `saved`, `usage`, `request_id` and, for music, `description`.

For a music model (`AudioGeneration`) the parameters `voice`, `speed`, and `format` are forbidden — passing them gives an error. For a speech-synthesis model (`TextToSpeech`) the `voice` parameter is required and `duration_seconds` is forbidden.

## Recognition tools

Alongside generation, there are two one-off recognition tools: `RecognizeImage` and `RecognizeAudio`. Each makes a single LLM call with a minimal recognition prompt, no tools, and minimal context: pass one or several images (or audio clips) plus a task, and get a direct answer. Use them when the agent needs to describe a picture, read text from it, compare pictures, or transcribe/analyze audio — without spending the main context. Operations that transform media (editing, converting, multi-step analysis with other tools) require a full subagent.

### RecognizeImage

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| `model` | string | yes | — | the exact model id with image input and text output |
| `provider` | string | no | absent | the exact name of a registered provider; omitted only when the model id is unique across providers |
| `prompt` | string | yes | — | the recognition task; refer to images as `image 1`, `image 2`, … in attachment order; the answer language follows the task language |
| `images` | array | yes | — | one or more images; the order defines the `image N` labels |

Each element: `source` (a media source, see the table below), optional `media_type` (required for `url` / file-id / `s3_uri` / `gcs_uri` sources; a local file or an inline payload is determined automatically), optional `detail` (`low` / `high` / `auto`).

Local images are fully validated and normalized under the shared vision policy: the longest side is bounded by 2000 pixels, an excessively elongated image is cut into tiles at native resolution (each tile is labeled `Image N (part M of K)`). URL images are passed to the provider as-is.

### RecognizeAudio

| parameter | type | required | default | meaning |
|---|---|---|---|---|
| `model` | string | yes | — | the exact model id with audio input and text output |
| `provider` | string | no | absent | the exact name of a registered provider |
| `prompt` | string | yes | — | the recognition task; refer to clips as `audio 1`, `audio 2`, … |
| `audio` | array | yes | — | one or more audio clips |

Each element: `source`, optional `media_type` (required for `base64` / `url` / file-id sources). A local file's format is inferred from its extension (mp3/wav/flac/ogg/aac/aiff/opus/m4a) and the clip is sent inline.

### Recognition behavior

- Model resolution: the model must be a chat-kind model with the required input modality (image or audio) and text output. An unsuitable model gives an error with the list of suitable ones; an unknown id is passed verbatim only under an explicit `provider`. Discover candidates through `Agent{list_models}`.
- A local file is read through the session's virtual file view: a staged (not yet synced) file version and staged deletions are honored.
- The result is a typed summary: `answer` (the model's text), `usage`, `request_id`, and on failure an `error` with a `code` (`abort`, `rate_limit`, `truncated`, `refusal`, `empty_response`, `incomplete_stream`, …). A partial answer and usage are preserved on any failure.
- Recognition tokens are folded into the session usage ledger.

Example — two pictures, one call:

```json
{
  "model": "glm-5.3-flash",
  "prompt": "What changed between image 1 and image 2? Answer briefly.",
  "images": [
    { "source": { "kind": "local_path", "path": "before.png" } },
    { "source": { "kind": "local_path", "path": "after.png" } }
  ]
}
```

Transcribing a local recording:

```json
{
  "model": "gemini-3.6-flash",
  "provider": "gemini",
  "prompt": "Что сказано в этой записи? Кратко.",
  "audio": [
    { "source": { "kind": "local_path", "path": "note.wav" } }
  ]
}
```

## Behavior and limits

- Model resolution: with a given `provider`, an unknown name gives an error with the list of registered providers; a model not found at any provider gives an error with the list of suitable models; a model occurring at several providers gives an error requiring `provider` to be specified; a model of the wrong kind gives an error with the list of suitable models of that provider.
- Media sources (`source`) are an object with a `kind` field and per-variant fields:

| kind | fields | value |
|---|---|---|
| `base64` | `data` | base64-encoded content |
| `url` | `url` | file address |
| `anthropic_file` | `id` | the file identifier on the Anthropic side |
| `openai_file` | `id` | the file identifier on the OpenAI side |
| `google_file` | `uri`, `mime_type` | a reference to a Google file and its type |
| `s3_uri` | `uri` | the object address in S3 |
| `gcs_uri` | `uri` | the object address in Google Cloud Storage |
| `local_path` | `path` | a path on disk; a relative path is resolved from the session working directory |
| `bytes` | `data`, `mime` | base64 content and its type |
- Declared image types: `png`, `jpeg`, `gif`, `webp`, `heic`, `heif`, `avif`, `bmp`. Declared video types: `mp4`, `mov`, `avi`, `mpeg`, `mpg`, `webm`, `flv`, `wmv`, `3gpp`. Audio formats: `mp3`, `wav`, `flac`, `ogg`, `aac`, `aiff`, `opus`, `pcm16`, `mp4audio`, `m4a`.
- `save_to` is resolved relative to the session working directory. A directory (or a path without an extension) receives a generated name of the form `image-1.png`, `video-1.mp4`, `audio-1.wav`; with several images and an explicit name base an index is added: `art-1.png`, `art-2.png`.
- File writing is atomic (with fsync), parent directories are created automatically.
- Without `save_to`, the image is returned inline. The inline-image budget is 8 blocks per call; images beyond the budget are written to disk and returned as paths, never discarded. An inline image is passed as is at a maximum side of no more than 2000 pixels; an excessively elongated one is cut into tiles at native resolution, the rest is squeezed under the boundary.
- A field the provider does not support returns an `unsupported` error — nothing is silently dropped. Provider error classes in `error.code`: `unsupported`, `unauthorized`, `rate_limit`, `overloaded`, `context_overflow`, `invalid_request`, `http`, `provider_error`, `agent_loop`.
- Interrupting a request returns the `media generation interrupted` error; no call result is produced, and a file that managed to be written to disk remains.
- If media providers are not configured in the current configuration, the call returns an error about media generation being unavailable.

## Examples

Create one image and save it to a file:

```json
{
  "operation": "generate_image",
  "model": "gemini-3.1-flash-image",
  "prompt": "still life in warm light",
  "save_to": "art.png",
  "size": "1024x1024"
}
```

Create two images inline, without saving to disk:

```json
{
  "operation": "generate_image",
  "model": "gpt-image-2",
  "prompt": "logo sketch",
  "n": 2
}
```

Create a video from text:

```json
{
  "operation": "generate_video",
  "model": "veo-3.1-generate-preview",
  "prompt": "the camera slowly orbits a mountain ridge at dawn",
  "save_to": "clip.mp4",
  "duration_seconds": 5
}
```

Edit an existing video:

```json
{
  "operation": "generate_video",
  "model": "sora-2",
  "prompt": "replace the background with a night city",
  "save_to": "edited.mp4",
  "task": "edit",
  "input_video": { "source": { "kind": "local_path", "path": "input.mp4" }, "media_type": "mp4" }
}
```

Create music and synthesize speech:

```json
{
  "operation": "generate_audio",
  "model": "lyria-3-clip-preview",
  "prompt": "a calm ambient theme",
  "save_to": "track.wav",
  "duration_seconds": 30
}
```

```json
{
  "operation": "generate_audio",
  "model": "gpt-4o-mini-tts",
  "prompt": "Hello, this is a speech-synthesis check.",
  "save_to": "speech.wav",
  "voice": "alloy"
}
```
