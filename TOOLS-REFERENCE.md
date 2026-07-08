# ModelClone MCP server

The ModelClone MCP server connects your ModelClone account to any MCP-capable AI client — Claude.ai, Claude Code, Claude Desktop, Cursor, or any tool that speaks MCP over Streamable HTTP. Once connected, the assistant can do everything the product does through the API: create AI models, generate SFW and NSFW images and videos, run ModelClone-X and img2img, drive Flow Studio pipelines, browse the community gallery, and poll jobs to completion.

- **Endpoint:** `https://mcp.modelclone.app/mcp`
- **Transport:** Streamable HTTP (Model Context Protocol)
- **Auth:** your integrator API key on every request — `X-Api-Key: mcl_…` or `Authorization: Bearer mcl_…`
- **Server version:** `2.0.0`

Everything the MCP does maps 1:1 onto the REST API at `https://modelclone.app/api/v1`. One MCP tool call is exactly one REST call — nothing is cached server-side, and the same credit costs and rate limits apply.

---

## Getting a key

Create and manage API keys in the app under **Settings → API**. Keys are prefixed `mcl_`. Send the key on every MCP request; Claude's connector "OAuth token" / API-key field maps directly to this header.

---

## Setup

### Claude.ai (web connector)

**Settings → Connectors → Add custom connector:**

- URL: `https://mcp.modelclone.app/mcp`
- Auth: paste your `mcl_…` key (sent as the bearer token)

### Claude Code

```bash
claude mcp add --transport http modelclone https://mcp.modelclone.app/mcp \
  --header "Authorization: Bearer mcl_YOUR_KEY"
```

### Cursor

Add to `.cursor/mcp.json` (project) or `~/.cursor/mcp.json` (global):

```json
{
  "mcpServers": {
    "modelclone": {
      "url": "https://mcp.modelclone.app/mcp",
      "headers": {
        "Authorization": "Bearer mcl_YOUR_KEY"
      }
    }
  }
}
```

### Claude Desktop / generic Streamable HTTP clients

Point any Streamable-HTTP MCP client at `https://mcp.modelclone.app/mcp` and send the header `Authorization: Bearer mcl_…` (or `X-Api-Key: mcl_…`) on every request.

### Verify connectivity

The health endpoint needs no auth and returns the server manifest (version, tool groups, resources):

```bash
curl https://mcp.modelclone.app/mcp/health
```

---

## Session behavior

- On the first request, the server issues an `mcp-session-id`. Send it back on subsequent requests (standard MCP client behavior handles this automatically).
- **Sessions are bound to the API key** that initialized them. A request that reuses a session id with a different key is rejected with `403` ("Session bound to a different API key").
- **Idle sessions expire after 30 minutes.** After expiry, just initialize a new session (re-connect) — no state is lost because everything is persisted server-side under your account.

---

## Tool reference

Every **submit** tool returns a generation ID (or job/session id). To get the final result, poll with `wait_for_generation` (or the relevant status tool). Parameters below are exactly the tool input schemas.

Many tools accept a free-form `body` or `options` object — additional route-specific JSON fields. When unsure of the exact shape, read the `modelclone://v1/openapi` resource.

### Account & meta

| Tool | Parameters | Description |
|---|---|---|
| `get_me` | — | `GET /me` — profile, credits, subscription tier. Call first in a session. |
| `get_pricing_generation` | — | `GET /pricing/generation` — authoritative live credit costs per generation type. |
| `confirm_adult` | — | `POST /auth/confirm-adult` — one-time 18+ confirmation unlocking NSFW features. |
| `get_notifications` | `page`, `pageSize`, `readState` (optional) | `GET /me/notifications/` — paginated notification history. |
| `notifications_mark_read` | `ids[]` (required) | `POST /me/notifications/mark-read` |
| `notifications_mark_all_read` | — | `POST /me/notifications/mark-all-read` |
| `get_notification_preferences` | — | `GET /me/notifications/preferences` |
| `set_notification_preferences` | `updates[]` (`category`, `channel`, `enabled`) | `PUT /me/notifications/preferences` |
| `get_my_flags` | — | `GET /me/flags` — account feature flags. |
| `get_plans` | — | `GET /plans` — subscription plan catalog (read-only; billing is session-only). |
| `api_v1_request` | `method` (enum `GET`/`POST`/`PUT`/`PATCH`/`DELETE`, required), `path` (string, required — under `/api/v1`, e.g. `/nsfw/generate`), `query` (object of string/number/boolean, optional), `body` (object, optional) | Generic escape hatch: call **any** allowed integrator route. Allowed paths are in the `modelclone://v1/route-catalog` resource; request/response shapes in `modelclone://v1/openapi`. |

Web push token registration (`GET /me/notifications/vapid-public-key`, token CRUD) requires a browser `PushManager` — document for clone apps (persona B), not API-key backends. See [Notifications](./23-notifications.md).

### Generations (history & polling)

