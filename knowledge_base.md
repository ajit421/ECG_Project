# ECG Project — Knowledge Base
> Single source of truth for all technical decisions, schemas, APIs, and environment setup.
> Updated as the project evolves. Any LLM assisting this project should read this first.

---

## 1. Project Overview

An AI-powered remote ECG monitoring system for hospitals.

- **Hardware**: ESP32 microcontroller + AD8232 ECG sensor (one unit per hospital room).
- **ML Pipeline**: Real-time Python inference using a trained Random Forest classifier (`ecg_rf_model_v1.pkl`).
- **Backend**: Flask + Socket.IO server serving a REST API and WebSocket stream.
- **Frontend**: React (Vite) SPA with role-based dashboards.
- **Database**: MongoDB Atlas (NoSQL, free M0 tier).

---

## 2. Repository Structure

```
ECG_Project/
├── backend/                  ← Flask server, ML inference, REST API
│   ├── src/
│   │   ├── signal_processing.py     # Butterworth filter + Pan-Tompkins QRS
│   │   ├── feature_extraction.py    # 6 HRV features + SQI
│   │   ├── realtime_inference.py    # ECGInferenceEngine (threading model)
│   │   └── ecg_simulator.py         # Demo mode ECG simulator
│   ├── model/
│   │   ├── ecg_rf_model_v1.pkl      # Trained RandomForest (14MB)
│   │   ├── scaler_v1.pkl            # StandardScaler
│   │   └── model_metadata.json
│   ├── logs/                        # Session CSV logs (gitignored)
│   ├── server.py                    # Flask app entry point
│   └── requirements.txt             # Production deps only (no eventlet, no wfdb)
│
├── frontend/
│   ├── react_app/            ← Vite + React SPA
│   └── legacy_dashboard/     ← OLD Streamlit/HTML — to be DELETED
│
├── firmware/
│   └── ECG_Firmware/
│       └── ECG_Firmware.ino  ← ESP32 Arduino code (needs WiFi rewrite)
│
├── ml_training/              ← Offline training only, never deployed
│   ├── model/
│   │   ├── data_preparation.py
│   │   ├── model_training.py
│   │   ├── model_evaluation.py
│   │   └── features_dataset.csv
│   └── requirements.txt      ← Training-only deps (wfdb, matplotlib etc.)
│
├── tests/                    ← pytest unit tests for backend/src/
├── assets/                   ← ML evaluation plots (to be moved to ml_training/assets/)
├── .env.example              ← Template — copy to .env and fill in real values
├── .gitignore
├── conftest.py               ← pytest path config (points to backend/src/)
├── todo_Feature.md
└── knowledge_base.md
```

---

## 3. Technology Stack Decisions

| Layer | Technology | Reason |
|-------|-----------|--------|
| Backend language | Python 3.11 | ML ecosystem, existing code |
| Backend framework | Flask + Flask-SocketIO | Already written, works well |
| SocketIO async mode | `threading` | Required by ECGInferenceEngine threading model |
| **DO NOT USE** | `eventlet` | Monkey-patches threading, deadlocks the inference queue |
| Database | MongoDB Atlas | Free tier, document model fits JSON sensor payloads |
| Big Data pattern | Bucket Pattern + TimeSeries | Prevents IOPS exhaustion on free tier |
| Frontend | React + Vite | Team skill, fast setup |
| Frontend routing | react-router-dom | Standard SPA routing |
| Charting | Recharts | Lightweight, works with React |
| Auth | JWT (PyJWT) | Stateless, works with REST + WS |
| Backend hosting | Render (free Web Service) | Free Python hosting |
| Frontend hosting | Vercel | Free, great for React/Vite |
| IoT protocol | HTTP POST (not MQTT) | Simpler, no broker needed, ESP32 HTTPClient support |

---

## 4. System Data Flow

