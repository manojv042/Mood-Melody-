# 🎵 Mood Melody

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE) <!-- replace with actual license badge if different -->
[![Python](https://img.shields.io/badge/python-3.10%2B-green.svg)](#)
[![Android](https://img.shields.io/badge/android-Kotlin-brightgreen.svg)](#)
[![Model: MusicGen](https://img.shields.io/badge/model-MusicGen-orange.svg)](#)

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
- Development & code pointers
- Troubleshooting
- Contributing & governance
- License & acknowledgements

---

What this project solves & who it's for
- Problem: Reducing friction in music discovery by generating short, mood-matched tracks automatically from a user's facial expression.
- Target users: researchers, hobbyists, and developers exploring multimodal AI (vision → LLM → generative audio).
- Primary value: an end-to-end demo that connects on-device emotion detection to generative audio, plus reusable backend components for prompt engineering and inference.

Features
- On-device emotion detection via a TFLite model (MobileNetV3-based classifier).
- Flask backend that:
  - Accepts emotion + confidence from the Android app.
  - Builds a music prompt (keyword rules + optional LLM enhancement via Gemini API).
  - Runs MusicGen to synthesize WAV audio and serves it to the client.
- An interactive script (backend/musicgenllm.py) for experimenting with prompts and profiles.
- Background cleanup to remove old generated audio.
- Example user profiles and simple personalization built into the backend.

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
│   ├── app/
│   │   └── src/           # Android app source (manifest, java/kotlin, res)
│   ├── build.gradle.kts   # Gradle config
│   └── gradlew*           # Gradle wrapper
│
├── backend/               # Flask server + MusicGen/LLM glue
│   ├── music_server.py    # Main Flask application (API: /generate, /music/<file>, /health)
│   ├── musicgenllm.py     # Interactive generator script (prompt building + MusicGen)
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
- Android Studio: to build/run the Android app.

Quickstart — backend (local dev)
1. Create and activate virtual environment:
   ```bash
   python -m venv venv
   source venv/bin/activate
   ```
2. Minimal suggested packages (example; pin in requirements.txt later):
   ```bash
   pip install torch torchvision --index-url https://download.pytorch.org/whl/cu117  # GPU wheel, choose correct version
   pip install transformers scipy numpy flask flask-cors requests
   ```
   - If you are on CPU-only, install CPU wheels (or conda).
   - For MusicGen: use the Hugging Face transformers package that supports the MusicGen model. Check the model repo docs for exact versions.
3. Set optional LLM API key:
   ```bash
   export GEMINI_API_KEY="your_gemini_api_key_here"
   ```
   If GEMINI_API_KEY is missing the backend falls back to a keyword-based prompt.
4. Run the backend:
   ```bash
   cd backend
   python music_server.py
   ```
   - The server loads the MusicGen model on startup (this can take several minutes and needs significant RAM/GPU memory).
   - Default port: 5000 (override with PORT env var).

Quickstart — Android app (local)
1. Open `android_app` in Android Studio (Import project using Gradle wrapper).
2. Ensure Android SDK, Kotlin and required SDK packages are installed.
3. Update the backend base URL in Retrofit/network config if needed (for emulator use http://10.0.2.2:5000).
4. Build & install:
   ```bash
   cd android_app
   ./gradlew installDebug
   ```
5. Run app on device. The app will:
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

Performance & cost considerations
- MusicGen inference is expensive and memory-heavy. Consider:
  - Running smaller MusicGen variants for prototyping.
  - Batch or queue requests and limit request length.
  - Using quantized or optimized runtime if available.

Security & privacy
- Camera/image data should be handled carefully. If deploying publicly:
  - Add authentication & authorization to the backend.
  - Prefer TLS for network traffic.
  - Avoid storing raw user images unless necessary; keep retention minimal.
  - Never commit secrets (GEMINI_API_KEY) to the repository — use environment variables or secret managers.

Development notes (where to look in code)
- backend/music_server.py
  - main Flask app: /generate endpoint, create_music_prompt, generate_music, cleanup thread
  - USER_PROFILES map and VALID_MOODS are defined in this file
- backend/musicgenllm.py
  - interactive prompt-generator and controller, useful for experimentation on a development machine
- emotion-detection/README.md
  - training strategy, augmentations, and model architecture notes
- android_app/
  - Android project; look in `android_app/app/src/main` for manifest and source code. Update Retrofit base URL to point to your local backend.

Suggestions & next steps (recommended)
- Add a pinned requirements.txt or environment.yml for reproducible installs.
- Add a sample small pre-downloaded TFLite model (or link to one) so Android developers can run the app without training.
- Add CI that runs linting and small unit tests (backend smoke tests).
- Add a CONTRIBUTING.md and CODE_OF_CONDUCT for open-source contributions.

Troubleshooting (common issues)
- Model OOM on load: try smaller model, reduce batch size, or increase GPU memory.
- No GPU available: generation may be unusably slow; use a tiny model for development.
- GEMINI API calls failing: check GEMINI_API_KEY and network connectivity; code includes fallbacks to keyword prompts.
- Android cannot reach backend: emulator uses 10.0.2.2 to reach host machine, or use device with local network routing.

Contributing
- Please open issues for bugs/feature requests.
- For PRs:
  - Fork the repo, create a topic branch, and open a PR describing the change and testing steps.
  - Add tests where appropriate (especially for backend logic).
  - Keep changes focused per PR.
- See also: add a CONTRIBUTING.md for more details.

Acknowledgements & references
- MusicGen model from Facebook / Hugging Face
- FER+ dataset and MobileNetV3
- CameraX and TensorFlow Lite Android tooling

License
- See LICENSE at the repository root.

Contact
- Repository owner: @manojv042 (GitHub). Open an issue or PR for help.