| Tool | Parameters | Description |
|---|---|---|
| `list_generations` | `type` (string), `modelId` (uuid), `status` (string — `processing`/`completed`/`failed` or comma list), `limit` (int 1–200), `offset` (int ≥0), `includeTotal` (boolean) — all optional | `GET /generations` — paginated history across all types. |
| `get_generation` | `generationId` (string, required) | `GET /generations/:id` — status + `outputUrl`. Canonical poll target. |
| `wait_for_generation` | `generationId` (string, required), `timeoutSec` (int 5–570, default 120), `intervalSec` (int 2–30, default 5) | Server-side poll until the generation is `completed`/`failed` or the timeout elapses. **Prefer this after any submit.** |
| `generations_batch_delete` | `body` with `ids[]` | `POST /generations/batch-delete` — delete multiple generations. |
| `generations_monthly_stats` | — | `GET /generations/monthly-stats` — monthly counts for the account. |

### Models & the appearance wizard

Model creation is **asynchronous** in the final step: pose/upload endpoints return **202** with `model.status: "processing"`. Poll with `models_status` every 3–5s until `ready` or `failed`. Confirm capacity with `list_models` (`canCreateMore`) before creating.

Live credit costs: call `get_pricing_generation`. Defaults below — always treat pricing response as authoritative.

| Tool | Pricing key | Default credits |
|------|-------------|-----------------|
| `models_generate_reference` | `modelStep1Reference` | 150 |
| `models_generate_poses` | `modelStep2Poses` | 750 |
| `wizard_custom_reference` (`regenerate: true`) | `wizardCustomRegenerate` | 10 |
| All other model/wizard tools listed here | — | 0 |

#### Creation paths

| Path | Tools (in order) | Poll |
|------|------------------|------|
| Wizard — niche | `wizard_look_variants` → `wizard_preview_images` → `wizard_finalize_poses` | `models_status` |
| Wizard — custom text | `wizard_custom_reference` → `wizard_finalize_poses` | `models_status` |
| Wizard — upload | `wizard_upload_save` | `models_status` |
| Classic two-step | `models_generate_reference` → `models_generate_poses` | `models_status` |
| Direct URLs | `create_model` | — (sync) |

Full per-tool schemas and examples: [`docs/public-api/10-models-and-wizard.md`](./10-models-and-wizard.md) and MCP deep reference [`docs/mcp/sections/07-tools-convenience.md`](../mcp/sections/07-tools-convenience.md) (merged into [`docs/mcp/MODELCLONE_MCP.md`](../mcp/MODELCLONE_MCP.md)).

#### `list_models`

No parameters. `GET /models` — returns `models[]`, `count`, `limit`, `canCreateMore`.

#### `get_model`

| Parameter | Type | Required |
|-----------|------|----------|
| `modelId` | string (UUID) | Yes |

`GET /models/:id` — reference photos, `status`, appearance, voice metadata.

#### `create_model`

| Field (in `body`) | Type | Required |
|-------------------|------|----------|
| `name` | string | Yes |
| `photo1Url`, `photo2Url`, `photo3Url` | string (URL) | Yes |
| `savedAppearance` | object | No |

`POST /models` — synchronous; no generation. User-uploaded models cannot use NSFW features.

**Example:**

```json
{
  "body": {
    "name": "Jordan",
    "photo1Url": "https://cdn.example.com/selfie.jpg",
    "photo2Url": "https://cdn.example.com/portrait.jpg",
    "photo3Url": "https://cdn.example.com/fullbody.jpg"
  }
}
```

#### `delete_model`

| Parameter | Type | Required |
|-----------|------|----------|
| `modelId` | string (UUID) | Yes |

`DELETE /models/:id`. **409** if generations for this model are still in progress.

#### `models_generate_reference`

| Field (in `body`) | Type | Required |
|-------------------|------|----------|
| `gender` | string | Yes |
| `age` | number/string | Yes (18–120) |
| All appearance chip fields | string | Yes |
| `referencePrompt` | string | No |
| `regenerate` | boolean | No |

`POST /models/generate-reference` — **synchronous** (200). Returns `referenceUrl`. Missing appearance categories → **400** with `missing` array.

**Example:**

```json
{
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
```

#### `models_generate_poses`

| Field (in `body`) | Type | Required |
|-------------------|------|----------|
| `name` | string | Yes |
| `referenceUrl` | string (URL) | Yes |
| `gender`, `age`, appearance fields | — | Recommended |
| `posesPrompt`, `outfitType`, `poseStyle` | string | No |

`POST /models/generate-poses` — **async** (202). Poll `models_status` with `model.id`.

**Example:**

```json
{
  "body": {
    "name": "Riley",
    "referenceUrl": "https://cdn.modelclone.app/references/ref-4f2a.jpg",
    "gender": "female",
    "age": "25"
  }
}
```

#### `models_status`

| Parameter | Type | Required |
|-----------|------|----------|
| `id` | string (UUID) | Yes |