```
[AD8232 Sensor]
      ↓ analog 0–3.3V
[ESP32 GPIO34]
      ↓ 12-bit ADC (0–4095), 250Hz hardware timer ISR
      ↓ 1-second batch (250 samples) + NTP timestamp
      ↓ HTTP POST  {"device_id", "timestamp", "ecg_values":[], "lead_off"}
      ↓ Header: X-API-Key
[Render: Flask /api/iot-data]
      ↓ Verify API key → lookup device_id → resolve patient_id
      ↓ Push 250 values into ECGInferenceEngine._raw_queue
[ECGInferenceEngine._inference_loop (background thread)]
      ↓ 5-second sliding window (1250 samples, 50% overlap)
      ↓ bandpass_filter → pan_tompkins_qrs → extract_features → scaler → RF model
      ↓ prediction: "Normal" or "ABNORMAL", probability, consecutive count
      ↓ Save summary to MongoDB ECG_Summaries
      ↓ If 3 consecutive ABNORMAL → save Alert to MongoDB Alerts
[Flask push_data_loop (every 200ms)]
      ↓ socketio.emit("update", {...}) to room:patient_<id>
[Vercel: React Doctor Dashboard]
      ↓ Socket.IO client subscribed to room:patient_<id>
      ↓ Recharts live waveform + BPM gauge + alert banner
[ESP32 alert polling (every 5s)]
      ↓ GET /api/alert-status?device_id=<MAC>
      ↓ If {"alert": true} → GPIO25 buzzer HIGH
```

---

## 5. MongoDB Collection Schemas

### 5.1 `users`
```json
{
  "_id": "ObjectId",
  "username": "dr_smith",
  "email": "smith@hospital.com",
  "password_hash": "<bcrypt hash>",
  "role": "doctor",  // "admin" | "doctor" | "nurse" | "patient"
  "created_at": "ISODate"
}
```

### 5.2 `patients`
```json
{
  "_id": "ObjectId",
  "user_id": "ObjectId (→ users)",
  "name": "John Doe",
  "dob": "ISODate",
  "assigned_room": "101",
  "assigned_doctors": ["ObjectId", ...],
  "assigned_nurses": ["ObjectId", ...]
}
```

### 5.3 `devices`
```json
{
  "_id": "ObjectId",
  "device_id": "AA:BB:CC:DD:EE:FF",  // ESP32 MAC address — the unique room identifier
  "room_number": "101",
  "status": "active",  // "active" | "inactive"
  "registered_at": "ISODate"
}
```

### 5.4 `ecg_summaries` — Bucket Pattern (Big Data)
One document per 5-second inference window per patient.
```json
{
  "_id": "ObjectId",
  "patient_id": "ObjectId",
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
  "prediction": "Normal",  // "Normal" | "ABNORMAL"
  "probability": 0.15,
  "consecutive_count": 0
}
```

### 5.5 `ecg_rawbuckets` — Time Series Collection (Big Data)
MongoDB native Time Series collection. One document per 1-second batch from ESP32.
```json
{
  "patient_id": "ObjectId",       // metadata field (for time series)
  "device_id": "AA:BB:CC:DD:EE:FF",
  "timestamp": "ISODate",         // time field (MongoDB uses this for bucketing)
  "ecg_values": [2048, 2051, ...], // 250-element array (1 second of raw ADC)
  "lead_off": 0
}
```
> **Note**: Create this as a MongoDB Time Series collection with `timeField: "timestamp"`, `metaField: "patient_id"`, `granularity: "seconds"`.

### 5.6 `alerts`
```json
{
  "_id": "ObjectId",
  "patient_id": "ObjectId",
  "device_id": "AA:BB:CC:DD:EE:FF",
  "severity": "HIGH",  // "HIGH" (≥3 consecutive ABNORMAL)
  "timestamp": "ISODate",
  "consecutive_count": 4,
  "probability": 0.91,
  "acknowledged": false,
  "acknowledged_by": null
}
```
> **Debounce rule**: Only create a new Alert if no unacknowledged Alert exists for the same `patient_id` in the last 5 minutes.

---

## 6. REST API Endpoints

