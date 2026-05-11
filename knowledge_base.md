# ECG Project — Knowledge Base (v2: RPi Edge Architecture)
> Single source of truth for all technical decisions, schemas, APIs, and environment setup.
> **Architecture Version**: v2 — Raspberry Pi 4B Edge AI Node (updated from ESP32 cloud approach).
> Any LLM assisting this project should read this file completely before making any changes.

---

## 1. Project Overview

An AI-powered remote ECG monitoring system for hospitals, using Edge AI on a Raspberry Pi.

- **Hardware Layer**: AD8232 ECG sensor → ESP32 (dumb USB ADC bridge, no WiFi) → Raspberry Pi 4B via USB serial
- **Edge AI Layer**: RPi 4B runs the full Python ML inference pipeline locally (offline-capable)
- **Cloud Layer**: Thin Flask REST API on Render — stores 5-second summaries and alerts, serves the React UI
- **Frontend**: React (Vite) SPA on Vercel — role-based dashboards for Admin, Doctor/Nurse, Patient
- **Database**: MongoDB Atlas (free M0) — stores only summaries and alerts, NOT raw waveform

---

## 2. Demo Mode — Presentation Guide

> This section is critical for the B.Tech final presentation.

### Demo Mode Architecture
The `EcgSimulator` class in `backend/src/ecg_simulator.py` loads real MIT-BIH Arrhythmia Database records and streams them through the exact same `ECGInferenceEngine` pipeline as live hardware. This means:
- The ML model processes real clinical ECG data
- The signal processing pipeline is identical to live patient mode
- Predictions are real — not faked or hardcoded

### MIT-BIH Demo Records

| Record | Arrhythmia Type | HR Pattern | Expected Model Response | Best For |
|--------|----------------|-----------|------------------------|---------|
| `100`  | Normal Sinus Rhythm | ~75 BPM, regular | "Normal" consistently | Show healthy baseline |
| `119`  | PVCs (Premature Ventricular Contractions) | Normal with frequent early beats | Oscillates Normal/ABNORMAL | Default demo record |
| `200`  | Ventricular Bigeminy | Every other beat is PVC | ABNORMAL triggers reliably within 15s | **Best for live demo climax** |
| `201`  | AFib + PVCs | Highly irregular, varying HR | Strong ABNORMAL, high consecutive count | Most dramatic alert |

**Recommended Demo Script for Professors:**
1. Start with record `100` (Normal) → show green "Normal" predictions, stable BPM around 75
2. Explain the system: "This is a healthy patient. The model sees regular HRV patterns."
3. Click "Switch to Arrhythmia" → system switches to record `200` (Bigeminy)
4. Within ~15 seconds (3 consecutive windows × 5s each): ABNORMAL triggers
5. Alert fires → Buzzer sounds on ESP32 → Doctor dashboard on second device shows red alert banner
6. Show the ground truth tab: "MIT-BIH cardiologist annotated these beats as 'V' (Ventricular). Our model independently detected them as ABNORMAL."

### MIT-BIH Beat Annotation Labels
When showing ground truth comparison to professors:
- `N` — Normal beat (model should predict Normal)
- `V` — Premature Ventricular Contraction (model should predict ABNORMAL)
- `A` — Atrial Premature Contraction (may trigger ABNORMAL depending on severity)
- `L` — Left Bundle Branch Block (model should predict ABNORMAL)
- `R` — Right Bundle Branch Block (model should predict ABNORMAL)

### Model Performance Facts (state these to professors)
- Trained on MIT-BIH Arrhythmia Database (PhysioNet) — the gold standard clinical ECG dataset
- Training samples: 8,372 | Test samples: 2,093
- Test accuracy: **81.6%** | F1 score: **76.8%** | Cross-validated F1: 77.6% ± 0.9%
- Features: 6 HRV time-domain features (clinically validated, used in cardiology literature)
- Model: Random Forest (100 trees) — interpretable, no black box
- Probability threshold: 0.70 (conservative — reduces false positives in a clinical setting)

