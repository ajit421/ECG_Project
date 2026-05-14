# AI-Based Real-Time ECG Anomaly Detection System (v2 Edge Architecture)

> **Final Year Project** — Edge AI on Raspberry Pi + React Web Dashboard + MongoDB Cloud
> Slide: https://docs.google.com/presentation/d/1WCODxqY1NNHRyLQVIZKMnhqAkkMkOI6z_rHUC1bUbVA/edit?usp=sharing
https://docs.google.com/presentation/d/1TkYRfWm5ueAqeKLgt7O9ZuyEE2wz-jzjRYnoaAk74RY/edit?usp=sharing
---

## 🔬 System Overview (v2 Edge AI)

```
[Patient] 
  → AD8232 ECG Sensor
    → ESP32 (Dumb USB ADC bridge @ 250Hz)
      → Raspberry Pi 4B (Local Edge AI Node)
        → Python Pipeline (Filter → Features → Random Forest)
        → Local Buzzer Alert (immediate)
        → 5-second Summaries HTTP POST
          → Render (Cloud REST API)
            → MongoDB Atlas
              → Vercel (React Dashboard for Doctors)
```

**Why Edge AI?** By running the ML model locally on the Raspberry Pi, the system works offline, prevents cloud latency for alerts, avoids sending massive 250Hz waveform data over the internet, and ensures the Render free tier never times out.

---

## 📁 Repository Structure

This monorepo serves as our team's planning hub and training environment. The production code has been split into three separate deployment repositories:

| Component | GitHub Repo | Deployment | Description |
|-----------|-------------|------------|-------------|
| **Backend** | [satyarth8/ecg-backend](https://github.com/satyarth8/ecg-backend) | Raspberry Pi & Render | Edge inference server + Cloud REST API |
| **Frontend** | [satyarth8/ecg-frontend](https://github.com/satyarth8/ecg-frontend) | Vercel | Vite/React SPA dashboard |
| **Firmware** | [satyarth8/ecg-firmware](https://github.com/satyarth8/ecg-firmware) | ESP32 | Arduino sketch for USB ADC reading |

### Local Monorepo Structure:
```
ECG_Project/
├── backend/            # Edge server & Cloud API (matches ecg-backend repo)
├── frontend/           # React dashboard (matches ecg-frontend repo)
├── firmware/           # ESP32 code (matches ecg-firmware repo)
├── ml_training/        # Training scripts & MIT-BIH dataset (Local only)
├── .env.example        # Environment variable template
├── knowledge_base.md   # Single source of truth for schemas & APIs
└── todo_Feature.md     # Phase-by-phase task tracking
```

---

## ⚡ Hardware Wiring

| AD8232 Pin | ESP32 Pin | Notes |
|---|---|---|
| **3.3V** | **3V3** | Power |
| **GND** | **GND** | Ground |
| **OUTPUT** | **GPIO34** | ADC input-only, 12-bit |
| **LO+** | **GPIO32** | Lead-off detection |
| **LO-** | **GPIO33** | Lead-off detection |

**Buzzer (5V Active Type):**
Requires a transistor (BC547/2N2222) switching circuit driven by **GPIO25** on the ESP32.

---

## 🛠 Setup & Installation

### 1. Edge Node (Raspberry Pi 4B)
1. Flash **Raspberry Pi OS 64-bit**.
2. Clone `ecg-backend` to the Pi.
3. Install dependencies: `pip install -r requirements.txt` (includes `scikit-learn`, `flask`, `pymongo`).
4. Copy `.env.example` to `.env` and fill in `MONGO_URI`, `EDGE_DEVICE_ID`, and `SERIAL_PORT`.
5. Run `python server.py`.

### 2. Cloud API (Render)
1. Connect Render to the `ecg-backend` repo.
2. Set build/start command to `python cloud_api.py`.
3. Set environment variables (`MONGO_URI`, `JWT_SECRET`, `EDGE_KEY`).

### 3. Frontend (Vercel)
1. Connect Vercel to the `ecg-frontend` repo.
2. Set environment variable `VITE_API_BASE_URL` to your Render API URL.
3. Deploy.

### 4. ESP32 Firmware
1. Open `ECG_Firmware.ino` from the `ecg-firmware` repo in Arduino IDE.
2. Flash to ESP32 (baud rate 115200). 
3. Connect ESP32 to Raspberry Pi via USB cable.

---

## 🧠 AI Model Specifications

| Parameter | Value |
|---|---|
| Algorithm | Random Forest (scikit-learn) |
| Trees | 100 |
| Features | 6 HRV features |
| Training data | MIT-BIH Arrhythmia Dataset (PhysioNet) |
| Performance | Accuracy: 81.6%  \|  F1: 76.8% |
| Alert threshold | P(abnormal) > **0.70** |
| Alert trigger | **3 consecutive** abnormal 5-second windows |

---

## 🔬 Signal Processing Pipeline

| Stage | Method | Parameters |
|---|---|---|
| Bandpass filter | Butterworth (4th order) | 0.5–40 Hz |
| QRS detection | Pan-Tompkins | Refractory: 200 ms |
| Feature extraction | HRV time-domain | 5-second windows, 50% overlap |
| Normalisation | StandardScaler | Fitted on MIT-BIH training data |

**Extracted Features:** `heart_rate`, `rr_mean`, `rr_std`, `sdnn`, `rmssd`, `beat_variance`.

---

## 🎬 Presentation Demo Mode

For final year project presentations, the system includes a Demo Mode that streams real MIT-BIH clinical data through the ML pipeline, proving the model's efficacy without needing a patient with a real arrhythmia.

Available Demo Records:
- `100` — Normal sinus rhythm
- `119` — PVCs
- `200` — Ventricular Bigeminy (Best for live demo climax)
- `201` — AFib + PVCs

*(See `knowledge_base.md` for full demo script and instructions).*

---

## ⚠️ Important Notes

- **This system detects abnormal ECG patterns only — it does NOT diagnose diseases.**
- AD8232 single-lead setup is NOT reliable for P-wave or ST-segment analysis.
- The model must always be used with its paired `scaler_v1.pkl` for normalisation.
- Buzzer activates after **3 consecutive abnormal windows** (~15 seconds) to prevent false alarms.
