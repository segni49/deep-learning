
# Deep Learning Project Overview

This project is a complete, reproducible system for classifying potato leaf diseases and deploying the model with a confidence-aware user experience. It covers three parts: model training (ml), an inference API (server), and a mobile-first frontend (web). Designed for students and practitioners to learn, reproduce, and extend on free infrastructure.

---

## Live Demo and Website

- **Demo URL:** [https://your-demo-url.example.com]

---

## System Architecture

```text
[Web (Next.js, Vercel)]
    ↕ HTTPS
[Server (FastAPI, Render)]
    ↔
[Artifacts (model.pt, labels.json, model_info.txt)]
```

- **Frontend:** Uploads an image and displays class, confidence bar, top-3 predictions, and an explicit “uncertain” state.
- **Backend:** Enforces preprocessing parity, runs inference, applies a confidence threshold, and returns structured JSON.
- **Artifacts:** Ensure identical preprocessing (image size, normalization) between training and inference for reliable results.

---

## Quick Start

-### Prerequisites

- **Python:** 3.10 for ml and server
- **Node.js:** 18+ for web
- **Package managers:** pip and npm
- **GPU (optional):** For faster training; inference runs on CPU

### Setup and Run Locally

1. **Train or use a model (ml)**

   - **Create venv and install deps:**

     ```bash
     cd ml
     python -m venv .venv && source .venv/bin/activate  # Windows: .venv\Scripts\activate
     pip install -r requirements.txt
     ```

   - **Run training:**

     ```bash
     python -m src.train \
       --data ./data/plantvillage_potato_3class \
       --img-size 160 \
       --epochs 10 \
       --seed 42
     ```

   - **Outputs:** `outputs/model.pt`, `outputs/labels.json`, `outputs/model_info.txt`, `outputs/metrics.json`, `outputs/confusion_matrix.png`

1. **Start the inference API (server)**

   - **Install and run:**

     ```bash
     cd server
     python -m venv .venv && source .venv/bin/activate
     pip install -r requirements.txt
     uvicorn app:app --host 0.0.0.0 --port 8000
     ```

   - **Health check:** <http://localhost:8000/health>

1. **Start the frontend (web)**

   - **Install and run:**

     ```bash
     cd web
     npm install
     echo "NEXT_PUBLIC_API_URL=http://localhost:8000" > .env.local
     npm run dev
     ```

- **Open UI:** [http://localhost:3000]

1. **Test a prediction**

```bash
   curl -X POST \
     -F "file=@./sample_images/late_blight.jpg" \
     <http://localhost:8000/predict>
   ```

   ```json
   {
     "class": "Potato___Late_blight",
     "confidence": 0.93,
     "top3": [
       {"class": "Potato___Late_blight", "confidence": 0.93},
       {"class": "Potato___Early_blight", "confidence": 0.06},
       {"class": "Potato___Healthy", "confidence": 0.01}
     ],
     "guidance": "Late blight detected. Avoid overhead irrigation; monitor spread closely."
   }
   ```

- **Training strategy:** Two phases (head training, then unfreeze `layer4`), Adam optimizer with early stopping.
- **Metrics:** Validation accuracy around ~92% and macro F1 around ~0.92 (typical).

### Reproducible Artifacts

- **model.pt:** Trained weights.
- **labels.json:** Index → class mapping.
- **model_info.txt:** Preprocessing spec (e.g., `arch=resnet18`, `img_size=160`, `normalize=imagenet`).
- **metrics.json:** Accuracy, F1, loss curves.
- **confusion_matrix.png:** Visual error analysis.

---

## Inference API (server)

### Endpoints

- **GET /health**
  - **Purpose:** Service readiness + metadata.
  - **Returns:** `status`, `labels`, `img_size`, `arch`.

- **POST /predict**
  - **Input:** Multipart `file` (JPEG/PNG).
  - **Pipeline:** Convert to RGB → resize 160×160 → ImageNet normalize → forward pass → softmax → top-3.
  - **Output (confident):**

  ```json
    {
      "class": "Potato___Late_blight",
      "confidence": 0.93,
      "top3": [
        {"class": "Potato___Late_blight", "confidence": 0.93},
        {"class": "Potato___Early_blight", "confidence": 0.06},
        {"class": "Potato___Healthy", "confidence": 0.01}
      ],
      "guidance": "Late blight detected. Avoid overhead irrigation; monitor spread closely."
    }

  
  ```

  - **Output (uncertain, <0.50):**

   ```json
    {
      "class": "uncertain",
      "confidence": 0.42,
      "top3": [
        {"class": "Potato___Late_blight", "confidence": 0.42},
        {"class": "Potato___Early_blight", "confidence": 0.36},
        {"class": "Potato___Healthy", "confidence": 0.22}
      ],
      "message": "Low confidence. Try better lighting, closer focus, or a different angle."
    }
    ```

### Operational Notes

- **Preprocessing parity:** The backend mirrors training transforms from `model_info.txt` to prevent drift.
- **Latency:** ~300–500 ms per image on CPU (Render free tier typical).
- **Cold start:** ~10–15 seconds on free tier; ping `/health` on boot to warm up.
- **Security:** Validate media type and size; restrict CORS to your frontend domain.

---

## Frontend (web)

### UX Principles

- **Mobile-first:** Single-column layout, large tap targets, fast initial load.
- **Transparency:** Confidence bar, top-3 predictions, and explicit “uncertain” state.
- **Guidance:** Actionable tips for better images (lighting, focus, angle).
- **Accessibility:** Semantic HTML, ARIA labels, high contrast for outdoor readability.
- **Polish:** Smooth transitions, consistent spacing and typography.

### Configuration

- **Environment variable:** `NEXT_PUBLIC_API_URL` must point to the backend URL.
- **Contract handling:** Always expect `class`, `confidence`, `top3`; handle `class="uncertain"` gracefully.

---

## Evaluation and Results

- **Validation accuracy:** ~92% (fine-tuned ResNet-18).
- **Macro F1:** ~0.92 across Healthy, Early blight, Late blight.
- **Confusion matrix:** Most errors are Early vs Late blight due to visual similarity in lesion stages.
- **Confidence behavior:** Majority >0.85; uncertain path below 0.50 with user guidance.

> Your exact numbers depend on dataset splits and training runs; see `ml/outputs/metrics.json`.

---

## Reproducibility Checklist

- **Fixed seeds:** Python, NumPy, and PyTorch (e.g., 42).
- **Documented transforms:** Stored in code and reflected in `model_info.txt`.
- **Parity at inference:** Backend reads and applies the same image size and normalization as training.
- **Versioned artifacts:** Model weights, labels, and preprocessing spec are saved and tracked.
- **Deterministic splits:** Stratified train/val split saved or scripted.

---

## Deployment Guide

### Backend (Render)

- **Build:** `pip install -r requirements.txt`
- **Start:** `uvicorn app:app --host 0.0.0.0 --port $PORT`
- **Environment:** CPU-only PyTorch build; Python 3.10
- **Warm-up:** Use `/health` call at startup to reduce cold start delays

### Frontend (Vercel)

- **Env:** `NEXT_PUBLIC_API_URL` set to your backend URL
- **Build:** Standard Next.js build pipeline (`npm run build`)
- **CORS:** Restrict backend CORS to your Vercel domain

---

## Troubleshooting

- **Blank or error in predictions**
  - **Check:** `server/artifacts` contains `model.pt`, `labels.json`, `model_info.txt`.
  - **Fix:** Ensure the backend’s transforms exactly match `model_info.txt`.
- **CORS blocked on frontend**
  - **Check:** Backend includes your Vercel domain in allowed origins.
  - **Fix:** Update CORS middleware and redeploy.
- **Cold start delays**
  - **Mitigate:** Ping `/health` during frontend boot or set a scheduled warm-up.
- **Unstable or low confidence**
  - **Check:** Lighting and focus in input images; verify resize and normalization.
  - **Fix:** Re-export model and `model_info.txt`; consider increasing image size to 192×192.

---

## FAQ

- **Can I add more diseases or crops?**
  - **Yes:** Expand `labels.json`, retrain with the new dataset, and redeploy artifacts.
- **How do I trust the confidence scores?**
  - **Baseline:** Softmax with a threshold at 0.50 for “uncertain.”
  - **Upgrade:** Apply temperature scaling or isotonic regression for calibration.
- **Can this run offline?**
  - **Currently no:** The frontend calls the backend API. You can explore local in-browser inference with TensorFlow.js or ONNX, but expect trade-offs.
- **Why ResNet-18?**
  - **Reason:** Strong speed/accuracy trade-off for free tiers; consider MobileNetV3 or EfficientNet-Lite for faster CPU inference.

---

## Acknowledgments

- **Dataset:** PlantVillage (color subset) by Hughes & Salathé.
- **Libraries:** PyTorch, FastAPI, Next.js, Tailwind CSS.
- **Hosting:** Render (backend), Vercel (frontend).
- **Inspiration:** Community tutorials and documentation used during development.

---

## License

- **Code:** MIT License (see LICENSE).
- **Model and data usage:** Educational purposes; do not use predictions as sole basis for agricultural decisions. Seek local expertise for field interventions.