### Pre-Download Commands (run on RPi before presentation)
```bash
# Activate virtual environment
source ~/ecg_env/bin/activate

# Pre-download all demo records (requires internet, ~50MB total)
python3 -c "
import wfdb
records = ['100', '119', '200', '201']
for r in records:
    print(f'Downloading {r}...')
    wfdb.dl_database('mitdb', dl_dir='demo_data', records=[r])
    print(f'{r} done.')
"

# Verify download
ls demo_data/
```

### Emergency Fallback (if internet unavailable on demo day)
`ecg_simulator.py` has a built-in synthetic ECG fallback — if MIT-BIH records can't load, it generates a synthetic waveform that alternates between normal (70 BPM) and fast/irregular (115 BPM) to trigger ABNORMAL detection. **Pre-download the records to avoid needing internet.**



---

## 2. Architecture Decision Log

### v1 (Abandoned) — ESP32 Cloud Approach
- ESP32 would handle WiFi + HTTP POST of raw ECG to a cloud Flask server
- Cloud Flask server would run ML inference (14MB Random Forest model on Render)
- **Rejected because**:
  - Render free tier cold starts (50s) would lose patient data during sleep
  - 14MB model deserialization race condition on cold start
  - Sending 250Hz raw ECG over internet is expensive and fragile
  - ESP32 HTTP overhead at high frequency is unsustainable
  - `eventlet` vs `threading` conflict with existing inference architecture

### v2 (Current) — Raspberry Pi 4B Edge AI
- ESP32 is a dumb USB serial ADC — exactly what it already does, zero firmware changes needed
- RPi 4B reads Serial, runs full ML pipeline locally (the existing Python code works as-is)
- Only 5-second summaries and alerts go to the cloud — massively reduced bandwidth and DB load
- Cloud API is thin and stateless — no ML model, no heavy compute — free tier is now sustainable
- **Benefits**: Works offline, better ADC quality path, "Edge AI" is a stronger academic pitch

---

## 3. Repository Structure

```
ECG_Project/                    ← Dev monorepo (stays local / GitHub for reference)
├── backend/
│   ├── src/
│   │   ├── signal_processing.py      # Butterworth filter + Pan-Tompkins QRS (unchanged)
│   │   ├── feature_extraction.py     # 6 HRV features + SQI (unchanged)
│   │   ├── realtime_inference.py     # ECGInferenceEngine — runs on RPi (unchanged)
│   │   └── ecg_simulator.py          # Demo mode simulator (unchanged)
│   ├── model/
│   │   ├── ecg_rf_model_v1.pkl       # 14MB Random Forest — deployed to RPi, NOT Render
│   │   ├── scaler_v1.pkl             # StandardScaler
│   │   └── model_metadata.json
│   ├── logs/                         # Local session logs (gitignored)
│   ├── server.py                     # RPi EDGE server — inference + local WebSocket
│   ├── cloud_api.py                  # NEW: Render CLOUD API — thin REST only
│   ├── database.py                   # NEW: MongoDB connection singleton
│   ├── requirements.txt              # RPi + dev deps
│   └── cloud_requirements.txt        # NEW: Render-only deps (no scipy/joblib/pyserial)
│
├── frontend/
│   └── react_app/               ← Vite + React SPA → deployed to Vercel
│
├── firmware/
│   └── ECG_Firmware/
│       └── ECG_Firmware.ino     ← ESP32 dumb ADC bridge — NO CHANGES NEEDED
│
├── ml_training/                 ← Offline training only, never deployed anywhere
│   ├── model/                   # Training scripts + dataset
│   └── assets/                  # Evaluation plots
│
├── tests/                       # pytest unit tests (run from dev machine)
├── .env.example                 # Credential template
├── .gitignore
├── conftest.py                  # pytest config (points to backend/src/)
├── todo_Feature.md
└── knowledge_base.md
```

---

## 4. Full Technology Stack

