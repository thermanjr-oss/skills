---
name: youtube-video-generator
description: Turn a topic into a YouTube video end to end (AI script, stock visuals, TTS voiceover, FFmpeg assembly, YouTube upload) by following the pipeline of the open-source MohamedElaassal/youtubeVideoGenerator Laravel app. Use when asked to build, extend, or debug an automated YouTube video generation pipeline. Only runs when explicitly invoked; it does not trigger on its own.
disable-model-invocation: true
---

# YouTube Video Generator

A pipeline recipe based on [MohamedElaassal/youtubeVideoGenerator](https://github.com/MohamedElaassal/youtubeVideoGenerator) (MIT), a Laravel 12 + Vue 3 / Inertia SaaS that automates topic → published video. This skill does not vendor that code. Clone the repo when you need the real implementation.

## Pipeline

Each video is a `videos` row that moves through a status enum. Advance one stage at a time and write the status before starting the next stage, so a failure leaves a resumable state.

| Stage | Status | What happens |
|---|---|---|
| 1. Script | `generating_script` | LLM (Gemini in the code, OpenAI in the README) writes a conversational script: hook, 3-5 points, conclusion with CTA. Input: `topic`, `duration` in minutes. |
| 2. Visuals | `fetching_visuals` | Pixabay videos (min 20s, `safesearch`) plus Unsplash landscape photos for the topic. |
| 3. Voiceover | `generating_voiceover` | Text-to-speech on the script (`scripts/gtts_client.py`, gTTS by default). |
| 4. Assembly | `assembling_video` | `scripts/create_video.py` downloads assets and combines them with FFmpeg into one MP4. |
| 5. Ready | `ready` | Final path and thumbnail are stored. A human can review before publishing. |
| 6. Upload | `uploading` → `uploaded` | YouTube Data API v3 using the user's stored OAuth token, refreshed when expired. Save `youtube_video_id`. |
| Error | `failed` | Store the reason in `failure_reason`. |

## Architecture to copy

- **Laravel owns every external API call and secret.** n8n (or any orchestrator) only calls Laravel webhook endpoints and never holds keys.
- **Webhook endpoints** (`routes/api.php`): `GET videos/{id}/details`, `POST videos/{id}/status`, `POST videos/{id}/generate-script`, `POST videos/{id}/fetch-media`, `POST videos/{id}/upload-youtube`, `POST videos/{id}/youtube-uploaded`, `GET autopilot/topics`, `POST video-requests`.
- **Autopilot:** a channel has a niche and schedule. The orchestrator polls `autopilot/topics` and creates `video-requests` for due topics.
- **Long work goes on queues** (Redis). Never assemble video inside a web request.
- **Google OAuth** doubles as login and YouTube channel connection. Tokens are stored per user.
- **Quota:** a `CheckVideoQuota` middleware gates video creation per user.

## Setup checklist

1. `cp .env.example .env` and fill `GEMINI_API_KEY` (or OpenAI), `PIXABAY_API_KEY`, `UNSPLASH_ACCESS_KEY`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`.
2. In Google Cloud Console, enable YouTube Data API v3 and add the OAuth redirect `…/auth/google/callback`.
3. `docker-compose up -d --build`, then `composer install`, `npm install --legacy-peer-deps`, `php artisan key:generate`, `php artisan migrate`, `npm run build`.
4. Python deps: `scripts/install_dependencies.sh` (needs FFmpeg).
5. Import the n8n workflow and point it at `LARAVEL_APP_URL`.

## Known gaps in the source repo

Check these before relying on it:

- The README says Pexels and `workflows/complete_video_generator.json`. The code uses **Pixabay + Unsplash**, and the **workflow file and `app/Services`/`app/Jobs` directories are not in the repo**. You must write the n8n workflow and the queue job yourself.
- The `.env.example` ships a real-looking `APP_KEY`. Generate your own.
- The `/api/videos/*` webhook routes have no auth middleware. Add a shared-secret header or Sanctum before exposing them.
- The script prompt is a single call to `gemini-pro`. For longer videos, generate per-section and stitch.

## Working rules

- Keep each stage idempotent: re-running a stage overwrites its own output only.
- Verify with a short (1-2 minute) test topic before enabling autopilot.
- Never commit API keys, OAuth tokens, or generated media.
