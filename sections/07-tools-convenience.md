# Typed tools (full product surface)

As of MCP v2.0.0 the server exposes **110 tools** (108 REST proxies + `wait_for_generation` + `api_v1_request`) covering every integrator-product workflow. Integrator overview: **`docs/public-api/30-mcp.md`**. Setup entry point: **`docs/MCP.md`**.

| Group | Tools |
|-------|-------|
| Meta & account | `get_me`, `get_pricing_generation`, `confirm_adult`, `get_notifications`, `notifications_mark_read`, `notifications_mark_all_read`, `get_notification_preferences`, `set_notification_preferences`, `get_my_flags`, `get_plans`, `api_v1_request` |
| Generations | `list_generations`, `get_generation`, **`wait_for_generation`**, `generations_batch_delete`, `generations_monthly_stats` |
| Models & wizard | `list_models`, `get_model`, `create_model`, `delete_model`, `models_generate_reference`, `models_generate_poses`, `models_status`, `wizard_*` (5) |
| SFW generate | `generate_image_identity`, `generate_recreate`, `generate_free`, `generate_preset_recreate`, `enhance_prompt`, `generate_motion_video`, `generate_video_motion`, `generate_video_directly`, `generate_face_swap_video`, `generate_image_faceswap`, `generate_complete_recreation`, `describe_target`, `extract_frames`, `generate_advanced`, `creator_studio_*` (5) |
| NSFW | `nsfw_*` (18) + `sexting_*` (6) + `nsfw_video_*` (4) |
| ModelClone-X | `mcx_config`, `mcx_generate`, `mcx_status` |
| img2img | `img2img_describe`, `img2img_describe_status`, `img2img_generate`, `img2img_status` |
| GPT-X | `gptx_conversations`, `gptx_send` |
| Tools | `upscale_image`, `synthid_remove`, `video_repurpose_*`, `reformatter_convert`, `reformatter_status` |
| Flow Studio | `flows_*` (10) |
| Gallery & profile | `gallery_*` (7), `set_username` |
| Avatars | `avatars_*` (5) |

All tool results are JSON in a single `text` content block.

---

## Meta and account tools

### `get_me`

| | |
|---|---|
| **Tool name** | `get_me` |
| **REST mapping** | `GET /me` |
| **Description** | Profile, credit balance, and subscription tier. Call **first in every session** to confirm the API key works and you have enough credits. |

#### Input schema

No parameters.

#### Example MCP tool call

```json
{
  "name": "get_me",
  "arguments": {}
}
```

#### Example response

```json
{
  "httpStatus": 200,
  "ok": true,
  "method": "GET",
  "path": "/me",
  "url": "https://modelclone.app/api/v1/me",
  "body": {
    "success": true,
    "apiVersion": 1,
    "data": {
      "user": {
        "id": "ckq1x2y3z0000abcd1234efgh",
        "email": "dev@example.com",
        "name": "Dev Account",
        "credits": 120,
        "subscriptionCredits": 500,
        "purchasedCredits": 250,
        "totalCredits": 870,
        "subscriptionTier": "business",
        "subscriptionStatus": "active",
        "isVerified": true,
        "onboardingCompleted": true
      },
      "authVia": "api_key"
    }
  }
}
```

#### Poll / completion flow

Synchronous — no polling. Read `body.data.user.totalCredits` before submitting generations.

#### Common errors

| HTTP | When | What to do |
|---|---|---|
| **401** | Missing or invalid API key | Fix MCP auth header |
| **404** | Account no longer exists | Contact support |
| **429** | Too many requests | Retry with backoff |

#### Credit cost notes

**Free** — no credits charged. Use with **`get_pricing_generation`** before quoting costs to users.

---

### `get_pricing_generation`

| | |
|---|---|
| **Tool name** | `get_pricing_generation` |
| **REST mapping** | `GET /pricing/generation` |
| **Description** | Authoritative live credit costs per generation type and option. Prices are server-configurable — never hard-code costs in agents or integrations. |

#### Input schema

No parameters.

#### Example MCP tool call

```json
{
  "name": "get_pricing_generation",
  "arguments": {}
}
```

#### Example response

```json
{
  "httpStatus": 200,
  "ok": true,
  "method": "GET",
  "path": "/pricing/generation",
  "url": "https://modelclone.app/api/v1/pricing/generation",
  "body": {
    "success": true,
    "pricing": {
      "recreateImage": 10,
      "freePromptImage": 5,
      "motionVideoPerSecond": 9.5,
      "nsfwImage": 6
    },
    "updatedAt": "2026-07-01T12:00:00.000Z",
    "contract": {
      "keys": ["recreateImage", "freePromptImage"],
      "defaults": { "recreateImage": 10 }
    }
  }
}
```

The `contract` object documents what each key in `pricing` means. Re-fetch periodically if you display prices to users.

#### Poll / completion flow

Synchronous — no polling.

#### Common errors

| HTTP | When | What to do |
|---|---|---|
| **401** | Missing or invalid API key | Fix MCP auth header |
| **429** | Too many requests | Retry with backoff |
| **500** | Server error loading pricing | Retry with backoff |

#### Credit cost notes

**Free** — returns costs; does not charge credits. This is the **authoritative** source for all generation pricing referenced by other tools.

---

## Generation history and polling tools

Every async submit tool returns a generation id. Use these three tools to list history, poll once, or poll server-side until completion.

### `list_generations`

| | |
|---|---|
| **Tool name** | `list_generations` |
| **REST mapping** | `GET /generations` |
| **Description** | Paginated generation history across all types, newest first. |

#### Input schema

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `type` | string | No | omitted | Filter by generation type (e.g. `image`, `video`, `nsfw`, `modelclone-x`, `creator-studio`) |
| `modelId` | string (UUID) | No | omitted | Only generations for one saved model |
| `status` | string | No | omitted | `processing`, `completed`, or `failed`; comma-separate for multiple (e.g. `completed,failed`) |
| `limit` | integer | No | `50` (REST default) | Page size, min 1, max 200 |
| `offset` | integer | No | `0` | Rows to skip, min 0 |
| `includeTotal` | boolean | No | `false` | When `true`, includes `pagination.total` row count |

#### Example MCP tool call

```json
{
  "name": "list_generations",
  "arguments": {
    "status": "completed",
    "limit": 20,
    "offset": 0,
    "includeTotal": true
  }
}
```

#### Example response