`GET /models/status/:id`. Status values: `processing`, `ready`, `failed` (includes `message` on failure).

#### `wizard_look_variants`

| Field (in `body`) | Type | Required |
|-------------------|------|----------|
| `gender`, `age`, `nicheName` | — | Yes |
| `nicheId`, `nicheBio` | string | No |
| `ethnicity`, `hairColor`, `eyeColor`, `bodyType` | string | No |

`POST /wizard/niche/look-variants` — free, synchronous. Returns four `variants[]` with `label` and `looks`.

**Example:**

```json
{ "body": { "gender": "female", "age": 24, "nicheName": "Fitness" } }
```

#### `wizard_preview_images`

| Field (in `body`) | Type | Required |
|-------------------|------|----------|
| `gender`, `age`, `variants` | — | Yes |
| `nicheId`, `nicheName`, `nicheBio` | string | No |

`POST /wizard/niche/preview-images` — free, synchronous. Pass `variants` from the previous step.

**Example:**

```json
{
  "body": {
    "gender": "female",
    "age": 24,
    "variants": [{ "label": "Soft", "looks": { "gender": "female", "ethnicity": "Latina" } }]
  }
}
```

#### `wizard_custom_reference`

| Field (in `body`) | Type | Required |
|-------------------|------|----------|
| `gender`, `age` | — | Yes |
| `referencePrompt` | string | No |
| `regenerate` | boolean | No |
| appearance fields / `savedAppearance` / `look` | — | No |

`POST /wizard/custom/reference` — synchronous. First call free; `regenerate: true` charged.

**Example:**

```json
{
  "body": {
    "gender": "female",
    "age": 26,
    "referencePrompt": "minimalist fashion creator, platinum blonde bob"
  }
}
```

#### `wizard_upload_save`

| Field (in `body`) | Type | Required |
|-------------------|------|----------|
| `name`, `photo1Url`, `photo2Url`, `photo3Url` | — | Yes |
| `age`, `gender`, `savedAppearance` | — | No |

`POST /wizard/upload/save` — free, async (202). Poll `models_status`.

#### `wizard_finalize_poses`

Same body as `models_generate_poses` — **free** on the wizard path. `POST /wizard/finalize-poses` → async (202) → poll `models_status`.

**Example (after preview step):**

```json
{
  "body": {
    "name": "FitCreator",
    "referenceUrl": "https://cdn.modelclone.app/references/prev-1.jpg",
    "gender": "female",
    "age": 24,
    "ethnicity": "Latina",
    "hairColor": "Dark Brown"
  }
}
```

### SFW image & video generation

REST details: [Image generation](./11-image-generation.md) · [Video generation](./12-video-generation.md) · [Creator Studio](./13-creator-studio.md). Full MCP field reference: [`docs/mcp/sections/07-tools-convenience.md`](../mcp/sections/07-tools-convenience.md).

| Tool | MCP input | REST route | Completion |
|---|---|---|---|
| `generate_image_identity` | `body` | `POST /generate/image-identity` | Async → `wait_for_generation` |
| `generate_recreate` | `modelId`, `options` (use `sourceImageUrl`, not top-level `imageUrl`) | `POST /generate/recreate` | Async → `wait_for_generation` |
| `generate_free` | `prompt`, `options` (must include `modelId`) | `POST /generate/free` | Async → `wait_for_generation` |
| `generate_preset_recreate` | `body` | `POST /generate/preset-recreate` | Async → `wait_for_generation` |
| `enhance_prompt` | `prompt`, `options` | `POST /generate/enhance-prompt` | **Sync** — read `body.enhancedPrompt` |
| `generate_motion_video` | `body` (`imageUrl`, `videoUrl`, …) | `POST /generate/motion-video` | Async → `wait_for_generation` |
| `generate_video_motion` | `body` | `POST /generate/video-motion` | Async → `wait_for_generation` |
| `generate_video_directly` | `body` | `POST /generate/video-directly` | Async → `wait_for_generation` |
| `generate_face_swap_video` | `body` | `POST /generate/face-swap-video` | Async → `wait_for_generation` |
| `generate_image_faceswap` | `body` | `POST /generate/image-faceswap` | Async → `wait_for_generation` |
| `generate_complete_recreation` | `body` | `POST /generate/complete-recreation` | Async — **two** generation ids (image + video) |
| `describe_target` | `body` | `POST /generate/describe-target` | **Sync** |
| `extract_frames` | `body` | `POST /generate/extract-frames` | **Sync** (free helper) |
| `generate_advanced` | `body` | `POST /generate/advanced` | Async → `wait_for_generation` |
| `creator_studio_image` | `body` | `POST /generate/creator-studio` | Async → `wait_for_generation` |
| `creator_studio_video` | `body` | `POST /generate/creator-studio/video` | Async → `wait_for_generation` |
| `creator_studio_extend` | `body` | `POST /generate/creator-studio/video/extend` | Async → `wait_for_generation` |
| `creator_studio_4k` | `body` | `POST /generate/creator-studio/video/4k` | Async → `wait_for_generation` |
| `creator_studio_1080p` | query `taskId` | `GET /generate/creator-studio/video/1080p` | Poll `wait_for_generation` |

