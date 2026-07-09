# 🎵 Mood Melody

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.10%2B-green.svg)](#)
[![Android](https://img.shields.io/badge/android-Kotlin-brightgreen.svg)](#)
[![Model: MusicGen](https://img.shields.io/badge/model-MusicGen-orange.svg)](#)
[![Release](https://img.shields.io/github/v/release/manojv042/Mood-Melody-?label=release)](https://github.com/manojv042/Mood-Melody-/releases)

Mood Melody is an end-to-end research / prototype application that detects a user's facial emotion on-device and generates a short piece of music that matches that emotion. It integrates three main components: an Android/Kotlin app that runs emotion detection on-device, a Python backend that composes prompts and runs MusicGen, and model training/convert tooling for the emotion detector.

This README is written for contributors and users who want to run the system locally, extend it, or deploy it.

Table of contents
- What this project solves & who it's for
- Features
- Architecture (high-level)
- Repository layout
- Quickstart (backend & Android)
- API reference & examples
- Model training & TFLite conversion
- Deployment notes
- Development & code pointers
- Troubleshooting
- Contributing & governance
- Releases & versioning
- License & acknowledgements

---

What this project solves & who it's for
- Problem: Reducing friction in music discovery by generating short, mood-matched tracks automatically from a user's facial expression.
- Target users: researchers, hobbyists, and developers exploring multimodal AI (vision → LLM → generative audio).

Features
- On-device emotion detection via a TFLite model (MobileNetV3-based classifier).
- Flask backend that:
  - Accepts emotion + confidence from the Android app.
  - Builds a music prompt (keyword rules + optional LLM enhancement via GEMINI_API_KEY).
  - Runs MusicGen to synthesize WAV audio and serves it to the client.
- An interactive script (backend/musicgenllm.py) for experimenting with prompts and profiles.
- Background cleanup to remove old generated audio.

Architecture (runtime flow)
1. Android app captures a face image → runs a TFLite model locally to predict an emotion + confidence.
2. App POSTs JSON to backend /generate endpoint with user_id, mood, confidence.
3. Backend composes music prompt (rules + optional LLM via GEMINI_API_KEY).
4. Backend invokes MusicGen via transformers API, produces a WAV file under static/music/.
5. Backend returns music URL to the Android app; app downloads/plays it.

Repository layout (top-level)
```
Mood-Melody/
├── android_app/           # Android project: Kotlin, CameraX, TensorFlow Lite integration
├── backend/               # Flask server + MusicGen/LLM glue
├── emotion-detection/     # Emotion detection training docs and tooling
├── README.md
├── LICENSE
└── .gitignore
```

How to get this repository and prerequisites
- Clone:
  ```bash
  git clone https://github.com/manojv042/Mood-Melody-.git
  cd Mood-Melody-
  ```
- Recommended host for backend: a GPU-enabled Linux machine (NVIDIA GPU). MusicGen model inference is GPU-intensive; CPU-only is extremely slow.
- Python: 3.10+
- Android Studio: to build/run the Android app.

Quickstart — backend (local dev)
This project provides the backend Python entrypoint at `backend/music_server.py` and an interactive prompt/experiment script at `backend/musicgenllm.py`.

1. Create and activate a virtual environment from the repository root:

   python -m venv .venv

   - On macOS/Linux: `source .venv/bin/activate`
   - On Windows (PowerShell): `.\.venv\Scripts\Activate.ps1`

2. Upgrade pip and install pinned dependencies for runtime:

   pip install --upgrade pip
   pip install -r backend/requirements.txt

   (For development tools run `pip install -r backend/requirements-dev.txt` — see `backend/requirements-dev.txt`.)

3. Optional: set a GEMINI API key to enable LLM-enhanced prompt creation (the code falls back to keyword prompts if not set):

   export GEMINI_API_KEY="your_gemini_api_key_here"

4. Run the backend (from repository root):

   python backend/music_server.py

   - The server loads the MusicGen model on startup (this can take several minutes and needs significant RAM/GPU memory).
   - Default port: 5000 (override with PORT env var).

Notes on entrypoints
- Primary runtime server: `backend/music_server.py` — this is the application the Android client calls (/generate, /music/<file>, /health).
- Interactive prompt generator: `backend/musicgenllm.py` — useful for experimenting with prompt text and MusicGen locally.

Quickstart — Android app (local)
The Android project uses Gradle (Kotlin DSL). Use the Gradle wrapper inside `android_app/` to build and install.

1. Open `android_app` in Android Studio (Import project using Gradle wrapper).
2. Update the backend base URL in Retrofit/network config if needed (for emulator use http://10.0.2.2:5000).
3. Build & install using the Gradle wrapper (from repo root):

   cd android_app
   ./gradlew installDebug

4. Run app on device. The app will:
   - Capture face images via CameraX.
   - Run TFLite model and POST results to backend /generate.
   - Play returned WAV file.

API reference
- POST /generate
  - Request:
    ```json
    {
      "user_id": "default",
      "mood": "happiness",
      "confidence": 0.82
    }
    ```
  - Response (200):
    ```json
    {
      "prompt": "<string used as MusicGen prompt>",
      "music_url": "http://<host>/music/music_gen_<timestamp>.wav"
    }
    ```

Example: test backend with curl
```bash
curl -X POST http://localhost:5000/generate \
  -H "Content-Type: application/json" \
  -d '{"user_id":"default","mood":"happiness","confidence":0.92}' | jq
```

Development & dependency management
- Runtime Python dependencies are pinned in `backend/requirements.txt` and `emotion-detection/requirements.txt`.
- Dev tools are available in `backend/requirements-dev.txt` (formatters, linters, test runner, pre-commit hooks).
- To install both runtime and dev deps (recommended for contributors):

  pip install -r backend/requirements.txt
  pip install -r backend/requirements-dev.txt

Releases & versioning
- Use Semantic Versioning for releases (MAJOR.MINOR.PATCH). Tag releases on GitHub (for example `v0.1.0`).
- Create a GitHub Release after a tag to populate the release badge.
- Suggested release steps (local):
  - Update CHANGELOG.md (if present) and bump version in docs.
  - git tag -a vX.Y.Z -m "Release vX.Y.Z"
  - git push origin vX.Y.Z
  - Create a GitHub Release using the pushed tag.

Formatting badges
- Badges at the top now include license, Python version support, Android/Kotlin and a release badge. If you want CI/build badges (GitHub Actions), add a workflow and I can insert the build status badge.

Suggestions & next steps I applied
- Added exact Python entrypoints and explicit run commands.
- Declared that the Android project uses Gradle and added exact Gradle wrapper commands.
- Added a `backend/requirements-dev.txt` (dev tools) as requested.
- Reformatted badges and added a Releases/Versioning section.

---

If you want, I can now:
- Add a simple GitHub Actions workflow that installs runtime deps and runs a backend smoke test.
- Add a CHANGELOG.md and an initial tag (v0.1.0) commit draft.
- Add CI badge to the README once a workflow is added.