```json
{
  "httpStatus": 200,
  "ok": true,
  "method": "GET",
  "path": "/generations",
  "url": "https://modelclone.app/api/v1/generations?status=completed&limit=20&offset=0&includeTotal=true",
  "body": {
    "success": true,
    "generations": [
      {
        "id": "gen_abc",
        "modelId": "model_123",
        "type": "image",
        "status": "completed",
        "prompt": "golden hour portrait on a rooftop",
        "outputUrl": "https://cdn.modelclone.app/generations/gen_abc.png",
        "errorMessage": null,
        "creditsCost": 6,
        "creditsRefunded": false,
        "createdAt": "2026-07-07T20:00:00.000Z",
        "completedAt": "2026-07-07T20:00:41.000Z"
      }
    ],
    "pagination": { "total": 1342, "limit": 20, "offset": 0 },
    "retention": { "maxCompletedPerModel": 500 }
  }
}
```

#### Poll / completion flow

Synchronous listing — for a single in-flight job, use **`get_generation`** or **`wait_for_generation`**.

#### Common errors

| HTTP | When | What to do |
|---|---|---|
| **401** | Missing or invalid API key | Fix MCP auth header |
| **429** | History list rate limit (90/min) | Reduce request frequency |

#### Credit cost notes

**Free** — no credits charged.

---

### `get_generation`

| | |
|---|---|
| **Tool name** | `get_generation` |
| **REST mapping** | `GET /generations/:id` |
| **Description** | Single generation status and `outputUrl`. Canonical one-shot poll target after any submit. |

#### Input schema

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `generationId` | string | **Yes** | — | Generation id returned by any submit tool |

#### Example MCP tool call

```json
{
  "name": "get_generation",
  "arguments": {
    "generationId": "gen_abc"
  }
}
```

#### Example response (processing)

```json
{
  "httpStatus": 200,
  "ok": true,
  "method": "GET",
  "path": "/generations/gen_abc",
  "url": "https://modelclone.app/api/v1/generations/gen_abc",
  "body": {
    "success": true,
    "generation": {
      "id": "gen_abc",
      "type": "image",
      "status": "processing",
      "outputUrl": null,
      "errorMessage": null,
      "creditsCost": 10,
      "createdAt": "2026-07-07T20:00:00.000Z"
    }
  }
}
```

#### Example response (completed)

When `body.generation.status` is `"completed"`, read `body.generation.outputUrl`.

#### Poll / completion flow

1. Submit any generation tool → extract `generationId`.
2. Call **`get_generation`** every 2–5 seconds until `status` is `completed` or `failed`.
3. Prefer **`wait_for_generation`** instead of client-side loops.

Status polls are rate-limited (120/min per account).

#### Common errors

| HTTP | When | What to do |
|---|---|---|
| **401** | Missing or invalid API key | Fix MCP auth header |
| **404** | Id not found or belongs to another account | Verify id from submit response |
| **429** | Poll rate limit exceeded | Use **`wait_for_generation`** or increase interval |

#### Credit cost notes

**Free** — polling does not charge credits. Generation cost was charged at submit time; see **`get_pricing_generation`** for price list.

---

### `wait_for_generation`

| | |
|---|---|
| **Tool name** | `wait_for_generation` |
| **REST mapping** | Server-side loop over `GET /generations/:id` |
| **Description** | Polls until the generation reaches a terminal status (`completed`/`failed`) or the timeout elapses. **Prefer this after any submit** instead of manual `get_generation` loops. |

#### Input schema

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `generationId` | string | **Yes** | — | Generation id from any submit tool |
| `timeoutSec` | integer | No | `120` | Max wait time in seconds, min 5, max 570 |
| `intervalSec` | integer | No | `5` | Seconds between polls, min 2, max 30 |

#### Example MCP tool call

```json
{
  "name": "wait_for_generation",
  "arguments": {
    "generationId": "gen_abc",
    "timeoutSec": 180,
    "intervalSec": 5
  }
}
```

#### Example response (completed)

```json
{
  "done": true,
  "status": "completed",
  "generation": {
    "id": "gen_abc",
    "type": "image",
    "status": "completed",
    "outputUrl": "https://cdn.modelclone.app/generations/gen_abc.png",
    "errorMessage": null,
    "creditsCost": 10,
    "creditsRefunded": false,
    "completedAt": "2026-07-07T20:00:41.000Z"
  }
}
```

#### Example response (failed)

```json
{
  "done": true,
  "status": "failed",
  "generation": {
    "id": "gen_abc",
    "status": "failed",
    "outputUrl": null,
    "errorMessage": "Generation failed — credits refunded",
    "creditsRefunded": true
  }
}
```

#### Example response (timeout — still processing)

```json
{
  "done": false,
  "timedOut": true,
  "hint": "Still processing — call wait_for_generation or get_generation again.",
  "last": {
    "success": true,
    "generation": {
      "id": "gen_abc",
      "status": "processing"
    }
  }
}
```

#### Example response (not found)

```json
{
  "done": false,
  "error": "Generation not found",
  "last": {
    "httpStatus": 404,
    "ok": false,
    "body": { "success": false, "message": "Generation not found" }
  }
}
```

#### Poll / completion flow

1. Submit → get `generationId`.
2. Call **`wait_for_generation`** once with an appropriate `timeoutSec` (videos may need 300+).
3. If `timedOut: true`, call again with the same id to keep waiting.
4. When `done: true` and `status: "completed"`, download `generation.outputUrl`.

Unlike other typed tools, this tool's response is **not** wrapped in `httpStatus`/`body` — it returns the poll result directly.

#### Common errors

| Shape | When | What to do |
|---|---|---|
| `{ "done": false, "error": "Generation not found" }` | **404** on poll | Verify generation id |
| `{ "done": false, "timedOut": true, … }` | Job still running after `timeoutSec` | Call again or increase timeout |
| `{ "ok": false, "error": "…" }` | Network or unexpected failure | Retry; check MCP connectivity |
| Underlying **401** / **429** on poll requests | Auth or rate limit during wait | Fix key; reduce poll frequency |

#### Credit cost notes

**Free** — waiting does not charge credits. Submit cost is in `generation.creditsCost`; compare against **`get_pricing_generation`** before submit.

---

### Terminal generation states

| status | Meaning |
|--------|---------|
| `completed` | `outputUrl` set — download the asset |
| `failed` | `errorMessage` set; credits refunded when applicable (`creditsRefunded: true`) |

Treat `completed` and `failed` as terminal. Stuck jobs are auto-failed by a server watchdog.

---

## Models and wizard tools

REST reference: [`10-models-and-wizard.md`](../../public-api/10-models-and-wizard.md). Model creation is **asynchronous** in the final pose step: pose/upload endpoints return **202** with `model.status: "processing"`. Poll with **`models_status`** every 3–5s until `ready` or `failed`. Confirm capacity with **`list_models`** (`canCreateMore`) before creating.

Live credit costs: **`get_pricing_generation`**. Defaults below — treat the pricing response as authoritative.

