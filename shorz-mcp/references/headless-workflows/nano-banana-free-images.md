# Free Nano Banana 2 images — `generate_images_nano_banana_free`

Generate images with Google's Nano Banana 2 models **without spending Shorz credits**. The tool
calls the Google AI Studio API (`generativelanguage.googleapis.com`) directly with the **user's own
AI Studio key**, so it consumes that key's Google free-tier quota instead of the Shorz credit
proxy.

No project, no `projectType`, no panels, no Create Video. Shorz must still be **running** — the app
writes the finished file into the Generated Images library.

## The two models are MCP-only and are NOT the paid ones

| MCP `imageModel` | Google model | Sizes | Search grounding | Billing |
|---|---|---|---|---|
| `nano-banana-2-free` (default) | `gemini-3.1-flash-image` | `1K`, `2K`, `4K` | yes (`enableWebSearch`) | free — user's Google quota |
| `nano-banana-2-lite-free` | `gemini-3.1-flash-lite-image` | `1K` only | no | free — user's Google quota |

The `-free` suffix is load-bearing. Shorz **also** ships paid, credit-metered models with almost the
same names (`Nano Banana 2` / `nano-banana-2`, `nano-banana-2-lite`) routed through AIML — those are
what `generate_images`, `generate_scene_image`, the Text-to-Video / AI B-roll / Thumbnail Creator
pickers and the Python stack use. Rules:

- **Never** pass `nano-banana-2-free` / `nano-banana-2-lite-free` to any other tool. They are not
  valid values for `set_text_to_video_settings`, `set_broll_settings`,
  `set_thumbnail_creator_settings`, `generate_images`, or `generate_scene_image`, and they appear in
  no in-app model picker.
- **Never** pass `nano-banana-2` / `gpt-image-2` to this tool — its enum rejects them.
- When the user asks for "a free image" or says they don't want to spend credits, this is the tool.
  When they want the image inside the app's normal paid pipeline, use `generate_images` /
  `generate_scene_image`.

## Prerequisite: the user's Google AI Studio key

Key lookup order, first non-empty wins:

1. env `SHORZ_GOOGLE_AI_STUDIO_API_KEY`
2. env `GOOGLE_AI_STUDIO_API_KEY`
3. env `GEMINI_API_KEY`
4. env `GOOGLE_API_KEY`
5. the single-line file `%LOCALAPPDATA%\Shorz\config\google_ai_studio_key.txt` (macOS/Linux: the
   same `config/` folder under the Shorz data root). It accepts a bare key or `{"key":"AIza..."}`.

Without a key the tool fails with `success:false` and an error naming all four env vars, the exact
file path, and <https://aistudio.google.com/apikey>. Do **not** ask the user to paste the key into
chat — tell them to set the env var in their MCP client config (`"env": { "GEMINI_API_KEY": "…" }`)
or save it to that file. The response reports `keySource` (the env var name or file path), never
the key itself.

## Arguments

| Arg | Notes |
|---|---|
| `description` | Required. The prompt. With `referenceImages` this describes the **scene/edit**, not the reference subject. |
| `imageModel` | `nano-banana-2-free` (default) or `nano-banana-2-lite-free`. |
| `aspectRatio` | One of 14: `1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9`, `1:4`, `4:1`, `1:8`, `8:1`. Default `16:9`. |
| `imageSize` | `1K` (default), `2K`, `4K`. **`2K`/`4K` are rejected for `nano-banana-2-lite-free`** with a clear error — Lite renders 1024px only. |
| `numVariations` | 1–4 (clamped). Each variation is a **separate request** against the free-tier quota. |
| `referenceImages` | Up to 5 absolute local paths, `local-resource://`, `data:` or http(s) URLs. Sent as inline bytes for editing an existing image, keeping the same face/subject, or composing several subjects. Unreadable refs become `warnings`, not a failure. |
| `enableWebSearch` | Ground the generation in Google Search. `nano-banana-2-free` only — ignored with a warning on Lite. Default `false`. |
| `fileNamePrefix` | Output filename prefix. Default `nano-banana-free`. |
| `awaitCompletion` | Async by default (see below). |

## Async by default

Returns `{ started, jobId }` immediately — poll `get_job_status { jobId }` until `lastStatus` is
`completed` (with `outputPaths`) or `error`. Pass `awaitCompletion: true` only if your tool-call
timeout exceeds ~8 minutes. Re-calling after a slow poll starts a **second** generation and burns
more of the free-tier quota, so poll instead.

## Result

```json
{
  "success": true,
  "provider": "google-ai-studio",
  "apiSurface": "interactions",
  "keySource": "env:GEMINI_API_KEY",
  "creditsSpent": 0,
  "requested": { "imageModel": "nano-banana-2-free", "googleModel": "gemini-3.1-flash-image", "aspectRatio": "16:9", "imageSize": "1K", "…": "…" },
  "savedLocalFilePaths": ["C:\\…\\Shorz\\Cache\\Generated_Images\\…png"],
  "generated": [{ "mimeType": "image/png", "approxBytes": 1284915 }],
  "modelNotes": [], "failures": [], "warnings": []
}
```

`savedLocalFilePaths` holds the absolute paths — hand those to `import_frontend_assets`,
`save_file_as`, or a project settings patch if the user wants the image inside a project (a
**separate** step). `generated` carries metadata only; raw image bytes are never returned.

## Failure modes worth recognising

- **HTTP 429** — the key's Google free-tier quota (per-minute or per-day) is exhausted. Options:
  wait for the window to reset, switch to `nano-banana-2-lite-free` (its own, larger quota), or use
  the credit-metered `generate_images`. Do not retry in a loop.
- **HTTP 401 / 403** — key rejected; the user should check the key and that the Generative Language
  API is enabled for it.
- **No image returned** — the error includes the block reason (e.g. `SAFETY`, `IMAGE_SAFETY`) when
  Google supplies one. Rephrase the prompt rather than retrying verbatim.
- **`"Shorz is not running"`** — the image generated but could not be saved. Start Shorz and retry.
- Partial success with `numVariations > 1` is normal: successful images are saved and the failed
  variations are listed in `failures`.