| Layer | Technology | Where it runs | Notes |
|-------|-----------|--------------|-------|
| ECG Sensor | AD8232 | Hardware | Analog output 0–3.3V |
| ADC + Timing | ESP32 + GPIO34 | Hardware | 12-bit ADC, 250Hz hardware timer ISR |
| Serial Bridge | USB cable | Physical | ESP32 → RPi `/dev/ttyUSB0` |
| Edge Compute | Raspberry Pi 4B (4GB) | Hospital room | Runs full Python ML pipeline |
| Edge OS | Raspberry Pi OS 64-bit (Bookworm) | RPi SD card | |
| Edge Framework | Flask + Flask-SocketIO | RPi | `async_mode="threading"` — DO NOT use eventlet |
| ML Model | scikit-learn RandomForest | RPi (local) | 14MB, loads once at boot |
| Edge DB upload | pymongo (HTTP to MongoDB Atlas) | RPi | Summaries only |
| Cloud API | Flask (cloud_api.py) | Render (free) | No ML, no serial, stateless REST |
| Database | MongoDB Atlas M0 | Cloud | Free tier, ~512MB storage |
| Frontend | React + Vite | Vercel (free) | |
| Auth | JWT (PyJWT) | Cloud API + React | Stateless, no session storage needed |
| Charts | Recharts | React | |
| **DO NOT USE** | eventlet | Anywhere | Deadlocks threading model |

---

## 5. Data Flow (v2 — RPi Edge Architecture)

```
[AD8232 Sensor]
      │  analog 0–3.3V
      ▼
[ESP32 GPIO34 — 12-bit ADC at 250Hz]
      │  Serial CSV: "<millis>,<ecg_val>,<lead_off>\n" at 115200 baud
      │  USB cable (/dev/ttyUSB0 or /dev/ttyACM0)
      ▼
[Raspberry Pi 4B — ECGInferenceEngine running server.py]
      │  _serial_reader_loop() reads from /dev/ttyUSB0
      │  _raw_queue (queue.Queue) feeds samples to inference thread
      │  5-second sliding window (1250 samples, 50% overlap)
      │  bandpass_filter → pan_tompkins_qrs → extract_features → scaler → RF predict
      │
      ├─── Every 5s: HTTP POST summary ──► [Render: cloud_api.py /api/ingest/summary]
      │                                          │
      ├─── On ALERT: HTTP POST ──────────► [Render: cloud_api.py /api/ingest/alert]
      │                                          │
      └─── BUZZ_ON: Serial write ────────► [ESP32 GPIO25 Buzzer]  ◄─ immediate, no cloud needed
                                                 ▼
                                      [MongoDB Atlas]
                                      ecg_summaries + alerts
                                             │
                              ┌──────────────┤
                              │              │
                    [Vercel: React]    [Render: cloud_api.py]
                    Doctor polls       serves /api/doctor/patients
                    /api/alerts        /api/patients/<id>/ecg-history
                    every 30s          /api/alerts
```

---

## 6. MongoDB Collection Schemas

### 6.1 `users`
```json
{
  "_id": "ObjectId",
  "username": "dr_smith",
  "email": "smith@hospital.com",
  "password_hash": "<bcrypt hash>",
  "role": "doctor",
  "created_at": "ISODate"
}
```
Roles: `"admin"` | `"doctor"` | `"nurse"` | `"patient"`

### 6.2 `patients`
```json
{
  "_id": "ObjectId",
  "user_id": "ObjectId (→ users._id)",
  "name": "John Doe",
  "dob": "ISODate",
  "assigned_room": "101",
  "assigned_doctors": ["ObjectId", "..."],
  "assigned_nurses": ["ObjectId", "..."]
}
```

### 6.3 `devices`
```json
{
  "_id": "ObjectId",
  "device_id": "rpi-room-101",
  "room_number": "101",
  "status": "active",
  "registered_at": "ISODate"
}
```
`device_id` is a human-readable unique string set by the admin (e.g., `rpi-room-101`). It is configured in the RPi's `.env` file as `EDGE_DEVICE_ID`.