| Tool | Pricing key | Default credits |
|------|-------------|-----------------|
| `models_generate_reference` | `modelStep1Reference` | 150 |
| `models_generate_poses` | `modelStep2Poses` | 750 |
| `wizard_custom_reference` (`regenerate: true`) | `wizardCustomRegenerate` | 10 |
| All other model/wizard tools in this section | — | 0 |

### Creation paths

| Path | Tools (in order) | Poll |
|------|------------------|------|
| Wizard — niche | `wizard_look_variants` → `wizard_preview_images` → `wizard_finalize_poses` | `models_status` |
| Wizard — custom text | `wizard_custom_reference` → `wizard_finalize_poses` | `models_status` |
| Wizard — upload | `wizard_upload_save` | `models_status` |
| Classic two-step | `models_generate_reference` → `models_generate_poses` | `models_status` |
| Direct URLs | `create_model` | — (sync) |

> **NSFW eligibility:** models from user uploads (`create_model`, `wizard_upload_save`) are not AI-generated and **cannot** use NSFW features.

### `list_models`

| | |
|---|---|
| **Tool name** | `list_models` |
| **REST mapping** | `GET /models` |
| **Description** | Saved AI models for this account plus plan limit metadata. |

#### Input schema

No parameters.

#### Example MCP tool call

```json
{ "name": "list_models", "arguments": {} }
```

#### Poll / completion flow

Synchronous. Read `body.models[]`, `body.count`, `body.limit`, and `body.canCreateMore` before starting creation.

#### Common errors

| HTTP | When | What to do |
|---|---|---|
| **401** | Invalid API key | Fix MCP auth |
| **429** | Rate limit | Back off and retry |

#### Credit cost notes

**Free** — no credits charged.

---

### `get_model`

| | |
|---|---|
| **Tool name** | `get_model` |
| **REST mapping** | `GET /models/:id` |
| **Description** | One saved model — reference photos, `status`, appearance, voice metadata. |

#### Input schema

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `modelId` | string (UUID) | **Yes** | Model id from `list_models` or creation response |

#### Example MCP tool call

```json
{ "name": "get_model", "arguments": { "modelId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890" } }
```

#### Poll / completion flow

Synchronous. When `body.model.status` is `processing`, poll with **`models_status`** instead.

#### Common errors

| HTTP | When | What to do |
|---|---|---|
| **401** | Invalid API key | Fix MCP auth |
| **404** | Model not found | Verify id |
| **429** | Rate limit | Back off |

#### Credit cost notes

**Free**.

---

### `create_model`

| | |
|---|---|
| **Tool name** | `create_model` |
| **REST mapping** | `POST /models` |
| **Description** | Save a model from three hosted photo URLs (upload via `POST /upload` first). Synchronous — no generation job. |

#### Input schema

| Field (in `body`) | Type | Required | Description |
|-------------------|------|----------|-------------|
| `name` | string | **Yes** | Display name (unique per account) |
| `photo1Url`, `photo2Url`, `photo3Url` | string (URL) | **Yes** | Selfie, portrait, full-body references |
| `savedAppearance` | object | No | Appearance attribute map |

#### Example MCP tool call

```json
{
  "name": "create_model",
  "arguments": {
    "body": {
      "name": "Jordan",
      "photo1Url": "https://cdn.example.com/selfie.jpg",
      "photo2Url": "https://cdn.example.com/portrait.jpg",
      "photo3Url": "https://cdn.example.com/fullbody.jpg"
    }
  }
}
```

#### Poll / completion flow

Synchronous — model is `ready` immediately. Use for direct URL saves; prefer wizard/classic flows for AI-generated identities.

#### Common errors

| HTTP | When | What to do |
|---|---|---|
| **400** | Missing fields or invalid URLs | Fix body; upload photos first |
| **409** | Name collision or plan limit | Rename or delete a model |
| **429** | Rate limit | Back off |

#### Credit cost notes

**Free**. User-uploaded models are **not** NSFW-eligible.

---

### `delete_model`

| | |
|---|---|
| **Tool name** | `delete_model` |
| **REST mapping** | `DELETE /models/:id` |
| **Description** | Delete a model and related assets. |

#### Input schema

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `modelId` | string (UUID) | **Yes** | Model to delete |

#### Example MCP tool call

```json
{ "name": "delete_model", "arguments": { "modelId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890" } }
```

#### Poll / completion flow

Synchronous.

#### Common errors

| HTTP | When | What to do |
|---|---|---|
| **404** | Model not found | Verify id |
| **409** | Generations still in progress for this model | Wait for jobs to finish |

#### Credit cost notes

**Free**.

---

### `models_generate_reference`

| | |
|---|---|
| **Tool name** | `models_generate_reference` |
| **REST mapping** | `POST /models/generate-reference` |
| **Description** | Classic phase 1 — generate a reference face from appearance chips. **Synchronous** (200). |

#### Input schema

| Field (in `body`) | Type | Required | Description |
|-------------------|------|----------|-------------|
| `gender`, `age` | string/number | **Yes** | Age 18–120 |
| Appearance chip fields | string | **Yes** | Ethnicity, hair, skin, eyes, face, body — see REST doc |
| `referencePrompt` | string | No | Creative hint |
| `regenerate` | boolean | No | Force a new reference |

#### Example MCP tool call

```json
{
  "name": "models_generate_reference",
  "arguments": {
    "body": {
      "gender": "female",
      "age": "25",
      "referencePrompt": "soft natural makeup",
      "ethnicity": "Latina",
      "hairColor": "Dark Brown",
      "hairType": "Wavy",
      "skinTone": "Medium",
      "eyeColor": "Brown",
      "eyeShape": "Almond",
      "faceShape": "Oval",
      "noseShape": "Straight",
      "lipSize": "Medium",
      "bodyType": "Athletic",
      "height": "Average",
      "breastSize": "Medium",
      "buttSize": "Medium",
      "waist": "Slim",
      "hips": "Medium",
      "tattoos": "None"
    }
  }
}
```

#### Poll / completion flow

Synchronous — read `body.referenceUrl`. Missing appearance categories → **400** with `missing` array. Feed `referenceUrl` into **`models_generate_poses`**.

#### Credit cost notes

Pricing key `modelStep1Reference` (default **150** credits). Confirm via **`get_pricing_generation`**.

---

### `models_generate_poses`

| | |
|---|---|
| **Tool name** | `models_generate_poses` |
| **REST mapping** | `POST /models/generate-poses` |
| **Description** | Classic phase 2 — create model row + async 3-pose reference set. |

#### Input schema

| Field (in `body`) | Type | Required | Description |
|-------------------|------|----------|-------------|
| `name` | string | **Yes** | Model display name |
| `referenceUrl` | string (URL) | **Yes** | From `models_generate_reference` |
| `gender`, `age`, appearance fields | — | Recommended | Passed through to generation |
| `posesPrompt`, `outfitType`, `poseStyle` | string | No | Pose styling hints |

