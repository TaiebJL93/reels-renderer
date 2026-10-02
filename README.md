# reels-renderer

Free, serverless renderer for short vertical videos (Instagram Reels, TikTok, Shorts).

A GitHub Actions job runs [MoneyPrinterTurbo](https://github.com/harry0703/MoneyPrinterTurbo) on a GitHub-hosted runner, turns a script into a 1080x1920 MP4 (stock footage, AI voice-over, word-by-word subtitles, background music, on-screen hook headline), uploads it to Cloudinary and posts the link back to whoever started the job. No server to pay for or keep running.

It is built to be driven by an automation tool such as [n8n](https://n8n.io), but anything that can send an HTTP request can start it.

## How it works

```
n8n (or any caller)
  │  1. POST workflow_dispatch  (job_id, request, callback_url)
  ▼
GitHub Actions: "Render reel"
  2. start MoneyPrinterTurbo in Docker
  3. render the video (Pixabay clips + Edge TTS voice + subtitles + music)
  4. add the hook headline for the first 3 s, re-encode at 30 fps, check 1080x1920
  5. upload the MP4 to Cloudinary
  │  6. POST result to callback_url
  ▼
n8n continues (saves the link, posts the reel later)
```

One render takes about 3 to 6 minutes. Only one render runs at a time (GitHub keeps at most one more waiting), so start renders a few minutes apart.

## What it costs

Nothing, within the free tiers:

| Service | Used for | Free tier |
|---|---|---|
| GitHub Actions | running MoneyPrinterTurbo | unlimited minutes on public repos, 2,000 min/month on private repos |
| Pixabay API | stock footage | free API key |
| Microsoft Edge TTS | voice-over | free, no key (unofficial, could change) |
| Cloudinary | hosting the finished MP4 | free plan (25 monthly credits) |

## Setup

### 1. Get the keys
- **Pixabay:** sign up at pixabay.com, then open https://pixabay.com/api/docs/ while logged in. Your key is shown under *Parameters*.
- **Cloudinary:** sign up at cloudinary.com (free plan). Under **Settings → API Keys**, use a key with full access (it must be allowed to upload, and to delete if you clean up after posting). Note the cloud name, API key and API secret.

### 2. Add the repository secrets
**Settings → Secrets and variables → Actions → New repository secret**, names exactly as below:

| Secret | Value |
|---|---|
| `PIXABAY_API_KEY` | your Pixabay key |
| `CLOUDINARY_CLOUD_NAME` | your Cloudinary cloud name |
| `CLOUDINARY_API_KEY` | your Cloudinary API key |
| `CLOUDINARY_API_SECRET` | your Cloudinary API secret |
| `PEXELS_API_KEY` | optional, only if you use `"video_source": "pexels"` |

Secrets are never shown in the logs.

### 3. Create a token for the caller
**GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token**
- Repository access: only this repository
- Permissions: **Actions: Read and write** (nothing else)

The caller sends it as the header `Authorization: Bearer <token>`.

## Starting a render

```
POST https://api.github.com/repos/<owner>/reels-renderer/actions/workflows/render.yml/dispatches
Authorization: Bearer <token>
Accept: application/vnd.github+json
X-GitHub-Api-Version: 2022-11-28

{
  "ref": "<default branch, e.g. main or master>",
  "inputs": {
    "job_id": "R001-123",
    "request": "<MoneyPrinterTurbo request body as a JSON string>",
    "callback_url": "https://your-automation/resume-url"
  }
}
```

GitHub answers `204 No Content` when the job is queued.

| Input | Meaning |
|---|---|
| `job_id` | Letters, digits, `-` and `_` only. Used as the run name and the Cloudinary file name (`reels/<job_id>`). |
| `request` | The body of MoneyPrinterTurbo's `POST /api/v1/videos` as a JSON string. One extra field, `overlay_hook`, is the headline shown for the first 3 seconds; it is removed before MoneyPrinterTurbo sees the request. |
| `callback_url` | Receives the result as a JSON POST. In n8n, use the Wait node's `$execution.resumeUrl`. |

Example `request`:

```json
{
  "video_subject": "Discipline beats motivation",
  "video_script": "You won't feel like it. Do it anyway. ...",
  "video_terms": ["sunrise", "running", "city night", "gym workout"],
  "video_aspect": "9:16",
  "video_concat_mode": "sequential",
  "video_transition_mode": null,
  "video_clip_duration": 2,
  "video_count": 1,
  "video_source": "pixabay",
  "voice_name": "en-US-AndrewMultilingualNeural-Male",
  "voice_rate": 1.0,
  "bgm_type": "random",
  "bgm_volume": 0.15,
  "subtitle_enabled": true,
  "subtitle_position": "custom",
  "custom_position": 60,
  "subtitle_display_mode": "word_by_word",
  "font_name": "MicrosoftYaHeiBold.ttc",
  "font_size": 90,
  "text_fore_color": "#FFFFFF",
  "stroke_color": "#000000",
  "stroke_width": 4,
  "overlay_hook": "You won't feel like it. Do it anyway."
}
```

The video is as long as the voice-over: about 2.5 words per second, so 60 to 80 words gives a 24 to 32 second reel.

### Result sent to `callback_url`

Success:
```json
{ "ok": true, "video_url": "https://res.cloudinary.com/<cloud>/video/upload/v.../reels/R001-123.mp4", "job_id": "R001-123", "run_url": "https://github.com/..." }
```

Failure:
```json
{ "ok": false, "error": "<reason> (log: <run url>)", "job_id": "R001-123" }
```

A cover image can be taken from any frame by adding a Cloudinary transformation to the video link, for example `/video/upload/so_1.5/...jpg` for the frame at 1.5 s.

## Troubleshooting

| Error | Fix |
|---|---|
| `401 Requires authentication` | The caller sent no token. The header must be `Authorization: Bearer <token>`. |
| `401 Bad credentials` | The token is wrong or expired. Create a new one. |
| `403 Resource not accessible` | The token needs **Actions: Read and write**. |
| `404 Not Found` | Wrong repo name, the token doesn't include this repo, or `render.yml` isn't on the branch you sent as `ref`. |
| `422 No ref found` | `ref` must be your default branch (`main` or `master`). |
| `Missing repository secret ...` | Add that secret. Names must match exactly. |
| `Cloudinary upload failed ... missing permissions` | The Cloudinary API key can't upload. Use a key with full access and update both key secrets. |
| `MoneyPrinterTurbo could not make the video` | Open the log link. Usually the Pixabay key, or no vertical clips for a very specific search term; use broader terms. |

## Good to know
- **Logs are public on a public repo.** The script and the callback URL appear in each run's log (secrets are masked). Make the repository private if that matters to you.
- Each run downloads the MoneyPrinterTurbo Docker image, so the first step takes a minute or two.
- Credits: video engine by [harry0703/MoneyPrinterTurbo](https://github.com/harry0703/MoneyPrinterTurbo) (MIT). Footage from Pixabay; check its license for your use.
