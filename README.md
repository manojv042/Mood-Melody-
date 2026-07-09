# 🎵 Mood Melody

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.10%2B-green.svg)](#)
[![Android](https://img.shields.io/badge/android-Kotlin-brightgreen.svg)](#)
[![Model: MusicGen](https://img.shields.io/badge/model-MusicGen-orange.svg)](#)
[![Release](https://img.shields.io/github/v/release/manojv042/Mood-Melody-?label=release)](https://github.com/manojv042/Mood-Melody-/releases)

Mood Melody is an end-to-end research / prototype application that detects a user's facial emotion on-device and generates a short piece of music that matches that emotion. It integrates three main parts: an on-device TensorFlow Lite emotion detector (Android), a Flask backend that builds and optionally enhances music prompts, and MusicGen (Hugging Face transformers) for audio synthesis.

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
- Development & dependency management
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
- Example user profiles and simple personalization built into the backend.

Languages
- Kotlin: 72.1% (Android app is the majority of source code)
- Python: 27.9% (backend + ML code)

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
│   ├── app/               # Android source (manifest, java/kotlin, res)
│   ├── build.gradle.kts
│   └── gradlew*
│
├── backend/               # Flask server + MusicGen/LLM glue
│   ├── music_server.py    # Main Flask application (API: /generate, /music/<file>, /health)
│   ├── musicgenllm.py     # Interactive generator script (prompt building + MusicGen)
│   ├── requirements.txt   # Pinned Python dependencies for the backend
│   └── Deployment .md     # Deployment notes for backend
│
├── emotion-detection/     # Emotion detection training docs and tooling
│   └── README.md
├── README.md              # (this file)
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
- Android Studio: to build/run the Android app (Kotlin source is the majority of this repo).

Quickstart — backend (local dev)
This project provides the backend Python entrypoint at `backend/music_server.py` and an interactive prompt/experiment script at `backend/musicgenllm.py`.

1. Create and activate a virtual environment from the repository root:

   ```bash
   python -m venv .venv
   # macOS / Linux
   source .venv/bin/activate
   # Windows (PowerShell)
   .\.venv\Scripts\Activate.ps1
   ```

2. Upgrade pip and install the pinned runtime packages from the repository's requirements file:

   ```bash
   pip install --upgrade pip
   pip install -r backend/requirements.txt
   ```

   Notes:
   - `backend/requirements.txt` contains pinned versions. Review the file if you run into incompatibilities (especially torch + CUDA).
   - Installing PyTorch with a GPU-enabled wheel is recommended. See https://pytorch.org for the correct wheel for your CUDA version; if needed, install PyTorch first and then `pip install -r backend/requirements.txt` minus torch.

3. Optional: set a GEMINI API key to enable LLM-enhanced prompt creation (the code falls back to a keyword prompt if not set):

   ```bash
   export GEMINI_API_KEY="your_gemini_api_key_here"
   ```

4. Run the backend (from repository root):

   ```bash
   python backend/music_server.py
   ```

   - The server loads the MusicGen model on startup (this can take several minutes and needs significant RAM/GPU memory).
   - Default port: 5000 (override with PORT env var).

Quickstart — Android app (local)
The Android project uses Gradle (Kotlin DSL). Use the Gradle wrapper inside `android_app/` to build and install.

1. Open `android_app` in Android Studio (Import project using Gradle wrapper).
2. Update the backend base URL in Retrofit/network config if needed (for emulator use `http://10.0.2.2:5000`).
3. Build & install using the Gradle wrapper (from repo root):

   ```bash
   cd android_app
   ./gradlew installDebug
   ```

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
  - Errors:
    - 400: missing fields
    - 500: model or generation errors
- GET /music/<filename> — serves generated WAV files from static/music/
- GET /health — returns { "status": "ok" }

Example: test backend with curl
```bash
curl -X POST http://localhost:5000/generate \
  -H "Content-Type: application/json" \
  -d '{"user_id":"default","mood":"happiness","confidence":0.92}' | jq
```

Model training & converting to TensorFlow Lite (overview)
- Training: the `emotion-detection` docs describe a MobileNetV3Large-based classifier trained on FER+ with augmentation and two-stage fine-tuning (freeze backbone → train head → unfreeze and fine-tune).
- Export & convert high-level steps (actual commands need to match your model and code):
  1. Save the trained PyTorch model (e.g., checkpoint.pth).
  2. Export to ONNX (a representative input shape like [1, 3, 224, 224]):
     ```python
     # example: export.py (run with your env loaded)
     import torch
     model = ...  # load model
     model.eval()
     dummy = torch.randn(1, 3, 224, 224)
     torch.onnx.export(model, dummy, "model.onnx", opset_version=13)
     ```
  3. Convert ONNX → TensorFlow using onnx-tf (or use torch.jit -> TFLite via TF conversion pipeline). Example:
     ```bash
     pip install onnx onnx-tf tensorflow
     onnx-tf convert -i model.onnx -o tf_model
     ```
  4. Convert TF SavedModel → TFLite:
     ```python
     import tensorflow as tf
     converter = tf.lite.TFLiteConverter.from_saved_model("tf_model")
     converter.optimizations = [tf.lite.Optimize.DEFAULT]
     tflite_model = converter.convert()
     open("model.tflite", "wb").write(tflite_model)
     ```
  - NOTE: Converting classification models optimized for mobile often requires careful ops support checks and may need quantization and operator mapping. If you want, I can provide a conversion script tailored to your exact PyTorch model file.

Deployment notes
- The backend loads large model weights and requires GPU. For production:
  - Use GPU-enabled VM or Kubernetes cluster with GPU nodes.
  - Front the backend with an authenticated API gateway and rate-limiting.
  - Store generated audio in object storage (S3/Cloud Storage) and serve via CDN instead of local disk for long-term availability.
  - Use logging and monitoring (Prometheus / Grafana).
- See backend/Deployment .md inside the repo for more notes.

Development & dependency management
- Runtime Python dependencies are pinned in `backend/requirements.txt`.
- If you plan to contribute developer tooling (linters, formatters, test runners), add `backend/requirements-dev.txt` and document it here. (At the time of writing only `backend/requirements.txt` is present.)
- To install runtime dependencies:

  ```bash
  pip install -r backend/requirements.txt
  ```

Releases & versioning
- Use Semantic Versioning for releases (MAJOR.MINOR.PATCH). Tag releases on GitHub (for example `v0.1.0`).
- Create a GitHub Release after a tag to populate the release badge.

Suggestions & next steps
- Add a sample small pre-downloaded TFLite model (or link to one) so Android developers can run the app without training.
- Add CI that runs linting and a backend smoke test.
- Add CONTRIBUTING.md and CODE_OF_CONDUCT for open-source contributions.

Troubleshooting (common issues)
- Model OOM on load: try smaller model, reduce batch size, or increase GPU memory.
- No GPU available: generation may be unusably slow; use a tiny model for development.
- GEMINI API calls failing: check GEMINI_API_KEY and network connectivity; code includes fallbacks to keyword prompts.
- Android cannot reach backend: emulator uses `10.0.2.2` to reach host machine, or use device with local network routing.

Contributing
- Please open issues for bugs/feature requests.
- For PRs:
  - Fork the repo, create a topic branch, and open a PR describing the change and testing steps.
  - Add tests where appropriate (especially for backend logic).
  - Keep changes focused per PR.

Acknowledgements & references
- MusicGen model from Facebook / Hugging Face
- FER+ dataset and MobileNetV3
- CameraX and TensorFlow Lite Android tooling

License
- See LICENSE at the repository root.

Contact
- Repository owner: @manojv042 (GitHub). Open an issue or PR for help.