#### Example MCP tool call

```json
{
  "name": "models_generate_poses",
  "arguments": {
    "body": {
      "name": "Riley",
      "referenceUrl": "https://cdn.modelclone.app/references/ref-4f2a.jpg",
      "gender": "female",
      "age": "25"
    }
  }
}
```

#### Poll / completion flow

**Async** (202). Poll **`models_status`** with `body.model.id` every 3–5s until `status` is `ready` or `failed`.

#### Credit cost notes

Pricing key `modelStep2Poses` (default **750** credits).

---

### `models_status`

| | |
|---|---|
| **Tool name** | `models_status` |
| **REST mapping** | `GET /models/status/:id` |
| **Description** | Model creation job status — canonical poll target after async pose/upload/finalize steps. |

#### Input schema

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | string (UUID) | **Yes** | Model id from async create response |

#### Example MCP tool call

```json
{ "name": "models_status", "arguments": { "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890" } }
```

#### Poll / completion flow

Poll every 3–5s. Terminal values: `ready` (use model id in generation tools) or `failed` (read `message`).

#### Credit cost notes

**Free** — polling only.

---

### `wizard_look_variants`

| | |
|---|---|
| **Tool name** | `wizard_look_variants` |
| **REST mapping** | `POST /wizard/niche/look-variants` |
| **Description** | Wizard step 1 — four niche look palettes for a target persona. **Free, synchronous.** |

#### Input schema

| Field (in `body`) | Type | Required | Description |
|-------------------|------|----------|-------------|
| `gender`, `age`, `nicheName` | — | **Yes** | Target niche (e.g. Fitness) |
| `nicheId`, `nicheBio` | string | No | Optional niche metadata |
| `ethnicity`, `hairColor`, `eyeColor`, `bodyType` | string | No | Seed constraints |

#### Example MCP tool call

```json
{ "name": "wizard_look_variants", "arguments": { "body": { "gender": "female", "age": 24, "nicheName": "Fitness" } } }
```

#### Poll / completion flow

Synchronous — pass `body.variants[]` to **`wizard_preview_images`**.

#### Credit cost notes

**Free**.

---

### `wizard_preview_images`

| | |
|---|---|
| **Tool name** | `wizard_preview_images` |
| **REST mapping** | `POST /wizard/niche/preview-images` |
| **Description** | Wizard step 2 — render preview faces for chosen variants. **Free, synchronous.** |

#### Input schema

| Field (in `body`) | Type | Required | Description |
|-------------------|------|----------|-------------|
| `gender`, `age`, `variants` | — | **Yes** | `variants` from `wizard_look_variants` |
| `nicheId`, `nicheName`, `nicheBio` | string | No | Optional context |

#### Example MCP tool call

```json
{
  "name": "wizard_preview_images",
  "arguments": {
    "body": {
      "gender": "female",
      "age": 24,
      "variants": [{ "label": "Soft", "looks": { "gender": "female", "ethnicity": "Latina" } }]
    }
  }
}
```

#### Poll / completion flow

Synchronous — pick a `referenceUrl` from previews → **`wizard_finalize_poses`**.

#### Credit cost notes

**Free**.

---

### `wizard_custom_reference`

| | |
|---|---|
| **Tool name** | `wizard_custom_reference` |
| **REST mapping** | `POST /wizard/custom/reference` |
| **Description** | Wizard — custom text reference face. First call free; `regenerate: true` is charged. **Synchronous.** |

#### Input schema

| Field (in `body`) | Type | Required | Description |
|-------------------|------|----------|-------------|
| `gender`, `age` | — | **Yes** | |
| `referencePrompt` | string | No | Appearance description |
| `regenerate` | boolean | No | `true` triggers paid regeneration |
| appearance fields / `savedAppearance` / `look` | — | No | Structured looks |

#### Example MCP tool call

```json
{
  "name": "wizard_custom_reference",
  "arguments": {
    "body": {
      "gender": "female",
      "age": 26,
      "referencePrompt": "minimalist fashion creator, platinum blonde bob"
    }
  }
}
```

#### Poll / completion flow

Synchronous — use returned `referenceUrl` in **`wizard_finalize_poses`**.

#### Credit cost notes

First call **free**. `regenerate: true` uses `wizardCustomRegenerate` (default **10** credits).

---

### `wizard_upload_save`

| | |
|---|---|
| **Tool name** | `wizard_upload_save` |
| **REST mapping** | `POST /wizard/upload/save` |
| **Description** | Wizard — create a model from three uploaded photos (auto-normalized to studio references). **Free, async (202).** |

#### Input schema

| Field (in `body`) | Type | Required | Description |
|-------------------|------|----------|-------------|
| `name`, `photo1Url`, `photo2Url`, `photo3Url` | — | **Yes** | Hosted photo URLs |
| `age`, `gender`, `savedAppearance` | — | No | Optional metadata |

#### Example MCP tool call

```json
{
  "name": "wizard_upload_save",
  "arguments": {
    "body": {
      "name": "UploadClone",
      "photo1Url": "https://cdn.example.com/selfie.jpg",
      "photo2Url": "https://cdn.example.com/portrait.jpg",
      "photo3Url": "https://cdn.example.com/fullbody.jpg"
    }
  }
}
```

#### Poll / completion flow

**Async** — poll **`models_status`** with returned model id.

#### Credit cost notes

**Free**. Upload path models are **not** NSFW-eligible.

---

### `wizard_finalize_poses`

| | |
|---|---|
| **Tool name** | `wizard_finalize_poses` |
| **REST mapping** | `POST /wizard/finalize-poses` |
| **Description** | Wizard final step — persist model + async 3-pose set. **Free on wizard path.** Same body shape as `models_generate_poses`. |

#### Input schema

Same as **`models_generate_poses`** — `name`, `referenceUrl`, appearance fields, optional pose hints.

#### Example MCP tool call

```json
{
  "name": "wizard_finalize_poses",
  "arguments": {
    "body": {
      "name": "FitCreator",
      "referenceUrl": "https://cdn.modelclone.app/references/prev-1.jpg",
      "gender": "female",
      "age": 24,
      "ethnicity": "Latina",
      "hairColor": "Dark Brown"
    }
  }
}
```

#### Poll / completion flow

**Async** (202) → **`models_status`** until `ready`.

#### Credit cost notes

**Free** on the wizard path (unlike paid `models_generate_poses` on the classic path).

---

## SFW image & video generation

Cross-references: [`11-image-generation.md`](../../public-api/11-image-generation.md), [`12-video-generation.md`](../../public-api/12-video-generation.md), [`13-creator-studio.md`](../../public-api/13-creator-studio.md).

