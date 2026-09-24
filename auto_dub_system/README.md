# 🎙️ Super AutoDub — AI-Powered Video Dubbing System

> Automatically dub any video into 35+ languages using speaker diarization, gender-aware TTS, and a fully async pipeline — powered by Sarvam AI, Celery, and FastAPI.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Pipeline Stages](#pipeline-stages)
- [Tech Stack](#tech-stack)
- [Installation](#installation)
- [Configuration](#configuration)
- [Running the System](#running-the-system)
- [API Reference](#api-reference)
- [Frontend](#frontend)
- [Docker](#docker)

---

## Overview

**Super AutoDub** is a production-grade, end-to-end automatic video dubbing system. Upload a video (or paste a YouTube URL), choose a target language, and the system will:

1. Separate vocals from background audio
2. Diarize speakers (who spoke when)
3. Transcribe speech-to-text (Sarvam AI STT)
4. Translate and align text
5. Generate gender-aware TTS audio (Sarvam Bulbul:v3)
6. Re-mix and synchronize dubbed audio back into the final video

---

## 🔍 How It Works

Here is the complete journey of a video through Super AutoDub — from upload to dubbed output.

---

### Step 1 — You Submit a Video

You can either:
- **Upload a local video file** via the Input Page, or
- **Paste a YouTube URL** and let the system download it automatically using `yt-dlp`

Along with the video you choose:
- **Source language** (or leave as `auto` for auto-detection)
- **Target language** (e.g. Hindi → English)
- **Gender hint** (Male / Female, as a fallback for speaker gender)

The backend saves the file to `data/uploads/` and immediately returns a **`task_id`** — the rest of the work happens asynchronously in the background.

---

### Step 2 — Audio Extraction & Speaker Identification *(Stage 1)*

The pipeline kicks off a **Celery chain** of 5 tasks in sequence:

```
audio_separator → diarization → overlap_split → segment → chunk
```

| What happens | How |
|---|---|
| The video's audio is extracted and split into **vocals** and **background noise** | `spleeter` source separation |
| Each moment of speech is tagged with a **speaker label** (e.g. `SPEAKER_00`, `SPEAKER_01`) and timestamp | `pyannote.audio` diarization |
| Overlapping speech regions are detected and isolated | `overlap_detector.py` |
| The vocal track is split into **one audio file per speaker segment** | `segment_separation.py` |
| Each segment is further broken into **smaller chunks** suitable for STT | `chunker.py` |

At the end of Stage 1, you have a list of audio chunks, each tagged with: speaker ID, start/end timestamp, and overlap flag.

---

### Step 3 — Gender Detection via Majority Vote *(Stage 2 — Setup)*

Before processing each chunk independently, the system runs **gender detection on every chunk** for every speaker and tallies votes:

```
Speaker_00: [male, male, female, male] → final gender = male
Speaker_01: [female, female, female]   → final gender = female
```

This majority-vote approach ensures all chunks for one speaker get the **same voice** in TTS — even if individual chunks are ambiguous. The detection uses an **XGBoost model + SpeechBrain embeddings + librosa features**.

---

### Step 4 — Parallel STT → Translation → TTS *(Stage 2 — Chunk Processing)*

All chunks are now processed **in parallel** using a Celery **chord** (parallel group with a callback). Each chunk goes through its own mini-pipeline:

```
┌─────────────────────────────────────────────────────────────────┐
│  chunk_0: gender → STT → translate/align → TTS                  │
│  chunk_1: gender → STT → translate/align → TTS   (parallel)     │
│  chunk_2: gender → STT → translate/align → TTS                  │
│  ...                                                             │
└─────────────────────────────────────────────────────────────────┘
```

| Sub-step | What happens | Service |
|---|---|---|
| **Gender** | Uses pre-assigned speaker gender (skips re-detection) | `gender_detection.py` |
| **STT** | Converts speech audio → text in the source language | Sarvam AI STT API |
| **Align** | Translates text to target language, aligns to original timestamps | `translation_matcher.py` |
| **TTS** | Converts translated text → natural speech audio using the assigned voice | Sarvam Bulbul:v3 API |

Each speaker is assigned a **deterministic voice** from the Bulbul:v3 roster (20 male, 16 female) — the same speaker always gets the same voice, even across multiple runs.

---

### Step 5 — Audio Assembly *(Stage 3)*

Once all chunks are done, Stage 3 assembles the final audio:

1. A **silent canvas** is created matching the exact duration of the original video
2. If background noise was separated, it is mixed in at **-3 dB** (quieter, so speech is clear)
3. Every TTS audio clip is **overlaid at its exact timestamp** in milliseconds using `pydub`
4. Overlapping TTS segments are resolved — the **longer segment wins**
5. The final mixed audio is exported as `data/outputs/final_audio.wav`

This approach (absolute-position mixing) avoids audio sync drift and eliminates ffmpeg "Non-monotonic DTS" errors.

---

### Step 6 — Final Video Output *(Stage 4)*

The last stage:

1. **Muxes** the dubbed `final_audio.wav` with the original video using `ffmpeg` (replaces the audio track)
2. Saves the result as a uniquely named file: `{original_name}_{uid}_dubbed.mp4`
3. **Optionally uploads** to AWS S3 and returns a **presigned URL** (valid 1 hour) for download
4. If S3 is not configured, the file is served directly via `GET /download/{filename}`

---

### Step 7 — You Watch & Download

Back in the browser:
- The frontend **polls** `GET /task/{task_id}` until status is `SUCCESS`
- The **Preview Page** shows the dubbed video in a player
- You can **download** the final dubbed MP4 with one click

---

### Full Flow Summary

```
You                →  Upload video + choose language
FastAPI            →  Save file, return task_id
Celery Stage 1     →  Separate vocals, diarize speakers, chunk audio
Celery Stage 2     →  Gender vote, STT, translate, TTS (parallel per chunk)
Celery Stage 3     →  Mix all TTS chunks onto a timeline canvas
Celery Stage 4     →  Mux into video, upload to S3 (optional)
Frontend           →  Poll task status → show dubbed video → download
```

---

## ✨ Features

- 🎬 **Upload or YouTube** — upload a local video or provide a YouTube URL
- 🗣️ **Multi-Speaker Support** — handles multiple speakers via pyannote diarization
- 🧠 **Gender Detection** — XGBoost + SpeechBrain model with majority-vote per speaker
- 🌐 **35+ Languages** — English, Hindi, Tamil, Telugu, Kannada, Bengali, Malayalam, Marathi, Spanish, French, German, Japanese, Korean, Arabic, and more
- 🔊 **Bulbul:v3 Voice Roster** — 20 male + 16 female natural-sounding voices
- 📦 **Async Task Queue** — Celery + Redis for non-blocking, parallel chunk processing
- ☁️ **AWS S3 Integration** — auto-uploads dubbed video and returns a presigned download URL
- 🖥️ **React Frontend** — modern UI with Landing Page, Input Page, and Preview/Download Page

---

## 🏗️ Architecture

```
User (Browser)
     │
     ▼
React Frontend (Vite + React 19 + TailwindCSS v4)
     │  REST API
     ▼
FastAPI Backend (port 8000)
     │
     ├── POST /upload
     ├── POST /dub_video
     ├── POST /youtube/info
     ├── POST /youtube/download
     ├── GET  /task/{task_id}
     └── GET  /download/{filename}
                  │
                  ▼ Celery Chain (via Redis)
     ┌────────────────────────────────────┐
     │  Stage 1: Audio Preparation         │
     │  ├─ audio_separator                 │
     │  ├─ diarization                     │
     │  ├─ overlap_split                   │
     │  ├─ segment                         │
     │  └─ chunk                           │
     ├────────────────────────────────────┤
     │  Stage 2: Per-Chunk (parallel)      │
     │  gender → STT → align → TTS         │
     ├────────────────────────────────────┤
     │  Stage 3: Audio Assembly (pydub)    │
     ├────────────────────────────────────┤
     │  Stage 4: Video Mux + S3 Upload     │
     └────────────────────────────────────┘
```

---

## 📁 Project Structure

```
Super_Clean/
├── README.md
├── requirements_venv311.txt
└── auto_dub_system/
    ├── .env                         # API keys and secrets (not committed)
    ├── Dockerfile
    ├── docker-compose.yml
    ├── run_pipeline.bat             # One-click Windows launcher
    ├── requirements.txt
    │
    ├── app/
    │   ├── main.py                  # FastAPI app entry point
    │   ├── config.py                # Settings + Bulbul voice roster
    │   │
    │   ├── api/
    │   │   ├── routes.py            # All REST API endpoints
    │   │   └── schemas.py           # Pydantic request schemas
    │   │
    │   ├── services/                # Core business logic
    │   │   ├── audio_extractor.py   # Vocal / background separation
    │   │   ├── chunker.py           # Audio chunking + final video mux
    │   │   ├── diarization.py       # Speaker diarization (pyannote)
    │   │   ├── gender_detection.py  # XGBoost gender classifier
    │   │   ├── language_detect.py   # Language auto-detection
    │   │   ├── overlap_detector.py  # Overlapping speech detection
    │   │   ├── s3_storage.py        # AWS S3 upload + presigned URL
    │   │   ├── segment_separation.py# Per-speaker segment splitting
    │   │   ├── stt.py               # Sarvam AI Speech-to-Text
    │   │   ├── translation_matcher.py # Text translation + alignment
    │   │   ├── tts.py               # Sarvam Bulbul:v3 Text-to-Speech
    │   │   ├── voice_separator.py   # Voice isolation helpers
    │   │   └── yt_downloader.py     # yt-dlp YouTube downloader
    │   │
    │   ├── tasks/                   # Celery async tasks
    │   │   ├── celery_app.py        # Celery app instance
    │   │   ├── stage1_tasks.py      # Audio prep pipeline (tasks 1–5)
    │   │   ├── stage2_tasks.py      # Per-chunk STT/TTS (parallel chord)
    │   │   ├── stage3_tasks.py      # Audio assembly (pydub mixing)
    │   │   └── stage4_tasks.py      # Video mux + S3 upload
    │   │
    │   ├── models/                  # ML model files
    │   │   ├── feature_extractor.py
    │   │   ├── xgboost.json         # Gender detection model weights
    │   │   └── config.yaml
    │   │
    │   └── utils/
    │       ├── ffmpeg_utils.py
    │       ├── file_manager.py
    │       ├── logger.py
    │       └── timestamp.py
    │
    ├── worker/
    │   └── start_worker.sh          # Celery worker startup (Linux/Docker)
    │
    ├── Frontend/                    # React + Vite frontend
    │   ├── index.html
    │   ├── package.json
    │   ├── vite.config.js
    │   └── src/
    │       ├── App.jsx              # Router: Landing / Input / Preview
    │       ├── components/
    │       │   ├── LandingPage/     # Hero, features, Navbar
    │       │   ├── InputPage/       # Upload form + YouTube input
    │       │   ├── PreviewPage/     # Dubbed video player + download
    │       │   └── authentication/  # Login / Register modals
    │       └── index.css
    │
    └── data/                        # Runtime data (auto-created)
        ├── uploads/                 # Incoming videos
        └── outputs/                 # Dubbed output videos
```

---

## ⚙️ Pipeline Stages

### Stage 1 — Audio Preparation

| Task | Service | Description |
|------|---------|-------------|
| `task_audio_separator` | `audio_extractor.py` | Separates vocals from background noise |
| `task_diarization` | `diarization.py` | Identifies speaker timestamps via pyannote.audio |
| `task_overlap_split` | `overlap_detector.py` | Detects and isolates overlapping speech |
| `task_segment` | `segment_separation.py` | Splits audio into per-speaker segments |
| `task_chunk` | `chunker.py` | Breaks segments into smaller processable chunks |

### Stage 2 — Parallel Chunk Processing (Celery Chord)

Each chunk is processed through the following chain **in parallel**:

```
task_gender → task_stt → task_align → task_tts
```

| Task | Service | Description |
|------|---------|-------------|
| `task_gender` | `gender_detection.py` | Majority-vote gender per speaker (XGBoost + SpeechBrain) |
| `task_stt` | `stt.py` | Sarvam AI Speech-to-Text transcription |
| `task_align` | `translation_matcher.py` | Translates and aligns text to timeline |
| `task_tts` | `tts.py` | Generates Sarvam Bulbul:v3 voice audio |

### Stage 3 — Audio Assembly

Uses **pydub absolute-position mixing** to overlay all TTS chunks onto a background canvas at precise timestamps. Overlapping segments are resolved by keeping the longer one. Background audio is lowered 3 dB for clarity.

### Stage 4 — Video Mux + S3 Upload

- Merges dubbed audio with original video using **ffmpeg**
- Optionally uploads to **AWS S3** and returns a presigned download URL (valid 1 hour)
- Falls back to local `/download/{filename}` endpoint if S3 is not configured

---

## 🛠️ Tech Stack

### Backend

| Component | Technology |
|-----------|-----------|
| API Framework | FastAPI + Uvicorn |
| Task Queue | Celery |
| Message Broker | Redis |
| Speaker Diarization | pyannote.audio |
| Gender Detection | XGBoost + SpeechBrain + librosa |
| STT / TTS | Sarvam AI (Bulbul:v3) |
| Audio Processing | pydub, librosa, soundfile, scipy |
| Video Processing | ffmpeg (via static-ffmpeg) |
| YouTube Download | yt-dlp |
| Cloud Storage | AWS S3 (boto3) |
| ML Framework | PyTorch, transformers, scikit-learn |
| Source Separation | spleeter |

### Frontend

| Component | Technology |
|-----------|-----------|
| Framework | React 19 + Vite 7 |
| Routing | React Router DOM v7 |
| Styling | TailwindCSS v4 |
| HTTP Client | Axios |
| Notifications | react-toastify |

---

## 🚀 Installation

### Prerequisites

- Python 3.11
- Node.js 18+
- Docker (for Redis)
- ffmpeg (auto-installed via `static-ffmpeg`)

### Backend Setup

```bash
cd auto_dub_system

# Create virtual environment
python -m venv venv311

# Activate (Windows)
venv311\Scripts\activate

# Activate (Linux/macOS)
# source venv311/bin/activate

# Install dependencies
pip install -r requirements.txt
```

### Frontend Setup

```bash
cd auto_dub_system/Frontend
npm install
```

---

## 🔧 Configuration

Create or edit `auto_dub_system/.env`:

```env
# Hugging Face token (required for pyannote.audio diarization)
HF_TOKEN=hf_your_token_here

# Celery / Redis
CELERY_BROKER_URL=redis://localhost:6379/0
CELERY_RESULT_BACKEND=redis://localhost:6379/0

# Sarvam AI (STT + TTS)
SARVAM_API_KEY=sk_your_key_here

# AWS S3 (optional — leave blank to skip S3 upload)
AWS_ACCESS_KEY_ID=your_key
AWS_SECRET_ACCESS_KEY=your_secret
AWS_S3_BUCKET_NAME=your_bucket
AWS_S3_REGION=us-east-1
```

> ⚠️ **Never commit `.env` to version control.** It is already listed in `.gitignore`.

---

## ▶️ Running the System

### Windows — One-Click Launcher

```bat
cd auto_dub_system
run_pipeline.bat
```

This script:
1. Starts a Redis container via Docker
2. Launches the Celery worker (`--pool=threads --concurrency=16`)
3. Launches the FastAPI server on `http://localhost:8000`

### Manual Start (All Terminals)

```bash
# Terminal 1 — Redis
docker run --name auto-dub-redis -p 6379:6379 -d redis:alpine

# Terminal 2 — Celery Worker
cd auto_dub_system
set PYTHONPATH=%cd%
venv311\Scripts\python -m celery -A app.tasks.celery_app worker --loglevel=info --pool=threads --concurrency=16

# Terminal 3 — FastAPI
venv311\Scripts\python -m uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

# Terminal 4 — Frontend Dev Server
cd auto_dub_system/Frontend
npm run dev
```

### Access URLs

| Service | URL |
|---------|-----|
| Frontend | http://localhost:5173 |
| API | http://localhost:8000 |
| API Docs (Swagger) | http://localhost:8000/docs |
| Redis | localhost:6379 |

---

## 📡 API Reference

### Health Check
```
GET /health
Response: { "status": "ok", "message": "Auto Dub System is running" }
```

### Upload Video for Dubbing
```
POST /upload
Content-Type: multipart/form-data

Fields:
  file         (required)  Video file
  source_lang  (optional)  Source language name, default: "auto"
  target_lang  (optional)  Target language name, default: "English"
  gender       (optional)  "Male" or "Female", default: "Male"
  recover_bg   (optional)  "true"/"false"

Response:
  { "filename": "...", "task_id": "...", "status": "processing_started",
    "source_lang": "...", "target_lang": "...", "gender": "...", "recover_bg": false }
```

### Dub via File or Pre-downloaded Path
```
POST /dub_video
Content-Type: multipart/form-data

Fields:
  file                (optional) Video file upload
  youtube_video_path  (optional) Path to a previously downloaded video
  source_lang, target_lang, gender, recover_bg
```

### Check Task Status
```
GET /task/{task_id}
Response:
  { "task_id": "...", "status": "SUCCESS|PENDING|FAILURE", "result": { ... } }
```

### YouTube — Get Video Info
```
POST /youtube/info
Body: { "url": "https://youtube.com/watch?v=..." }
```

### YouTube — Download Video
```
POST /youtube/download
Body: { "url": "https://youtube.com/watch?v=...", "quality": "720p" }
Response: { "status": "success", "file_path": "...", "filename": "...", "size": "12.34 MB" }
```

### Download Output File
```
GET /download/{filename}
```

### Auth Endpoints (Stub)
```
POST /api/users/login
POST /api/users/register
Body: { "username": "...", "email": "...", "password": "..." }
```

---

## 🌐 Supported Languages

| Code | Language | Code | Language |
|------|----------|------|----------|
| `en` | English | `hi` | Hindi |
| `ta` | Tamil | `te` | Telugu |
| `kn` | Kannada | `ml` | Malayalam |
| `bn` | Bengali | `mr` | Marathi |
| `gu` | Gujarati | `pa` | Punjabi |
| `or` | Odia | `es` | Spanish |
| `fr` | French | `de` | German |
| `zh` | Mandarin | `ja` | Japanese |
| `ko` | Korean | `ar` | Arabic |
| `ru` | Russian | `pt` | Portuguese |
| `it` | Italian | `nl` | Dutch |
| `pl` | Polish | `tr` | Turkish |
| `th` | Thai | `vi` | Vietnamese |
| `he` | Hebrew | `sv` | Swedish |
| `da` | Danish | `fi` | Finnish |
| `no` | Norwegian | `cs` | Czech |
| `hu` | Hungarian | `el` | Greek |
| `ro` | Romanian | `bg` | Bulgarian |

---

## 🎭 Frontend Pages

| Route | Component | Description |
|-------|-----------|-------------|
| `/` | `LandingPage` | Hero section, features, CTA |
| `/login` | `Login` | Login modal overlay |
| `/register` | `Register` | Register modal overlay |
| `/input` | `InputPage` | Video upload or YouTube URL form |
| `/preview` | `PreviewPage` | Dubbed video player + download |

---

## 🐳 Docker

```bash
cd auto_dub_system

# Start all services (Redis + API + Celery Worker)
docker-compose up --build

# Run in background
docker-compose up -d --build

# Stop everything
docker-compose down
```

| Service | Port | Description |
|---------|------|-------------|
| `redis` | 6379 | Redis broker + result backend |
| `app` | 8000 | FastAPI server |
| `worker` | — | Celery worker |

---

## 📝 Operational Notes

- On **Windows**, Celery runs with `--pool=threads`. For production parallelism on Linux, use `--pool=prefork -c 12`.
- `data/uploads/` and `data/outputs/` are created automatically at runtime.
- Gender detection uses a **majority-vote** strategy: all chunks per speaker are sampled before TTS to ensure consistent voice assignment.
- Background audio is lowered by **3 dB** to make dubbed speech clearer.
- S3 upload is **optional** — if credentials are not set, the dubbed video is served locally via `/download/{filename}`.
- Bulbul:v3 voice assignment is **deterministic** based on speaker ID, so the same speaker always gets the same voice across runs.
