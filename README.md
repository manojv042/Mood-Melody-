# 🎵 Mood Melody

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.10%2B-green.svg)](#)
[![Android](https://img.shields.io/badge/android-Kotlin-brightgreen.svg)](#)
[![Model: MusicGen](https://img.shields.io/badge/model-MusicGen-orange.svg)](#)

Mood Melody is a research / prototype application that detects a user's facial emotion on-device and generates a short piece of music that matches that emotion. It combines an Android client (Kotlin) with a Python backend that composes prompts and runs MusicGen to synthesize audio.

This README is aimed at contributors and users who want to run the system locally, extend it, or deploy it.

---

Table of Contents
- Project overview
- Features
- Architecture (high-level)
- Repository layout
- Quickstart (backend & Android)
- API reference
- Model & TFLite conversion (overview)
- Deployment notes
- Development & testing
- Contributing
- License & contact

---

## Project overview

Mood Melody maps facial emotion -> short generated music. The Android app captures a face image and runs a TensorFlow Lite emotion classifier on-device, then sends the emotion + confidence to a Python backend which builds a prompt (optionally enhanced by an LLM) and generates audio using MusicGen.

Languages:
- Kotlin — ~72.1% (Android client)
- Python — ~27.9% (backend, glue, and ML utilities)

## Features
- On-device emotion detection via a TFLite model
- Flask backend with endpoints to request generation and serve audio
- Prompt-building logic with optional LLM enrichment (GEMINI_API_KEY)
- Music synthesis via MusicGen (produces WAV files)
- Example scripts for experimentation and prompt tuning

## Architecture (runtime flow)
1. Android app captures a face image and runs TFLite to predict an emotion + confidence.
2. App POSTs JSON to backend `/generate` with `user_id`, `mood`, and `confidence`.
3. Backend composes a MusicGen prompt (rules + optional LLM) and runs MusicGen to synthesize audio.
4. Backend saves a WAV file (static/music/) and returns a URL to the app.

## Repository layout (top-level)
```
Mood-Melody/
├── android_app/           # Android project: Kotlin, CameraX, TFLite integration
│   ├── app/               # Android source (manifest, kotlin, res, etc.)
│   ├── build.gradle.kts
│   └── gradlew*

├── backend/               # Flask server + MusicGen/LLM glue
│   ├── music_server.py    # Main Flask application (API: /generate, /music/<file>, /health)
│   ├── musicgenllm.py     # Interactive generator & prompt builder
│   ├── requirements.txt
│   └── Deployment .md     # Deployment notes

├── emotion-detection/     # Emotion detection training docs and tooling
│   └── README.md
├── README.md
├── LICENSE
└── .gitignore
```

### Files inside emotion-detection/
The emotion-detection directory contains the training code and dependency list used to prepare the on-device TFLite model. The key files are:

- emotion-detection/README.md — Module overview, training strategy, preprocessing and model architecture notes.
- emotion-detection/requirements.txt — Pinned Python packages required for training and evaluation (matplotlib, numpy, Pillow, scikit-learn, seaborn, torch, torchvision, etc.).
- emotion-detection/train_model.py — The main training script. What it does:
  - Loads FER+ labels from a CSV and maps image filenames to train/validation/test splits.
  - Implements a PyTorch MobileNetV3Large-based model with a custom classification head.
  - Implements data loading, preprocessing (grayscale→RGB, resize, normalization), light augmentations, and class-weighted loss.
  - Trains in two phases (freeze backbone → train head, then unfreeze and fine-tune), with early stopping and learning rate scheduling.
  - Saves best model checkpoint, and produces training curves (training_curves.png) and confusion matrix output.

If there are other files you expect to see in this folder, tell me their names and I will include them in the documentation and link to usage examples.

---

## Quickstart — backend (local development)
Notes: MusicGen requires large models and benefits greatly from a GPU. For quick iteration, test with small models or mocked responses.

1. Clone and enter repo:
```bash
git clone https://github.com/manojv042/Mood-Melody-.git
cd Mood-Melody-
```

2. Create & activate a Python virtual environment:
```bash
python -m venv .venv
# macOS / Linux
source .venv/bin/activate
# Windows (PowerShell)
.\.venv\Scripts\Activate.ps1
```

3. Install backend dependencies:
```bash
pip install --upgrade pip
pip install -r backend/requirements.txt
```

4. (Optional) Set GEMINI API key for LLM prompt enhancement:
```bash
export GEMINI_API_KEY="your_gemini_api_key_here"
```

5. Run the backend server:
```bash
python backend/music_server.py
```
Default port: 5000 (override with `PORT` env var). The first model load may take several minutes and requires significant RAM/GPU.

## Quickstart — Android app (local)
1. Open `android_app` in Android Studio (import the Gradle project).
2. Update the backend base URL in the app network/Retrofit config. For the emulator, use `http://10.0.2.2:5000`.
3. Build & install via Gradle wrapper:
```bash
cd android_app
./gradlew installDebug
```
4. Run the app. Typical flow:
- Capture face via CameraX
- Run TFLite emotion model locally
- POST emotion to backend `/generate`
- Download & play returned WAV file

---

## API reference
- POST /generate
  - Request JSON:
    {
      "user_id": "default",
      "mood": "happiness",
      "confidence": 0.82
    }
  - Response (200):
    {
      "prompt": "<string used as MusicGen prompt>",
      "music_url": "http://<host>/music/music_gen_<timestamp>.wav"
    }
  - Errors: 400 (bad request), 500 (model/generation errors)

- GET /music/<filename> — serves generated WAV files from `static/music/`
- GET /health — returns `{ "status": "ok" }`

Example curl:
```bash
curl -X POST http://localhost:5000/generate \
  -H "Content-Type: application/json" \
  -d '{"user_id":"default","mood":"happiness","confidence":0.92}' | jq
```

---

## Model training & TFLite conversion (overview)
See `emotion-detection/README.md` for details. High-level steps:
1. Train on FER+ (MobileNetV3Large backbone + custom head).
2. Export to ONNX (or TorchScript) and convert to TensorFlow SavedModel if needed.
3. Convert SavedModel → TFLite with appropriate optimizations/quantization for mobile.

If you want a conversion script for your specific model, I can provide one.

---

## Deployment notes
- Production should run the heavy MusicGen workload on GPU-enabled hosts.
- Consider moving generated audio to object storage (S3 / Cloud Storage) and serve via CDN.
- Use an API gateway with authentication & rate limiting.
- Add logging, monitoring, and automated health checks.

---

## Development & testing
- Backend dependencies: `backend/requirements.txt`.
- Kotlin: use Android Studio + Gradle wrapper in `android_app/`.

Kotlin tests (examples):
```bash
# From android_app
./gradlew test
./gradlew connectedAndroidTest
```

Python tests (if present):
```bash
pytest
```

---

## Contributing
1. Fork the repo and create a topic branch: `git checkout -b feature/your-feature`
2. Make changes, add tests where appropriate.
3. Open a Pull Request describing your changes and how to test them.

Consider adding `CONTRIBUTING.md` and `CODE_OF_CONDUCT.md` for community contributions.

---

## License
This project is licensed under the MIT License — see the `LICENSE` file for details.

## Contact
Repository owner: @manojv042
Open issues or PRs on GitHub for questions or support.

---

Thanks — if you'd like, I can:
- Update the README to include screenshots and example WAV clips
- Add a small pre-downloaded TFLite model for quick Android testing
- Create CONTRIBUTING.md and a PR template