**Mask uploads:** MCP is JSON-only. Presign → PUT → pass `maskUrl` in `creator_studio_video` body. See [Uploads](./06-uploads.md).

**Webhooks:** pass `integrationCallbackUrl` (+ optional `integratorWebhookSecret`) inside `options` or `body`. See [Webhooks](./05-webhooks.md).

**Credits:** call `get_pricing_generation` before quoting. Key defaults: recreate **10–16**/image; free **5–25**/image (+ enhancer); enhance **5** sync; motion **9.5**/s; Creator Studio per model/family (see linked docs).

#### `generate_recreate`

| REST field (`options` or body) | Type | Required | Notes |
|--------------------------------|------|----------|-------|
| `sourceImageUrl` | URL | Yes | Photo to recreate (public HTTPS) |
| `modelId` | UUID | Yes | Model with 3 reference photos |
| `count` | int | No | 1–10 (default 1) |
| `outfitMode` | string | No | `model` (default), `source`, `external` |
| `externalOutfitImageUrl` | URL | When external | |
| `extraGuidance` | string | No | Max 400 chars |
| `genModel` | string | No | `wan-2.7-image` (default) or `nano-banana-pro` |
| webhook fields | string | No | `integrationCallbackUrl`, `integratorWebhookSecret` |

Credits: `recreateImage` (**10**/image) or `recreateImageNanoBanana` (**16**/image) × `count`.

```json
{
  "modelId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "options": {
    "sourceImageUrl": "https://cdn.example.com/inspo/pose.jpg",
    "outfitMode": "model",
    "integrationCallbackUrl": "https://your.app/hooks/modelclone"
  }
}
```

#### `generate_free`

Requires `modelId` in `options` (identity locked by 3 reference photos).