| Tool | REST | Async | Poll |
|------|------|-------|------|
| `generate_image_identity` | `POST /generate/image-identity` | Yes | `wait_for_generation` |
| `generate_recreate` | `POST /generate/recreate` | Yes | `wait_for_generation` |
| `generate_free` | `POST /generate/free` | Yes | `wait_for_generation` |
| `enhance_prompt` | `POST /generate/enhance-prompt` | **Sync** | — |
| `generate_motion_video` | `POST /generate/motion-video` | Yes | `wait_for_generation` |
| `creator_studio_image` | `POST /generate/creator-studio` | Yes | `wait_for_generation` |
| `creator_studio_video` | `POST /generate/creator-studio/video` | Yes | `wait_for_generation` |

**Webhooks:** `integrationCallbackUrl` + optional `integratorWebhookSecret` in `options` (recreate/free/enhance/image-identity) or `body` (motion/creator studio). **Credits:** `get_pricing_generation`.

### `generate_image_identity`

`POST /generate/image-identity` — legacy target-photo identity recreation. Prefer `generate_recreate` for new work.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `modelId` | string | yes | Saved model UUID (3 reference photos) |
| `targetImage` | string | yes | Public HTTPS URL of the photo to edit |
| `quantity` | number | no | 1–10 (default 1) |
| `prompt` | string | no | Extra creative guidance |
| `clothesMode` | string | no | Outfit handling (e.g. `reference`) |
| `options` | object | no | Webhook fields and additional REST keys |

