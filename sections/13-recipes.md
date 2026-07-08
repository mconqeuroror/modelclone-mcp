## Recipe A — Verify account

**Tools:** `get_me` → `get_pricing_generation`

1. `get_me` — confirm email, `totalCredits`, `subscriptionTier`.
2. `get_pricing_generation` — cache costs for later submits.

No credits charged.

---

## Recipe B — Creator Studio image (typed tool)

**Tools:** `creator_studio_image` → `wait_for_generation`

**Submit:**

```json
{
  "body": {
    "prompt": "minimal product shot, white background",
    "generationModel": "nano-banana-pro",
    "aspectRatio": "1:1",
    "resolution": "2K",
    "numImages": 1
  }
}
```

**Poll:** `wait_for_generation` with `body.generation.id` from the tool result (typically 30–90 s).

Credits: `creatorStudio1K2K` default **15**/image at 2K — confirm via `get_pricing_generation`.

---

## Recipe B2 — Recreate a reference photo

**Tools:** `generate_recreate` → `wait_for_generation`

**Prerequisites:** model with 3 reference photos (`get_model`).

```json
{
  "modelId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "options": {
    "sourceImageUrl": "https://cdn.example.com/inspo/rooftop-pose.jpg",
    "outfitMode": "model",
    "extraGuidance": "golden hour warmth"
  }
}
```

Poll `body.generation.id`. Credits: `recreateImage` default **10**/image (or **16** with `genModel: "nano-banana-pro"`).

---

## Recipe B3 — Free prompt with aspect ratio

**Tools:** `generate_free` → `wait_for_generation`

```json
{
  "prompt": "candid mirror selfie, soft morning light",
  "options": {
    "modelId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "genModel": "nano-banana-pro",
    "aspectRatio": "9:16",
    "resolution": "2K",
    "enhance": true
  }
}
```

When `enhance: true`, model gender must be set or the API returns `MODEL_GENDER_REQUIRED`.

---

## Recipe B4 — Motion video from still + driving clip

**Tools:** `generate_motion_video` → `wait_for_generation`

```json
{
  "body": {
    "modelId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "imageUrl": "https://cdn.modelclone.app/generations/still.png",
    "videoUrl": "https://cdn.example.com/uploads/dance.mp4",
    "duration": 8,
    "prompt": "natural cinematic motion"
  }
}
```

Poll `body.generationId`. Credits: `motionXPerSec` default **9.5**/s × duration (8 s → **76** credits).

---

## Recipe C — ModelClone-X txt2img

**Tools:** `mcx_generate` → `wait_for_generation`

**Submit:**

```json
{
  "body": {
    "prompt": "portrait photo, neutral background",
    "aspectRatio": "1:1",
    "qty": 1,
    "preOptimized": true,
    "useCustomPrompt": true
  }
}
```

**Poll:** `generationIds[0]` from response → `wait_for_generation`.

Typical time: 2–5 minutes.

---

## Recipe D — NSFW image (classic LoRA)