| REST field | Type | Required | Notes |
|------------|------|----------|-------|
| `modelId` | UUID | Yes | |
| `prompt` | string | Yes | Top-level MCP param |
| `genModel` | string | No | `nano-banana-pro` (default), `wan-2.7-image`, `seedream-4.5-edit` |
| `aspectRatio` / `resolution` | string | No | Engine-dependent — see [model-caps](./11-image-generation.md#get-generate-model-caps) |
| `enhance` | boolean | No | Default `true` (+`enhancePromptDefault`, **5**) |
| `count` | int | No | 1–8 |
| webhook fields | string | No | |

```json
{
  "prompt": "mirror selfie, soft morning hotel light",
  "options": {
    "modelId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "genModel": "nano-banana-pro",
    "aspectRatio": "9:16",
    "resolution": "2K"
  }
}
```

#### `enhance_prompt`

Synchronous — no `wait_for_generation`. Credits: `enhancePromptDefault` (**5** default).

```json
{
  "prompt": "coffee shop candid",
  "options": { "genModel": "nano-banana-pro", "modelLooks": { "gender": "female" } }
}
```

#### `generate_motion_video`

Driving clip 3–15 s. Credits: `motionXPerSec` (**9.5**/s default) × duration.

```json
{
  "body": {
    "imageUrl": "https://cdn.modelclone.app/generations/still.png",
    "videoUrl": "https://cdn.example.com/dance.mp4",
    "duration": 8
  }
}
```

Poll with `wait_for_generation` using `body.generationId`.

#### `creator_studio_image` / `creator_studio_video` / `creator_studio_extend` / `creator_studio_4k`

Pass full REST body. Image: `prompt`, `generationModel`, `aspectRatio`, `resolution`, `referencePhotos`, `maskUrl`, `numImages`. Video: `family`, `mode`, `prompt`, `durationSeconds`, `imageUrl`, … — see [Creator Studio](./13-creator-studio.md). Video rate bucket: `cinematic` (30/min). For mask-requiring modes, upload via presign/PUT first.

### NSFW studio

REST detail: [NSFW Studio](./14-nsfw.md). Account gate: first purchase + 18+ confirmation (`403 NSFW_NEEDS_PURCHASE` / `NSFW_NEEDS_AGE_CONFIRMATION`); call `confirm_adult` first. Classic LoRA tools need `nsfwUnlocked`; v2 tools need NSFW-verified model + three reference photos (**403** `NSFW_NOT_VERIFIED` until support approves). Custom LoRA training photo uploads require account permission (**403** otherwise).

| Tool | REST | Role |
|------|------|------|
| `nsfw_generate` | `POST /nsfw/generate` | Classic NSFW image (trained LoRA) |
| `nsfw_generate_video` | `POST /nsfw/generate-video` | Image-to-video (5 s / 8 s) |
| `nsfw_train_lora` | `POST /nsfw/train-lora` | Start LoRA training |
| `nsfw_training_status` | `GET /nsfw/training-status/:modelId` | Training progress |
| `nsfw_lora_create` | `POST /nsfw/lora/create` | Create LoRA record |
| `nsfw_lora_list` | `GET /nsfw/loras/:modelId` | List LoRAs |
| `nsfw_lora_set_active` | `POST /nsfw/lora/set-active` | Set active LoRA |
| `nsfw_lora_delete` | `DELETE /nsfw/lora/:loraId` | Delete LoRA |
| `nsfw_initialize_training` | `POST /nsfw/initialize-training` | Init training session |
| `nsfw_generate_training_images` | `POST /nsfw/generate-training-images` | Generate training images |
| `nsfw_assign_training_images` | `POST /nsfw/assign-training-images` | Assign gallery images to training |
| `nsfw_nudes_pack` | `POST /nsfw/nudes-pack` | Batch pose pack |
| `nsfw_extend_video` | `POST /nsfw/extend-video` | Extend NSFW video |
| `nsfw_motion_video` | `POST /nsfw/generate-motion-video` | Motion-control NSFW video |
| `nsfw_advanced_image` | `POST /nsfw/generate-advanced` | Advanced NSFW image |
| `nsfw_v2_preset` | `POST /nsfw-v2/presets` | Fast preset still |
| `nsfw_v2_undress` | `POST /nsfw-v2/undress` | Undress still |
| `nsfw_v2_free_prompt` | `POST /nsfw-v2/free-prompt` | Free-text still |

#### Sexting scripts (`sexting_*`)

Fully documented even when disabled. Non-admin accounts receive **503** `SEXTING_SCRIPTS_UNAVAILABLE` while the feature flag is off server-side.

| Tool | REST |
|------|------|
| `sexting_scripts_list` | `GET /nsfw/sexting-scripts` |
| `sexting_script_get` | `GET /nsfw/sexting-scripts/:id` |
| `sexting_script_create` | `POST /nsfw/sexting-scripts` |
| `sexting_script_run` | `POST /nsfw/sexting-scripts/:id/run` |
| `sexting_script_runs` | `GET /nsfw/sexting-scripts/runs` |
| `sexting_script_run_status` | `GET /nsfw/sexting-scripts/runs/:runId` |

Deep schemas: `docs/mcp/sections/07-tools-convenience.md`.

#### `nsfw_generate`

| Parameter | Type | Required |
|-----------|------|----------|
| `modelId` | string | Yes (API; tool schema optional) |
| `prompt` | string | Yes |
| `options` | object | No — merged into body |

Common fields via `options` or top-level: `quantity` (1\|2), `attributes`, `attributesDetail`, `sceneDescription`, `options.loraStrength`, `options.quickFlow`, `options.resolution`, `options.postProcessing`, `integrationCallbackUrl`, `integratorWebhookSecret`.

**Response:** `{ generation, generations[], creditsUsed }` → `wait_for_generation`.

#### `nsfw_generate_video`

| Body field | Type | Required |
|------------|------|----------|
| `modelId` | string | Yes |
| `imageUrl` | string | Yes |
| `prompt` | string | No |
| `duration` | number | No — `5` (default) or `8` |

**Response:** `{ generationId, creditsUsed, duration }` → `wait_for_generation`.

#### `nsfw_train_lora`

| Body field | Type | Required |
|------------|------|----------|
| `modelId` | string | Yes |
| `loraId` | string | No |

**Response:** `{ deferred: true, triggerWord, creditsUsed }` → poll `nsfw_training_status` (~60 s interval).

#### `nsfw_training_status`

| Parameter | Type | Required |
|-----------|------|----------|
| `modelId` | string | Yes |

**Response:** `{ status, nsfwUnlocked, loraUrl?, triggerWord?, loraId? }`. REST query `?loraId=` not exposed on typed tool — use `api_v1_request` if needed.

#### `nsfw_v2_preset`

| Body field | Type | Required |
|------------|------|----------|
| `modelId` | string | Yes |
| `presetId` | string | Yes — v2 catalog id (210 presets; category prefixes in REST doc) |
| `aspectRatio` | string | No — `1:1`, `4:5`, `9:16`, `16:9`, `3:4`, `2:3` |
| `count` | number | No — 1–8, default 1 |

**Response:** `{ generationIds[], creditsCost, preset }` → poll each with `wait_for_generation`.

#### `nsfw_v2_undress`

| Body field | Type | Required |
|------------|------|----------|
| `modelId` | string | Yes |
| `sourceImageUrl` | string | Yes |
| `count` | number | No |

#### `nsfw_v2_free_prompt`

| Body field | Type | Required |
|------------|------|----------|
| `modelId` | string | Yes |
| `prompt` | string | Yes |
| `aspectRatio` | string | No |
| `count` | number | No |
| `enhance` | boolean | No — default `true` |

**Webhooks:** add `integrationCallbackUrl` + `integratorWebhookSecret` to any submit body ([Webhooks](./05-webhooks.md)).

### NSFW video preset sessions (multi-step)

REST detail: [NSFW Video Sessions](./15-nsfw-video.md). Flow: **presets → create session → poll previews → select → (edit) → approve → submit → poll completed**.

| Tool | REST | Role |
|------|------|------|
| `nsfw_video_presets` | `GET /nsfw-video/presets` | List active presets |
| `nsfw_video_create_session` | `POST /nsfw-video/sessions` | Start session + free preview batch |
| `nsfw_video_get_session` | `GET /nsfw-video/sessions/:id` | Poll session state |
| `nsfw_video_session_action` | `POST /nsfw-video/sessions/:id/<action>` | Advance session |

#### `nsfw_video_presets`

No parameters. Returns `{ presets: [{ id, key, label, thumbnailUrl, durationSeconds }] }`. Use **`id`** as `presetId` when creating a session.

Public scenario keys (when activated): `frontal-dildo-riding` (Frontal Riding POV), `frontal-dildo-riding-static`, `bareback-dildo-riding`, `anal-dildo-riding`, `blowjob` (Sucking Cock POV), `fingering`.

#### `nsfw_video_create_session`

| Body field | Type | Required |
|------------|------|----------|
| `modelId` | string | Yes |
| `mode` | string | Yes — `preset` or `recreate` |
| `presetId` | string | Preset mode — from presets list |
| `uploadedVideoUrl` | string | Recreate mode — public https |
| `uploadedVideoDurationSeconds` | number | Recreate mode — max 15 s |
| `audioEnabled` | boolean | Recreate only; presets use curated audio |

First preview batch (3 frames) is **free**. Optional webhook fields apply to the preview generation.

#### `nsfw_video_get_session`

| Parameter | Type | Required |
|-----------|------|----------|
| `sessionId` | string | Yes |

Poll every **3–5 s** until `previewImageUrls.length === 3` or edit completes; **5–10 s** after submit. When `status === "completed"`, use `finalGenerationId` with `wait_for_generation`.

#### `nsfw_video_session_action`

| Parameter | Type | Required |
|-----------|------|----------|
| `sessionId` | string | Yes |
| `action` | enum | Yes — see table |
| `body` | object | Per action |

| `action` | `body` | Credits |
|----------|--------|---------|
| `select-preview` | `{ previewUrl }` | 0 |
| `regenerate-previews` | `{}` | 20 |
| `edit-frame` | `{ prompt, refImageUrl? }` | 10 |
| `approve` | `{}` | 0 |
| `submit` | `{}` | `ceil(sourceVideoDurationSeconds × 31.25)` |

**Sanitization:** generations for this pipeline return `prompt: null` and omit internal `engine` in API-key/MCP JSON. User edit prompts remain on `session.editHistory[].promptUsed`. Global provider fields stripped on all API-key responses.

**Webhooks:** pass webhook fields on `create_session`, `edit-frame`, and `submit` bodies when those steps create generations. Session polling remains the primary status loop.

### ModelClone-X (`mcx_*`)

Character-identity image generation. See [ModelClone-X](./16-modelclone-x.md).

| Tool | Parameters | Description |
|---|---|---|
| `mcx_config` | — | `GET /modelclone-x/config` — live pricing, limits, feature flags. Call once per session. |
| `mcx_generate` | `body` (object, required) | `POST /modelclone-x/generate` — async; returns `generationIds`. Common fields: `prompt`, `aspectRatio`, `qty`, `modelId`, `characterLoraId`, `preOptimized`. |
| `mcx_status` | `generationId` (string, required) | `GET /modelclone-x/status/:generationId` — MCX job status. `wait_for_generation` also works. |

**Typical flow:** `mcx_config` → `mcx_generate` → poll `mcx_status` or `wait_for_generation`.

### img2img (`img2img_*`)

Outfit/scene transformation with a trained character LoRA. See [img2img](./17-img2img.md).

| Tool | Parameters | Description |
|---|---|---|
| `img2img_describe` | `body` (object, required — `inputImageUrl`/`inputImageBase64`, `triggerWord`) | `POST /img2img/describe` — free vision pass → `describeJobId`. |
| `img2img_describe_status` | `id` (string, required) | `GET /img2img/describe-status/:id` — poll describe job. |
| `img2img_generate` | `body` (object, required) | `POST /img2img/generate` — async transformation. Omit `prompt` to describe internally. |
| `img2img_status` | `jobId` (string, required) | `GET /img2img/status/:jobId` — job status + `outputUrl`. |

**Typical flow:** `img2img_describe` → `img2img_describe_status` → `img2img_generate` → `img2img_status`.

### GPT-X (`gptx_*`)

Conversational prompt assistant — enhances messages into generation-ready prompts; does **not** generate media itself. See [GPT-X](./18-gptx.md).

| Tool | Parameters | Description |
|---|---|---|
| `gptx_conversations` | — | `GET /gptx/conversations` — list saved conversations. |
| `gptx_send` | `body` (object, required — `message`; optional `conversationId`, `modelId`, `modelName`, `isNsfw`, `referenceImageUrl`) | `POST /gptx/send` — synchronous; returns `enhancedPrompt`, `aspectRatio`, `conversationId`, `aiMessageId`. |

**Typical flow:** `gptx_send` → generate with any image tool using `enhancedPrompt` → `api_v1_request` `PATCH /gptx/messages/:aiMessageId` with `{ generationId }` to attach the result.

### Utility tools

See [Utility tools](./19-tools.md).

| Tool | Parameters | Description |
|---|---|---|
| `upscale_image` | `body` (object, required — `inputImageUrl` or generation reference) | `POST /upscale` — async upscale. Poll with `wait_for_generation` or `api_v1_request` `GET /upscale/status/:generationId`. |
| `synthid_remove` | `body` (object, required) | `POST /synthid-remove` — async watermark removal. Poll with `wait_for_generation`. |
| `video_repurpose_generate` | `body` (object, required) | `POST /video-repurpose/generate-with-worker` — async batch repurposing. Returns `jobId`. |
| `video_repurpose_job` | `jobId` (string, required) | `GET /video-repurpose/jobs/:jobId` — job status + output URLs. |
| `reformatter_convert` | `body` (object, required) | `POST /reformatter/convert-with-worker` — async media conversion. |
| `reformatter_status` | `jobId` (string, required) | `GET /reformatter/status/:jobId` — converter job status. |

### Flow Studio (`flows_*`)

Visual pipeline builder. See [Flow Studio](./20-flows.md).

| Tool | Parameters | Description |
|---|---|---|
| `flows_list` | — | `GET /flows` — saved pipelines (up to 100). |
| `flows_node_types` | — | `GET /flows/node-types` — node type catalog (ports, defaults, credit costs). |
| `flows_get` | `flowId` (string, required) | `GET /flows/:id` — flow document (nodes + edges) + recent runs. |
| `flows_create` | `body` (object, required — `name`, `nodes`, `edges`, …) | `POST /flows` — create a flow. |
| `flows_update` | `flowId`, `body` | `PUT /flows/:id` — update flow document. |
| `flows_delete` | `flowId` (string, required) | `DELETE /flows/:id` |
| `flows_run` | `flowId` (string, required), `body` (object, optional) | `POST /flows/:id/run` — fire-and-forget execution. Returns `{ runId, status: "pending" }`. |
| `flows_run_status` | `runId` (string, required) | `GET /flows/runs/:runId` — canonical poll target; read `nodeResults` when `status` is terminal. |
| `flows_cancel_run` | `runId` (string, required) | `DELETE /flows/runs/:runId` — cancel pending/running run. |
| `flows_runs` | `flowId` (string, required) | `GET /flows/:id/runs` — run history. |

**Typical flow:** `flows_node_types` → build nodes/edges document → `flows_create` → `flows_run` → poll `flows_run_status` every 3–5s.

**SSE (direct HTTP only):** `GET /flows/runs/:runId/stream` pushes real-time `log`, `node`, and `flow` events over Server-Sent Events. MCP has no streaming tool — use `flows_run_status` polling in agents.

### Community gallery (`gallery_*`)

See [Gallery](./21-gallery.md). Posts enter the gallery when eligible SFW generations are submitted with `shareWithCommunity: true` — not via a direct "post" tool.

| Tool | Parameters | Description |
|---|---|---|
| `gallery_feed` | `limit` (int 1–100), `cursor` (string), `sort` (`popular`/`recent`/`oldest`) — all optional | `GET /public-gallery` — community feed. Works without auth; authenticated calls include `likedByMe`. |
| `gallery_post` | `postId` (string, required) | `GET /public-gallery/:postId` — one post with media. |
| `gallery_like` | `postId` (string, required) | `POST /public-gallery/:postId/like` — toggle like. |
| `gallery_comment` | `postId` (string, required), `text` (string, required) | `POST /public-gallery/:postId/comments` — add a comment. |
| `gallery_my_publications` | query params optional | `GET /me/public-gallery/publications` |
| `gallery_unpublish` | `id` (string, required) | `DELETE /me/public-gallery/publications/:id` |
| `set_username` | `body` with `username` | `PUT /me/username` |

Posts enter the gallery when eligible generations are submitted with `shareWithCommunity: true` — not via a direct publish tool. Profile: `api_v1_request` `GET /me/public-profile`.

### Talking avatars (`avatars_*`)

See [Video generation](./12-video-generation.md). Poll with `avatars_video_status`, not `wait_for_generation`.

| Tool | REST |
|------|------|
| `avatars_list` | `GET /avatars?modelId=` |
| `avatars_detect_subjects` | `POST /avatars/detect-subjects` |
| `avatars_preview_audio` | `POST /avatars/preview-audio` |
| `avatars_generate` | `POST /avatars/generate` |
| `avatars_video_status` | `GET /avatars/videos/:videoId` |

Upload photo/audio via presign/PUT or `POST /upload` first; pass URLs in `avatars_generate` body.

### API keys (no typed tools)

Key CRUD has no dedicated MCP tools. Use `api_v1_request`:

| Action | Call |
|---|---|
| List | `GET /user/api-keys` |
| Create | `POST /user/api-keys` |
| Update | `PATCH /user/api-keys/:keyId` |
| Delete | `DELETE /user/api-keys/:keyId` |
| Rotate secret | `POST /user/api-keys/:keyId/regenerate` |

All synchronous. The raw `mcl_…` secret is returned only on create/regenerate. Do not delete the key bound to your active MCP session.

---

## Resources

Read-only context fetched via MCP `resources/read` (or auto-loaded by some clients). They do not execute API calls or charge credits.

| URI | MIME | Content |
|---|---|---|
| `modelclone://v1/route-catalog` | `application/json` | Allowed integrator routes with rate-limit buckets and `sessionOnlyNamespaces`. |
| `modelclone://v1/openapi` | `text/yaml` | Full OpenAPI 3 contract — request/response schemas for every route. |
| `modelclone://v1/base` | `text/plain` | REST base URL and auth header hints. |
| `modelclone://v1/getting-started` | `text/markdown` | Workflow guide and recommended tool sequencing. |

### When to read resources vs tools

| Need | Use |
|---|---|
| Confirm auth + credits | `get_me` tool (live state) |
| Live credit costs | `get_pricing_generation` tool |
| Run a known action | Typed tool (`mcx_generate`, `flows_run`, `gallery_feed`, `confirm_adult`, …) |
| Poll async job | `wait_for_generation` or feature status tool (`avatars_video_status`, `img2img_status`, …) |
| Check if a path is allowed | Resource `route-catalog` |
| Learn JSON body field names/types | Resource `openapi` |
| Long tail (JWT signup, Fanvue OAuth start, web push tokens, API key CRUD) | `api_v1_request` after reading catalog + openapi |
| Real-time flow SSE | Direct HTTP `GET /flows/runs/:runId/stream` — not available as an MCP tool |

Do not load the entire OpenAPI spec on every turn — fetch it once when building an unfamiliar payload.

---

## Example workflows

Full step-by-step recipes with JSON payloads: **`docs/mcp/sections/13-recipes.md`**. The MCP resource `modelclone://v1/getting-started` mirrors the same sequencing.

| Goal | Tool chain |
|------|------------|
| Model + SFW image | `get_me` → wizard or classic model tools → `models_status` → `generate_recreate` → `wait_for_generation` |
| NSFW video preset session | `nsfw_video_presets` → `nsfw_video_create_session` → poll `nsfw_video_get_session` → `nsfw_video_session_action` (`select-preview` → optional `edit-frame` → `approve` → `submit`) → `wait_for_generation(finalGenerationId)` — detail in [NSFW Video Sessions](./15-nsfw-video.md) and Recipe D3 in `13-recipes.md` |
| Flow Studio pipeline | `flows_node_types` → `flows_create` → `flows_run` → `flows_run_status` |
| Route without a typed tool | `api_v1_request` after reading `modelclone://v1/route-catalog` and `modelclone://v1/openapi` |

---

## Limits and behavior

- **Same credits and rate limits as REST.** Generation submits are metered per API key into capacity buckets (`standard` 30/min, `advanced` 30/min, `dedicated` 60/min, `cinematic` 30/min, `repurposer` 100/min). A bucket overflow surfaces as an error result on that tool call — back off and retry.
- **One tool call = one REST call.** Nothing is cached server-side.
- **Session-only, not available via MCP/API key:** Stripe/crypto billing, referrals, DSAR, and admin operations. These require a signed-in web session.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `401` — "Missing API key. Send X-Api-Key: mcl_… or Authorization: Bearer mcl_…" | No key on the request | Add the header. In Claude connectors, paste the key into the auth field. |
| `401` — "Invalid or revoked API key" | Key is wrong, disabled, or revoked | Generate a fresh key in **Settings → API**. |
| `403` — "Session bound to a different API key" | A session id was reused with a different key | Start a fresh session (reconnect) using the correct key. |
| `410` — `API_V1_SUNSET` (with a `v2BaseUrl`) | The v1 API-key surface has been sunset in favor of v2 | Point your client at the `v2BaseUrl` from the error payload. |
| Tool returns `{ "ok": false, "error": … }` | The underlying REST call failed (validation, credits, rate limit) | Read the `error` message; check credits with `get_me` and costs with `get_pricing_generation`. |
| Session silently stops working after inactivity | Idle sessions expire after 30 minutes | Reconnect — state is preserved server-side. |
| "Route not on integrator product surface" from `api_v1_request` | The path isn't part of the integrator product surface (e.g. billing/admin) | Use only paths listed in `modelclone://v1/route-catalog`. |

> **V1 sunset notice:** when v2 launches, MCP and `/api/v1` API-key traffic may return `410 API_V1_SUNSET` with a `v2BaseUrl` pointer. Watch the API changelog for the cutover date.