Credits: `imageIdentity` (**10**/image). See [`11-image-generation.md`](../../public-api/11-image-generation.md#post-generateimage-identity).

### `generate_recreate`

`POST /generate/recreate` — core identity recreation.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `modelId` | string | yes | Saved model UUID (3 reference photos) |
| `sourceImageUrl` | string | yes | Public HTTPS URL of the photo to recreate |
| `count` | number | no | 1–10 (default 1) |
| `outfitMode` | string | no | `model` (default), `source`, or `external` |
| `externalOutfitImageUrl` | string | when `outfitMode: external` | Outfit reference image |
| `extraGuidance` | string | no | Extra creative direction (max 400 chars) |
| `genModel` | string | no | `wan-2.7-image` (default) or `nano-banana-pro` |
| `options` | object | no | Webhook fields and any additional REST body keys |

Credits: `recreateImage` (**10**/image) or `recreateImageNanoBanana` (**16**/image).

```json
{
  "modelId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "sourceImageUrl": "https://cdn.example.com/inspo/pose.jpg",
  "outfitMode": "model",
  "count": 1
}
```

### `generate_free`

`POST /generate/free` — prompt-driven image with model identity locked by 3 reference photos.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `modelId` | string | yes | Saved model UUID |
| `prompt` | string | yes | Scene/idea description |
| `genModel` | string | no | `nano-banana-pro` (default), `wan-2.7-image`, `seedream-4.5-edit` |
| `refImageUrls` | string[] | no | Extra reference images |
| `aspectRatio` / `resolution` | string | no | Engine-dependent — see [`11-image-generation.md`](../../public-api/11-image-generation.md) |
| `enhance` | boolean | no | Default `true` (+`enhancePromptDefault` **1** credit once per request) |
| `count` | number | no | 1–8 (default 1) |
| `options` | object | no | Webhook fields and additional REST keys |

### `enhance_prompt`

Sync — returns `body.enhancedPrompt`. `options`: `mode`, `genModel`, `modelLooks.gender`. Credits: `enhancePromptDefault` (**1**).

### `generate_motion_video`

`POST /generate/motion-video` — SFW motion-control video.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `imageUrl` | string | yes | Still image to animate |
| `videoUrl` | string | yes | Driving reference clip (public HTTPS) |
| `prompt` | string | no | Motion description |
| `duration` | number | no | Output seconds 2–15 (default 5) |
| `skipSeconds` | number | no | Skip from start of driving clip |
| `seed` | number | no | Fixed seed |
| `options` | object | no | Webhook fields, trim fields, etc. |

Credits: `motionXPerSec` (**9.5**/s) × duration. Poll `wait_for_generation` on `body.generationId`.

### `creator_studio_image` / `creator_studio_video`

Full REST body — model/resolution matrix in [`13-creator-studio.md`](../../public-api/13-creator-studio.md). Video: `cinematic` bucket. Extend/4K via `api_v1_request`.

---

## NSFW access (all NSFW tools)

Same gate as REST ([NSFW Studio](../../public-api/14-nsfw.md#access-requirements)):

| Layer | Requirement | Typical `403` `code` |
|-------|-------------|------------------------|
| Account | First purchase + 18+ confirmation | `NSFW_NEEDS_PURCHASE`, `NSFW_NEEDS_AGE_CONFIRMATION` |
| Classic LoRA pipeline (`nsfw_generate`, `nsfw_train_lora`) | AI-generated model, 18+ persona, trained LoRA (`nsfwUnlocked`) | — (message: train LoRA first) |
| v2 stills + NSFW video sessions | NSFW-verified model + all three NSFW reference photos | `NSFW_NOT_VERIFIED`, `NSFW_REFS_INCOMPLETE` |

Age confirmation is a one-time dashboard action — not grantable via API key.

**Webhooks:** any generation-creating POST body (including `nsfw_video_create_session` and `nsfw_video_session_action` when it creates a generation) may include `integrationCallbackUrl` and `integratorWebhookSecret`. See [Webhooks](../../public-api/05-webhooks.md).

**Sanitization (API-key / MCP responses):** provider fields are stripped globally. NSFW Video pipeline generation rows additionally return `prompt: null` and omit `engine` — the composition is proprietary. Session objects are not sanitized beyond normal provider stripping. Details in [NSFW Video](../../public-api/15-nsfw-video.md#notes).

---

## NSFW studio — typed tools

REST reference: [NSFW Studio](../../public-api/14-nsfw.md). Costs: `get_pricing_generation`.

### `nsfw_generate`

`POST /nsfw/generate` — classic NSFW image with a trained LoRA.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `modelId` | string | yes | Model UUID |
| `prompt` | string | yes | Scene prompt — include the model's LoRA trigger word |
| `quantity` | number | no | `1` (default) or `2` |
| `attributes` | string | no | Comma-separated appearance/scene chips |
| `sceneDescription` | string | no | Short scene note |
| `options` | object | no | `loraStrength`, `quickFlow`, `resolution`, `postProcessing`, webhook fields |

**Response:** `{ success, generation, generations[], creditsUsed, creditsRemaining, imageQuantity }` — poll `wait_for_generation` on each `generation.id`.

**Completion:** `wait_for_generation(generationId)` — typically 1–3 minutes.

### `nsfw_generate_video`

`POST /nsfw/generate-video` — animate a still image into a short NSFW video (5 s or 8 s).

| Body field | Type | Required | Description |
|------------|------|----------|-------------|
| `modelId` | string | yes | NSFW-eligible model |
| `imageUrl` | string | yes | Source image URL (typically a completed generation `outputUrl`) |
| `prompt` | string | no | Motion description; natural-motion default if omitted |
| `duration` | number | no | `5` (default) or `8` |
| `integrationCallbackUrl` / `integratorWebhookSecret` | string | no | Webhook fields |

**Response:** `{ success, generationId, creditsUsed, creditsRemaining, duration }` → `wait_for_generation(generationId)`.

### `nsfw_train_lora`

`POST /nsfw/train-lora` — start LoRA identity training (long-running).

| Body field | Type | Required | Description |
|------------|------|----------|-------------|
| `modelId` | string | yes | AI-generated model |
| `loraId` | string | no | Target LoRA row; omit for legacy per-model path |

**Response:** `{ success, deferred: true, triggerWord, creditsUsed, message }` (HTTP 202 semantics).

**Poll:** `nsfw_training_status` every ~60 s until `status: "ready"` or `failed`. Training prep/training takes ~1–6 h by tier.

> LoRA setup (create LoRA, upload/generate training images, set active) is **not** exposed as typed MCP tools — use `api_v1_request` with paths from `modelclone://v1/route-catalog` and shapes from `modelclone://v1/openapi`. See [LoRA training](../../public-api/14-nsfw.md#lora-training).

### `nsfw_training_status`

`GET /nsfw/training-status/:modelId`

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `modelId` | string | yes | Model UUID |
| `loraId` | string | no | Specific LoRA row id (REST query `?loraId=`) |

**Response:** `{ success, status, loraUrl?, triggerWord?, nsfwUnlocked, loraId?, firstLoraBonus?, preprocessing? }` where `status` is `none` \| `awaiting_images` \| `images_ready` \| `training` \| `ready` \| `failed`.

### `nsfw_v2_preset`

`POST /nsfw-v2/presets` — fast preset still from three NSFW reference photos (no LoRA).

| Body field | Type | Required | Description |
|------------|------|----------|-------------|
| `modelId` | string | yes | NSFW-verified model with complete refs |
| `presetId` | string | yes | Preset from v2 catalog (210 presets — category prefixes in REST doc) |
| `aspectRatio` | string | no | `1:1`, `4:5`, `9:16` (default), `16:9`, `3:4`, `2:3` |
| `count` | number | no | 1–8 images (default 1); 6 credits each |
| `integrationCallbackUrl` / `integratorWebhookSecret` | string | no | Webhook fields |

**Response:** `{ success, generationIds[], generations[], creditsCost, prompt, preset: { id, name, categoryId } }` — poll each id with `wait_for_generation`.

### `nsfw_v2_undress`

`POST /nsfw-v2/undress` — nude version of an existing image. 15 credits per image.

| Body field | Type | Required | Description |
|------------|------|----------|-------------|
| `modelId` | string | yes | |
| `sourceImageUrl` | string | yes | Public https URL of source image |
| `count` | number | no | 1–8 (default 1) |
| webhook fields | string | no | As above |

**Response:** `{ success, generationIds[], generations[], creditsCost }`.

### `nsfw_v2_free_prompt`

`POST /nsfw-v2/free-prompt` — free-text still with optional AI prompt enhancement. 6 credits per image.

| Body field | Type | Required | Description |
|------------|------|----------|-------------|
| `modelId` | string | yes | |
| `prompt` | string | yes | Your scene description |
| `aspectRatio` | string | no | Same values as presets |
| `count` | number | no | 1–8 (default 1) |
| `enhance` | boolean | no | `true` (default) = AI rewrite before render; `false` = light wrap only |
| webhook fields | string | no | As above |

**Response:** `{ success, generationIds[], generations[], creditsCost, prompt }` — when `enhance: true`, the echoed `prompt` is pre-enhancement; final text is on the generation record (visible in SPA; integrator generation rows follow normal sanitization rules).

---

## NSFW video preset sessions — typed tools

REST reference: [NSFW Video Sessions](../../public-api/15-nsfw-video.md). Multi-step flow: **preview → select → (optional edit) → approve → submit**.

### Session status machine

```
previewing → editing → approved → submitted → completed
     │           │                      │
     └───────────┴──────────────────────┴──→ failed
```

Poll `nsfw_video_get_session` every **3–5 s** during previews/edits; every **5–10 s** after submit.

### Preset catalog — `nsfw_video_presets`

`GET /nsfw-video/presets` — no parameters.

**Response:** `{ success, presets: [{ id, key, label, thumbnailUrl, durationSeconds }] }`

Use the returned **`id`** (e.g. `cpre_…`) as `presetId` when creating a session — not the stable `key`. Public scenario slugs (availability depends on admin activation):

| `key` | `label` |
|-------|---------|
| `frontal-dildo-riding` | Frontal Riding POV |
| `frontal-dildo-riding-static` | Frontal Dildo Ride Static |
| `bareback-dildo-riding` | Bareback Dildo Riding |
| `anal-dildo-riding` | Anal Dildo Riding |
| `blowjob` | Sucking Cock POV |
| `fingering` | Fingering |

Only presets with `isActive` and a live reference video appear in the list.

### `nsfw_video_create_session`

`POST /nsfw-video/sessions` — extracts first frame, starts **free** batch of 3 preview frames.

| Body field | Type | Required | Description |
|------------|------|----------|-------------|
| `modelId` | string | yes | NSFW-verified + complete refs |
| `mode` | string | yes | `preset` or `recreate` |
| `presetId` | string | preset mode | `id` from `nsfw_video_presets` |
| `uploadedVideoUrl` | string | recreate mode | Public https source clip |
| `uploadedVideoDurationSeconds` | number | recreate mode | Clip length, max **15 s** |
| `audioEnabled` | boolean | no | Recreate only; default `true`. Presets always use curated audio |
| webhook fields | string | no | Apply to the preview generation |

**Response:** `{ success, session }` with `status: "previewing"`, empty `previewImageUrls` until batch completes.

### `nsfw_video_get_session`

`GET /nsfw-video/sessions/:id`

| Parameter | Type | Required |
|-----------|------|----------|
| `sessionId` | string | yes |

**Response:** `{ success, session }` — reconciles in-flight preview/edit/final work. Key fields: `status`, `previewImageUrls[]`, `selectedPreviewUrl`, `currentFrameUrl`, `editHistory[]`, `finalGenerationId`, `sourceVideoDurationSeconds`, `creditsSpentOnPreviews`, `creditsSpentOnEdits`.

When `status === "completed"`, fetch video URL via `get_generation` / `wait_for_generation` using `finalGenerationId`.

### `nsfw_video_session_action`

`POST /nsfw-video/sessions/:id/<action>`

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `sessionId` | string | yes | Session UUID |
| `action` | enum | yes | See table below |
| `body` | object | no | Action-specific fields |

| `action` | Body | Credits | Effect |
|----------|------|---------|--------|
| `select-preview` | `{ previewUrl }` — must be in `previewImageUrls` | free | → `editing`; sets `currentFrameUrl` |
| `regenerate-previews` | `{}` | 20 | Fresh batch of 3; → `previewing` |
| `edit-frame` | `{ prompt, refImageUrl? }` | 10 | Async edit; poll session until `currentFrameUrl` updates |
| `approve` | `{}` | free | → `approved` (requires working frame, no pending edit) |
| `submit` | `{}` | `ceil(duration × 31.25)` | Final 720p video; → `submitted`; sets `finalGenerationId` |

**Typical MCP sequence:**

1. `nsfw_video_presets` → pick `presetId`
2. `nsfw_video_create_session` → loop `nsfw_video_get_session` until `previewImageUrls.length === 3`
3. `nsfw_video_session_action` `{ action: "select-preview", body: { previewUrl } }`
4. (optional) `edit-frame` → poll until edit completes
5. `nsfw_video_session_action` `{ action: "approve" }`
6. `nsfw_video_session_action` `{ action: "submit" }` → poll until `status === "completed"`
7. `wait_for_generation(finalGenerationId)` → `outputUrl`

**Sanitization:** generations created by this pipeline (`nsfw-video-preview`, `nsfw-video-frame-edit`, `nsfw-video-preset`, `nsfw-video-recreate`) return `prompt: null` and omit internal `engine` in API-key responses. User-supplied edit prompts in `editHistory[].promptUsed` are returned on the session object.

---

## NSFW routes without typed MCP tools

Use `api_v1_request` + `modelclone://v1/openapi` for: LoRA CRUD, training-image upload/generation, nudes pack, prompt planner, advanced NSFW image, extend-video, motion-video, and NSFW video recreate when you prefer raw REST over the session action enum. Full route list: `modelclone://v1/route-catalog`.

---

## ModelClone-X (`mcx_*`)

Character-identity image generation with optional trained character LoRA. See **`docs/public-api/16-modelclone-x.md`** for full REST field reference.

### Typical sequence

1. `mcx_config` — pricing, limits, feature flags (call once per session).
2. `mcx_generate` with a `body` object — returns `generationIds` (async).
3. Poll each id with `mcx_status` **or** `wait_for_generation` / `get_generation`.

### `mcx_config`

No parameters. `GET /modelclone-x/config` — live credit pricing, step/CFG limits, training image counts, and whether image-to-prompt is enabled.

### `mcx_generate`

| Parameter | Type | Description |
|-----------|------|-------------|
| `body` | object | Required. Common fields: `prompt`, `aspectRatio`, `qty` (1–4), `modelId`, `characterLoraId`, `preOptimized`, `useCustomPrompt`, `modelcloneXImg2Img`, `inputImageUrl`. See OpenAPI resource for the full schema. |

`POST /modelclone-x/generate`. Returns generation ids immediately. Character mode requires a trained LoRA (`modelId` + `characterLoraId`). Pass `integrationCallbackUrl` in `body` for webhook completion instead of polling.

### `mcx_status`

| Parameter | Type | Description |
|-----------|------|-------------|
| `generationId` | string | Required — from `mcx_generate` response |

`GET /modelclone-x/status/:generationId`. MCX-specific status view; `get_generation` works as a cross-type poll target too.

---

## img2img (`img2img_*`)

Outfit/scene transformation: re-render a source photo with your character's identity while preserving outfit, pose, and scene. Two-step flow (describe → generate) or single-step generate. See **`docs/public-api/17-img2img.md`**.

**Prerequisites:** trained character LoRA (`triggerWord`, and optionally `modelId` with NSFW LoRA unlocked).

### Typical sequence (two-step)

1. `img2img_describe` with `{ inputImageUrl, triggerWord, lookDescription? }` → `describeJobId`.
2. Poll describe status via `api_v1_request`: `GET /img2img/describe-status/:id` (no typed tool — synchronous in practice).
3. `img2img_generate` with `{ inputImageUrl, prompt, triggerWord, loraUrl, … }` → job id.
4. `img2img_status` until `completed`/`failed`.

### `img2img_describe`

| Parameter | Type | Description |
|-----------|------|-------------|
| `body` | object | Required. At minimum `inputImageUrl` (or `inputImageBase64`) and `triggerWord`. Free — no credits. |

`POST /img2img/describe`. Returns `{ describeJobId }`. Poll with `api_v1_request` `GET /img2img/describe-status/:describeJobId`.

### `img2img_generate`

| Parameter | Type | Description |
|-----------|------|-------------|
| `body` | object | Required. Source image + character LoRA fields. Omit `prompt` to run describe internally. |

`POST /img2img/generate`. Returns a job id; poll `img2img_status`.

### `img2img_status`

| Parameter | Type | Description |
|-----------|------|-------------|
| `jobId` | string | Required |

`GET /img2img/status/:jobId` — job status and `outputUrl` when complete.

---

## GPT-X (`gptx_*`)

Conversational prompt assistant — enhances natural-language requests into generation-ready prompts. **Does not generate media itself.** See **`docs/public-api/18-gptx.md`**.

### Typical sequence

1. `gptx_send` with `{ message, conversationId?, modelId?, modelName?, isNsfw?, referenceImageUrl? }` → `enhancedPrompt`, `aspectRatio`, `conversationId`, `aiMessageId`.
2. Call a generation tool (`generate_recreate`, `mcx_generate`, `creator_studio_image`, …) with the enhanced prompt.
3. Attach the result to the conversation via `api_v1_request` `PATCH /gptx/messages/:aiMessageId` with `{ generationId }`.
4. `gptx_conversations` or `api_v1_request` `GET /gptx/conversations/:id` to render the thread.

GPT-X calls are free; credits apply only to downstream generation.

### `gptx_conversations`

No parameters. `GET /gptx/conversations` — list saved assistant conversations.

### `gptx_send`

| Parameter | Type | Description |
|-----------|------|-------------|
| `body` | object | Required. At minimum `message`. Optional: `conversationId`, `modelId`, `modelName`, `isNsfw`, `referenceImageUrl`. |

`POST /gptx/send` — synchronous. Returns enhanced prompt and conversation metadata.

---

## Utility tools

Standalone media utilities. See **`docs/public-api/19-tools.md`**.

### `upscale_image`

| Parameter | Type | Description |
|-----------|------|-------------|
| `body` | object | Required. `inputImageUrl` (public HTTPS URL) or reference a prior generation via `generationId`. |

`POST /upscale` — async. Returns a `generationId`. Poll with `wait_for_generation` or `api_v1_request` `GET /upscale/status/:generationId`. Credits refunded on failure.

### `synthid_remove`

| Parameter | Type | Description |
|-----------|------|-------------|
| `body` | object | Required. Image URL or generation reference. |

`POST /synthid-remove` — async watermark removal. Poll with `wait_for_generation`.

### `video_repurpose_generate`

| Parameter | Type | Description |
|-----------|------|-------------|
| `body` | object | Required. Batch video repurposing config (source videos, output presets). |

`POST /video-repurpose/generate-with-worker` — async batch job. Returns a `jobId`.

### `video_repurpose_job`

| Parameter | Type | Description |
|-----------|------|-------------|
| `jobId` | string | Required |

`GET /video-repurpose/jobs/:jobId` — job status and output URLs.

---

## Flow Studio (`flows_*`)

Visual pipeline builder: define a graph of typed nodes connected by edges, then execute server-side. See **`docs/public-api/20-flows.md`**.

### Typical sequence

1. `flows_node_types` — discover node types, ports, default config, and per-node credit costs.
2. Build a `{ name, nodes[], edges[] }` document (see public-api flows doc for port/handle rules).
3. `flows_create` with `{ body: flowDocument }` → `flowId`.
4. `flows_run` with `{ flowId, body? }` → `{ runId, status: "pending" }`.
5. Poll `flows_run_status` with `{ runId }` every few seconds until `status` is `completed`, `failed`, or `cancelled`. Read outputs from `nodeResults`.

### `flows_list`

No parameters. `GET /flows` — saved pipelines (up to 100, most recently updated).

### `flows_node_types`

No parameters. `GET /flows/node-types` — full node type catalog with input/output ports and `defaultData`.

### `flows_get`

| Parameter | Type | Description |
|-----------|------|-------------|
| `flowId` | string | Required |

`GET /flows/:id` — flow document (nodes + edges) plus recent run summaries.

### `flows_create`

| Parameter | Type | Description |
|-----------|------|-------------|
| `body` | object | Required. `name`, `description`, `nodes`, `edges`, optional `thumbnail`, `isPublic`. |

`POST /flows` — creates a flow. Returns `{ flow: { id, … } }`.

### `flows_run`

| Parameter | Type | Description |
|-----------|------|-------------|
| `flowId` | string | Required |
| `body` | object | Optional run overrides |

`POST /flows/:id/run` — fire-and-forget background execution. Returns `{ runId, status: "pending" }`.

### `flows_run_status`

| Parameter | Type | Description |
|-----------|------|-------------|
| `runId` | string | Required |

`GET /flows/runs/:runId` — canonical poll target. Returns `status`, `creditsUsed`, and per-node `nodeResults`.

**Additional flow routes** (no typed tools — use `api_v1_request`): `PUT /flows/:id`, `DELETE /flows/:id`, `GET /flows/:id/runs`, `DELETE /flows/runs/:runId` (cancel), `POST /flows/estimate-credits`.

### SSE live stream (direct HTTP only)

MCP has **no SSE tool**. For real-time flow progress outside MCP, open a direct REST connection:

```
GET /api/v1/flows/runs/:runId/stream
```

- Content-Type: `text/event-stream`
- Same API-key auth as other endpoints
- Pushes `log`, `node`, and `flow` events as JSON in SSE `data:` frames
- Heartbeat comment lines every ~20s — ignore them
- Connection lifetime is capped (~25s default); server sends a `reconnect` event and closes — **reconnect immediately** or fall back to `flows_run_status` polling
- If the run is already terminal when you connect, one final payload is sent and the stream closes

**In MCP agents:** prefer `flows_run_status` polling (every 3–5s). SSE is for custom HTTP clients and the web UI, not MCP tool calls.

---

## Community gallery (`gallery_*`)

Browse and interact with the public SFW community feed. See **`docs/public-api/21-gallery.md`**.

**Note:** You cannot post directly to the gallery. Content enters when an eligible SFW generation is submitted with `shareWithCommunity: true` on the generation request. A username (`PUT /me/username` via `api_v1_request`) is required before sharing.

### `gallery_feed`

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | int 1–100 | Optional page size |
| `cursor` | string | Optional pagination cursor |
| `sort` | string | Optional — `popular`, `recent`, `oldest` |

`GET /public-gallery` — community feed. Works without auth; authenticated calls include `likedByMe`.

### `gallery_post`

| Parameter | Type | Description |
|-----------|------|-------------|
| `postId` | string | Required |

`GET /public-gallery/:postId` — one post with media, prompt, and engagement counts.

### `gallery_like`

| Parameter | Type | Description |
|-----------|------|-------------|
| `postId` | string | Required |

`POST /public-gallery/:postId/like` — toggle like (idempotent toggle).

### `gallery_comment`

| Parameter | Type | Description |
|-----------|------|-------------|
| `postId` | string | Required |
| `text` | string | Required comment body |

`POST /public-gallery/:postId/comments` — add a comment.

**Additional gallery routes** (via `api_v1_request`): `PUT /me/username`, share-as-video, list/delete own publications — see OpenAPI resource.

---

## API keys (no typed tools)

Integrator API key CRUD is on the product surface but has **no dedicated MCP tools**. Use `api_v1_request`:

| Action | Call |
|--------|------|
| List keys | `GET /user/api-keys` |
| Create key | `POST /user/api-keys` with `{ name?, scopes?, defaultWebhookUrl? }` |
| Update key | `PATCH /user/api-keys/:keyId` — rename, scopes, default webhook |
| Delete key | `DELETE /user/api-keys/:keyId` |
| Rotate secret | `POST /user/api-keys/:keyId/regenerate` |

All are synchronous. The raw `mcl_…` secret is returned only on create/regenerate — store it immediately. Body shapes: `modelclone://v1/openapi` paths under `/user/api-keys`.

**Do not delete the key currently bound to your MCP session** — subsequent tool calls will fail with `401`.

---

## When to use typed tools vs `api_v1_request`

| Situation | Prefer |
|-----------|--------|
| Common product action with a typed tool | Typed tool (`mcx_generate`, `flows_run`, …) |
| Route has no typed tool (describe-status, flow cancel, GPT-X message patch, API keys) | `api_v1_request` |
| Unsure a path exists | Read `modelclone://v1/route-catalog` first |
| Unsure of JSON body shape | Read `modelclone://v1/openapi` first |