### 6.4 `ecg_summaries` — Bucket Pattern
One document per 5-second inference window. This is the Big Data pattern — instead of one document per raw sample (would be 250 docs/sec), we store aggregated features.
```json
{
  "_id": "ObjectId",
  "patient_id": "ObjectId",
  "device_id": "rpi-room-101",
  "start_time": "ISODate",
  "end_time": "ISODate",
  "heart_rate": 72.4,
  "rr_mean": 828.5,
  "rr_std": 12.3,
  "sdnn": 12.3,
  "rmssd": 9.8,
  "beat_variance": 151.3,
  "r_peak_count": 6,
  "sqi": 0.9,
  "prediction": "Normal",
  "probability": 0.15,
  "consecutive_count": 0
}
```

### 6.5 `alerts`
```json
{
  "_id": "ObjectId",
  "patient_id": "ObjectId",
  "device_id": "rpi-room-101",
  "severity": "HIGH",
  "timestamp": "ISODate",
  "consecutive_count": 4,
  "probability": 0.91,
  "acknowledged": false,
  "acknowledged_by": null
}
```
**Debounce rule**: Only create a new Alert if no unacknowledged Alert exists for the same `patient_id` within the last 5 minutes.

> **No raw ECG collection**: Raw 250Hz waveform data stays on RPi local storage (session CSV logs in `backend/logs/`). It is never uploaded to the cloud. This keeps MongoDB Atlas well within free tier limits.

---

## 7. REST API Endpoints

### Cloud API (Render — `cloud_api.py`)

| Method | Path | Auth | Called by | Description |
|--------|------|------|-----------|-------------|
| POST | `/api/auth/login` | None | React | Returns JWT |
| POST | `/api/ingest/summary` | X-Edge-Key | RPi | Save 5-sec summary |
| POST | `/api/ingest/alert` | X-Edge-Key | RPi | Save alert |
| GET | `/api/doctor/patients` | JWT (doctor/nurse) | React | Assigned patients list |
| GET | `/api/patients/<id>/ecg-history` | JWT (doctor) | React | Paginated summaries |
| GET | `/api/alerts` | JWT (doctor/nurse) | React | Unacknowledged alerts |
| POST | `/api/alerts/<id>/acknowledge` | JWT (doctor/nurse) | React | Mark alert seen |
| GET | `/api/patients/me` | JWT (patient) | React | Own records |
| POST | `/api/admin/users` | JWT (admin) | React | Create user |
| POST | `/api/admin/assign-device` | JWT (admin) | React | Map device_id → room |
| POST | `/api/admin/assign-patient` | JWT (admin) | React | Map patient → room |
| POST | `/api/admin/assign-doctor` | JWT (admin) | React | Link doctor to patient |
| GET | `/api/status` | None | cron-job.org | Health check (keep Render awake) |

### RPi Edge Server (`server.py` — local network only)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/start` | Start ECGInferenceEngine (serial or demo mode) |
| POST | `/api/stop` | Stop engine |
| POST | `/api/calibrate` | Start 30-second baseline calibration |
| GET | `/api/status` | Engine status + current prediction |
| WS | `socket.io` | Pushes ECG waveform + prediction every 200ms to local browser |

---

## 8. Environment Variables

### RPi `.env` (on the Raspberry Pi at `~/ecg_edge/.env`)
```
MONGO_URI=mongodb+srv://ecg_admin:<password>@cluster0.xxxxx.mongodb.net/ecg_db
FLASK_SECRET_KEY=<long random string>
JWT_SECRET=<long random string>
EDGE_DEVICE_ID=rpi-room-101
SERIAL_PORT=/dev/ttyUSB0
CLOUD_API_URL=https://<your-render-app>.onrender.com
EDGE_KEY=<shared secret between RPi and Render>
```

### Render `.env` (set in Render dashboard)
```
MONGO_URI=<same as RPi>
JWT_SECRET=<same as RPi>
FLASK_SECRET_KEY=<can be different>
EDGE_KEY=<same shared secret as RPi>
```