| Method | Path | Auth | Owner | Description |
|--------|------|------|-------|-------------|
| POST | `/api/auth/login` | None | Satyarth | Returns JWT |
| POST | `/api/iot-data` | X-API-Key header | Satyarth | ESP32 data ingestion |
| GET | `/api/alert-status` | X-API-Key header | Satyarth | ESP32 polls for alert |
| GET | `/api/status` | None | Satyarth | Server health check |
| POST | `/api/start` | JWT (admin) | Satyarth | Start inference engine |
| POST | `/api/stop` | JWT (admin) | Satyarth | Stop inference engine |
| POST | `/api/calibrate` | JWT (admin) | Satyarth | Calibrate baseline |
| POST | `/api/admin/assign-device` | JWT (admin) | Rupam | Map MAC → room |
| POST | `/api/admin/assign-patient` | JWT (admin) | Rupam | Map patient → room |
| POST | `/api/admin/assign-doctor` | JWT (admin) | Rupam | Link doctor to patient |
| POST | `/api/admin/users` | JWT (admin) | Rupam | Create user |
| GET | `/api/doctor/patients` | JWT (doctor/nurse) | Deepika | Get assigned patients |
| GET | `/api/patients/me` | JWT (patient) | Deepika | Get own records |
| GET | `/api/patients/<id>/ecg-history` | JWT (doctor) | Deepika | Paginated ECG history |

---

## 7. Environment Variables

All team members must copy `.env.example` to `.env` and fill in real values.
**NEVER commit `.env` to git.**

```
MONGO_URI             # From MongoDB Atlas → Connect → Drivers
JWT_SECRET            # Any long random string (use: python -c "import secrets; print(secrets.token_hex(32))")
IOT_API_KEY           # Any random string — hardcoded in ESP32 firmware
FLASK_SECRET_KEY      # Any long random string
VITE_API_BASE_URL     # http://localhost:5000 for dev, https://<render-url> for production
```

---

## 8. Key ML Pipeline Facts

- **Model**: Random Forest Classifier, trained on MIT-BIH Arrhythmia Database via `wfdb`
- **Model file**: `backend/model/ecg_rf_model_v1.pkl` (14MB — large, consider Git LFS)
- **Scaler**: `backend/model/scaler_v1.pkl` — StandardScaler, must be applied before prediction
- **Input**: 6 HRV features: `heart_rate`, `rr_mean`, `rr_std`, `sdnn`, `rmssd`, `beat_variance`
- **Window**: 1250 samples = 5 seconds at 250Hz, 50% overlap (new prediction every 2.5 seconds)
- **Threshold**: `predict_proba > 0.70` → ABNORMAL (not just argmax, reduces false positives)
- **Alert**: 3 consecutive ABNORMAL windows → fire alert
- **SQI gate**: If Signal Quality Index < 0.5 → skip inference, mark as "Poor Signal"
- **Filtering**: 4th-order Butterworth bandpass 0.5–40 Hz applied before feature extraction

---

## 9. ESP32 Firmware Facts

- **Chip**: ESP32 (dual-core)
- **Sensor**: AD8232 single-lead ECG module
- **ADC pin**: GPIO34 (input-only, 12-bit, 0–4095)
- **Lead-off pins**: GPIO32 (LO+), GPIO33 (LO-)
- **Buzzer pin**: GPIO25
- **Sampling**: 250Hz via hardware timer ISR (4ms interval, IRAM_ATTR)
- **Current mode**: USB Serial output only → **MUST be rewritten for WiFi HTTP**
- **Target mode**: WiFi HTTP POST 250-sample batches every 1 second + alert polling every 5 seconds

---

## 10. Known Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| Render cold start (50s) while ESP32 is streaming | Cron job at cron-job.org pings /api/status every 10 min |
| MongoDB Atlas IOPS limit on free tier | Bucket Pattern + Time Series collection — not one doc per sample |
| 14MB model deserialisation on cold start | Pre-load model at server startup, not per-request |
| ESP32 HTTP timeout during Render wake-up | Exponential backoff retry (3 attempts) in firmware |
| Credential leaks to GitHub | .gitignore + .env.example + team rule: no secrets in code |
| Single-instance in-memory queue limitation | Acceptable for free tier + college demo; documented as known limitation |
| Alert fatigue | 5-minute debounce window on Alert creation |