**Prerequisites:** trained LoRA (`nsfw_training_status` → `status: "ready"`, `nsfwUnlocked: true`). LoRA setup via `api_v1_request` — [NSFW Studio](../../public-api/14-nsfw.md#lora-training).

**Tools:** `nsfw_generate` → `wait_for_generation`

```json
{
  "modelId": "<uuid>",
  "prompt": "<triggerWord> scene description…",
  "options": {
    "quantity": 1,
    "resolution": "768x1344",
    "integrationCallbackUrl": "https://api.example.com/hooks/modelclone",
    "integratorWebhookSecret": "whsec_…"
  }
}
```

Poll `generation.id`. Typical time: 1–3 minutes. Default **30** credits ( **50** for `quantity: 2`).

---

## Recipe D2 — NSFW v2 preset still

**Prerequisites:** NSFW-verified model with three reference photos.

**Tools:** `api_v1_request` (`POST /nsfw-v2/presets`) → `wait_for_generation` (per id)

```json
{
  "body": {
    "modelId": "<uuid>",
    "presetId": "lt_01_black_lace_bed",
    "aspectRatio": "9:16",
    "count": 1
  }
}
```

Preset catalog: 210 ids — category prefixes in [NSFW Studio](../../public-api/14-nsfw.md#preset-stills). Default **6** credits/image.

---

## Recipe D3 — NSFW video preset session (preview → select → render)

**Tools:** `nsfw_video_presets` → `nsfw_video_create_session` → `nsfw_video_get_session` (poll) → `nsfw_video_session_action`

**1. List presets** — call `nsfw_video_presets`; save a preset **`id`** (not `key`) and `durationSeconds`.

**2. Create session**

```json
{
  "body": {
    "modelId": "<uuid>",
    "mode": "preset",
    "presetId": "cpre_01hxyz"
  }
}
```

**3. Poll previews** — `nsfw_video_get_session` every 3–5 s until `previewImageUrls.length === 3`.

**4. Select preview**

```json
{
  "sessionId": "cnvs_01hxyz",
  "action": "select-preview",
  "body": { "previewUrl": "https://cdn.modelclone.app/…/preview-1.png" }
}
```

**5. (Optional) Edit frame** — 10 credits; poll session until `currentFrameUrl` updates:

```json
{
  "sessionId": "cnvs_01hxyz",
  "action": "edit-frame",
  "body": { "prompt": "remove the necklace, keep everything else identical" }
}
```

**6. Approve → submit**

```json
{ "sessionId": "cnvs_01hxyz", "action": "approve", "body": {} }
```

```json
{ "sessionId": "cnvs_01hxyz", "action": "submit", "body": {} }
```

**7. Final poll** — `nsfw_video_get_session` every 5–10 s until `status === "completed"` → `wait_for_generation(finalGenerationId)`.

Submit cost: `ceil(sourceVideoDurationSeconds × 31.25)` credits. First preview batch is **free**; regenerate previews **20** credits.

**Sanitization:** pipeline generation rows return `prompt: null` over API key — rely on session state and `outputUrl`.

Full REST detail: [NSFW Video Sessions](../../public-api/15-nsfw-video.md).

---

## Recipe E — Enhance prompt then free generate

**Tools:** `enhance_prompt` (sync) → `generate_free` → `wait_for_generation`

**Step 1 — enhance (5 credits default):**

```json
{
  "prompt": "sunset rooftop portrait",
  "options": {
    "mode": "casual",
    "genModel": "nano-banana-pro",
    "modelLooks": { "gender": "female" }
  }
}
```

**Step 2 — generate with enhanced text (`enhance: false` to avoid double-charging):**

```json
{
  "prompt": "<body.enhancedPrompt from step 1>",
  "options": {
    "modelId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "genModel": "nano-banana-pro",
    "aspectRatio": "3:4",
    "enhance": false
  }
}
```


## Recipe F — Create a model end-to-end (wizard niche path)

**Tools:** `get_me` → `list_models` → `wizard_look_variants` → `wizard_preview_images` → `wizard_finalize_poses` → `models_status` → `get_model`

**Prerequisites:** `canCreateMore: true` from `list_models`. Wizard path is **free** (no credits on look variants, previews, or finalize).

### 1. Confirm account

```json
{}
```

Call `get_me` — note `totalCredits` (only needed if you later use the classic paid path).

### 2. Check model slot

Call `list_models` with no parameters. Abort if `canCreateMore` is false.

### 3. Generate look variants

```json
{
  "body": {
    "gender": "female",
    "age": 24,
    "nicheName": "Fitness",
    "ethnicity": "Latina"
  }
}
```

Tool: `wizard_look_variants`. Pick one entry from `variants[]` (e.g. label `"Soft"`).

### 4. Render preview images

```json
{
  "body": {
    "gender": "female",
    "age": 24,
    "nicheName": "Fitness",
    "variants": [
      { "label": "Soft", "looks": { "gender": "female", "ethnicity": "Latina", "hairColor": "Dark Brown", "bodyType": "Athletic" } }
    ]
  }
}
```

Tool: `wizard_preview_images`. Save `previews[0].referenceUrl` and `previews[0].looks`.

### 5. Finalize poses (async)

```json
{
  "body": {
    "name": "FitCreator24",
    "referenceUrl": "https://cdn.modelclone.app/references/prev-1.jpg",
    "gender": "female",
    "age": 24,
    "ethnicity": "Latina",
    "hairColor": "Dark Brown",
    "bodyType": "Athletic"
  }
}
```

Tool: `wizard_finalize_poses`. Read `model.id` from the 202 response.

### 6. Poll until ready

```json
{ "id": "550e8400-e29b-41d4-a716-446655440000" }
```

Tool: `models_status` every **3–5s** until `status === "ready"` (or `"failed"`).

Typical time: 1–3 minutes.

### 7. Verify

```json
{ "modelId": "550e8400-e29b-41d4-a716-446655440000" }
```

Tool: `get_model` — confirm all three `photo*Url` fields and `status: "ready"`. Use `model.id` in `generate_recreate` and other generation tools.

---

### Alternate paths (same poll target)

| Path | Submit sequence | Credits (defaults) |
|------|-----------------|-------------------|
| **Custom description** | `wizard_custom_reference` → `wizard_finalize_poses` | 0 (+ 10 per `regenerate: true` on custom reference) |
| **Upload photos** | Upload 3 URLs → `wizard_upload_save` | 0 |
| **Classic two-step** | `models_generate_reference` (150) → `models_generate_poses` (750) | 900 total |

All async paths poll `models_status` with the returned `model.id`.

---

## Pacing (avoid 429)

- Wait **6+ seconds** between POST submits on the same account.
- Poll every **10–15s**, not sub-second.
- One async job at a time when testing.