### Vercel `.env` (set in Vercel dashboard)
```
VITE_API_BASE_URL=https://<your-render-app>.onrender.com
```

**Generate secure secrets**: `python3 -c "import secrets; print(secrets.token_hex(32))"`

**NEVER commit any `.env` file to git.** Use `.env.example` as the template.

---

## 9. ML Pipeline Facts

- **Model**: Random Forest Classifier (scikit-learn), trained on MIT-BIH Arrhythmia Database
- **Model file**: `backend/model/ecg_rf_model_v1.pkl` (14MB)
- **Scaler**: `backend/model/scaler_v1.pkl` — StandardScaler, must be applied before prediction
- **Input**: 6 HRV features: `heart_rate`, `rr_mean`, `rr_std`, `sdnn`, `rmssd`, `beat_variance`
- **Window**: 1250 samples = 5 seconds at 250Hz, 50% overlap (new prediction every 2.5 seconds)
- **Threshold**: `predict_proba > 0.70` → ABNORMAL (reduces false positives vs simple argmax)
- **Alert**: 3 consecutive ABNORMAL windows → fire alert
- **SQI gate**: Signal Quality Index < 0.5 → skip inference, mark "Poor Signal"
- **Filtering**: 4th-order Butterworth bandpass 0.5–40 Hz before feature extraction
- **RPi 4B performance**: Full 5-second pipeline completes in ~50–150ms. No performance issue.

---

## 10. ESP32 Firmware Facts (Unchanged — Dumb USB ADC Bridge)

- **Role in v2**: Purely an ADC + serial transmitter. No WiFi, no HTTP, no intelligence.
- **ADC pin**: GPIO34, 12-bit (0–4095), AD8232 analog output
- **Lead-off pins**: GPIO32 (LO+), GPIO33 (LO-)
- **Buzzer pin**: GPIO25 — receives `BUZZ_ON`/`BUZZ_OFF` commands from RPi over Serial
- **Sampling**: 250Hz via hardware timer ISR (4ms interval, `IRAM_ATTR`)
- **Serial output**: `<millis>,<ecg_value>,<lead_off>\n` at 115200 baud
- **Connection to RPi**: USB cable → `/dev/ttyUSB0` or `/dev/ttyACM0`
- **Firmware changes needed**: **None** — current `ECG_Firmware.ino` is already correct for this role

---

## 11. RPi 4B Setup Facts

- **OS**: Raspberry Pi OS 64-bit (Bookworm) — required for ARM64 Python packages
- **Python**: 3.11+ available via apt
- **Key packages**: numpy, scipy, scikit-learn — all have prebuilt ARM64 wheels, install via pip
- **Serial access**: requires `sudo usermod -aG dialout $USER` for non-root serial port access
- **Auto-start**: systemd service `ecg_edge.service` — starts on boot, restarts on crash
- **Local access**: `http://<hostname>.local:5000` from any device on same WiFi
- **Hostname**: set during OS flash in Raspberry Pi Imager (e.g., `ecg-node-101`)

---

## 12. Known Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| Render cold start (50s delay) | cron-job.org pings `/api/status` every 10 min. RPi retries upload 3 times with 5s delay. |
| RPi loses internet mid-session | Inference continues locally. Summaries queued to a local file, flushed when reconnected. |
| RPi crashes | systemd `Restart=always` with 5s cooldown brings it back automatically. |
| MongoDB Atlas IOPS limit | No raw data in cloud. Only one doc per 5 seconds per patient. Free tier is very comfortable. |
| Serial port not found on RPi | Check with `ls /dev/tty*` after plugging ESP32. Update `SERIAL_PORT` in `.env`. |
| Credential leak to GitHub | `.gitignore` covers `.env`. `.env.example` is the only committed template. |
| Alert fatigue | 5-minute debounce window on Alert document creation per patient. |
| Single RPi instance limitation | Acceptable for college demo. Each room needs its own RPi + ESP32 unit. |
