# ECG Project — Full Feature & Task Checklist
> This is the single source of truth for all work across all phases.
> Each person owns the tasks tagged next to them.

---

## 🔧 Phase 0 — Breakage Fixes (COMPLETED — Satyarth)
- [x] Fix `conftest.py` broken `sys.path` (was pointing at deleted `src/`, now `backend/src/`)
- [x] Fix `tests/test_signal_processing.py` broken import path → `backend/src/`
- [x] Fix `tests/test_feature_extraction.py` broken import path → `backend/src/`
- [x] Remove `eventlet` from `backend/requirements.txt` (conflicts with `async_mode="threading"`)
- [x] Remove training-only deps (`wfdb`, `matplotlib`) from `backend/requirements.txt`
- [x] Create `ml_training/requirements.txt` for training environment only
- [x] Add `pymongo` and `python-dotenv` to `backend/requirements.txt`
- [x] Create `.env.example` with all required env variable templates
- [x] Create `.gitignore` (was completely missing — credentials were at risk)
- [ ] Delete stale `__pycache__/` at project root (`git rm -r --cached __pycache__`)
- [ ] Delete `frontend/legacy_dashboard/` (dead Streamlit code — fully replaced by React)
- [ ] Move `assets/` folder (confusion_matrix.png, roc_curve.png etc.) → `ml_training/assets/`
- [ ] Create `backend/logs/` directory with a `.gitkeep` so the path exists on server start
- [ ] Commit all Phase 0 fixes: `git commit -m "fix: resolve all Phase 0 breakages"`

---

## 🏗️ Phase 1 — Architecture Restructure (COMPLETED)
- [x] Create `backend/`, `frontend/`, `ml_training/` directories
- [x] Move `src/` → `backend/src/`
- [x] Move `server.py` → `backend/server.py`
- [x] Move `requirements.txt` → `backend/requirements.txt`
- [x] Move `model/ecg_rf_model_v1.pkl` + `scaler_v1.pkl` + `model_metadata.json` → `backend/model/`
- [x] Move training scripts → `ml_training/model/`
- [x] Move dashboard to `frontend/legacy_dashboard/` (to be deleted in Phase 0 cleanup)
- [x] Scaffold React app with Vite in `frontend/react_app/`
- [x] Commit restructure: `git commit -m "refactor: Phase 1 architecture restructure"`

---

## 🗄️ Phase 2 — Database Setup (Sohan)
> **Sign up**: [MongoDB Atlas](https://www.mongodb.com/cloud/atlas/register)

- [ ] Create MongoDB Atlas account (free M0 tier)
- [ ] Create cluster, database user (`ecg_admin`), allow `0.0.0.0/0` network access
- [ ] Copy the `MONGO_URI` connection string and share with Satyarth securely (NOT via git)
- [ ] Design and document all collection schemas in `knowledge_base.md`
- [ ] Create `Users` collection (fields: `username`, `password_hash`, `role`: admin/doctor/nurse/patient)
- [ ] Create `Patients` collection (fields: `name`, `dob`, `assigned_room`, `assigned_doctors[]`, `assigned_nurses[]`)
- [ ] Create `Devices` collection (fields: `device_id` (ESP32 MAC), `room_number`, `status`: active/inactive)
- [ ] Create `ECG_Summaries` collection — **Bucket Pattern** (fields: `patient_id`, `start_time`, `readings[]`, `predictions[]`)
- [ ] Create `ECG_RawBuckets` collection — **MongoDB Time Series** (fields: `patient_id`, `timestamp`, `ecg_values[]` array of 1-sec batches)
- [ ] Create `Alerts` collection (fields: `patient_id`, `severity`, `timestamp`, `consecutive_count`, `acknowledged`: bool)
- [ ] Verify connection from a local Python script using `MONGO_URI`
- [ ] Commit: `git commit -m "docs: add MongoDB schemas to knowledge_base"`

---

## ⚙️ Phase 3 — Hardware / Firmware Rewrite (Ajit)
> This is the **longest lead-time task**. Start immediately.

- [ ] Verify ESP32 + AD8232 hardware connection (current pin mapping is correct, keep it)
- [ ] **Rewrite `ECG_Firmware.ino`** — add Wi-Fi + HTTP client capability:
  - [ ] Add `WiFi.h`, `HTTPClient.h`, `ArduinoJson.h` libraries
  - [ ] Store Wi-Fi SSID/password and API URL in firmware constants (will be replaced by OTA config later)
  - [ ] On boot: connect to Wi-Fi, sync time via NTP (`configTime()`)
  - [ ] Keep existing 250Hz hardware timer ISR (it is correct, do not touch)
  - [ ] Accumulate 250 samples (1 second of data) into a local array buffer
  - [ ] Every 1 second: serialize buffer as JSON `{"device_id":"<MAC>","timestamp":<epoch_ms>,"ecg_values":[...],"lead_off":0}`
  - [ ] HTTP POST to `https://<backend-url>/api/iot-data` with header `X-API-Key: <IOT_API_KEY>`
  - [ ] On HTTP 503/timeout: implement exponential backoff retry (3 attempts)
  - [ ] Remove old `Serial.print` data streaming (it is only used for USB mode now, keep for debug)
  - [ ] **Alert polling**: every 5 seconds, HTTP GET `/api/alert-status?device_id=<MAC>` → if response `{"alert": true}`, fire buzzer
- [ ] Test firmware locally against the Flask dev server before deployment
- [ ] Commit: `git commit -m "feat(firmware): full WiFi+HTTP ECG streaming with alert polling"`

---

## 🔌 Phase 4 — Backend API & ML Cloud Adaptation (Satyarth)
> Build over the existing Flask + Socket.IO `server.py`. Do NOT rewrite from scratch.

### 4a — Environment & Config
- [ ] Add `from dotenv import load_dotenv` to `server.py`, load `.env` at startup
- [ ] Replace hardcoded `SECRET_KEY = "ecg_secret_2026"` with `os.getenv("FLASK_SECRET_KEY")`
- [ ] Create `backend/database.py` — MongoClient singleton, collection accessors
- [ ] Verify `backend/model/` path resolves correctly from `backend/server.py` (ROOT_DIR fix)
- [ ] Create `backend/logs/` with `.gitkeep` so CSV session logging doesn't crash

### 4b — IoT Data Ingestion Endpoint
- [ ] Add `POST /api/iot-data` endpoint in `server.py`:
  - [ ] Verify `X-API-Key` header matches `IOT_API_KEY` env var (reject 401 if wrong)
  - [ ] Parse JSON: `device_id`, `timestamp`, `ecg_values[]`, `lead_off`
  - [ ] Look up `device_id` in `Devices` collection → get `room_number`
  - [ ] Look up `room_number` in `Patients` collection → get `patient_id`
  - [ ] If no mapping found → return 404 (device not assigned)
  - [ ] Push each `ecg_val` from `ecg_values[]` into the running `ECGInferenceEngine._raw_queue`
  - [ ] Add `add_iot_data(ecg_val, lead_off)` public method to `ECGInferenceEngine`

### 4c — MongoDB Persistence (replacing CSV log)
- [ ] In `realtime_inference.py` `_run_inference()`: remove `self._log_file.write(...)` CSV logic
- [ ] Instead, push a summary document to `ECG_Summaries` collection after each 5-second window
- [ ] In `server.py`: on ABNORMAL alert, insert document to `Alerts` collection
- [ ] Add debounce: only insert new Alert if no Alert for this `patient_id` in last 5 minutes
- [ ] Add `GET /api/alert-status?device_id=<MAC>` endpoint for ESP32 polling → returns `{"alert": true/false}`

### 4d — Auth & RBAC (JWT)
- [ ] Add `PyJWT` to `backend/requirements.txt`
- [ ] Create `POST /api/auth/login` endpoint → validate user from `Users` collection, return JWT
- [ ] Create JWT middleware decorator `@require_auth` and `@require_role("admin")` etc.
- [ ] Protect all sensitive endpoints with the auth decorator
- [ ] Create `POST /api/admin/assign-device` → map `device_id` to `room_number` (admin only)
- [ ] Create `POST /api/admin/assign-patient` → map `patient_id` to `room_number` (admin only)
- [ ] Create `POST /api/admin/assign-doctor` → link doctor/nurse to patient (admin only)
- [ ] Create `GET /api/patients/me` → patient sees their own records
- [ ] Create `GET /api/doctor/patients` → doctor sees assigned patients + latest status
- [ ] Create `GET /api/patients/<id>/ecg-history` → paginated ECG summaries from MongoDB

### 4e — Real-time WebSocket Routing (multi-patient)
- [ ] Modify `push_data_loop` in `server.py` to emit on room-specific Socket.IO rooms (e.g., `room:patient_<id>`)
- [ ] Doctor frontend subscribes to `room:patient_<id>` when viewing a patient
- [ ] Admin can subscribe to all rooms

### 4f — Deployment Prep
- [ ] Create `backend/Procfile` with: `web: python server.py`
- [ ] Create `backend/render.yaml` (optional but helpful for Render config-as-code)
- [ ] Test entire backend runs cleanly with `python server.py` after all changes
- [ ] Commit: `git commit -m "feat(backend): IoT endpoint, MongoDB persistence, JWT auth, RBAC"`

---

## 🎨 Phase 5 — React Frontend (Deepika + Rupam)

### 5a — Foundation & Routing (Rupam)
- [ ] Install dependencies: `npm install react-router-dom axios socket.io-client recharts`
- [ ] Set up `react-router-dom` with routes: `/login`, `/admin`, `/doctor`, `/patient`
- [ ] Create `AuthContext` (stores JWT token, user role, expiry) using React Context API
- [ ] Create `ProtectedRoute` component — redirects to `/login` if not authenticated
- [ ] Create `src/api/` folder with Axios instance that auto-attaches JWT header

### 5b — Login Page (Deepika)
- [ ] Build `LoginPage.jsx` — email + password form, POST to `/api/auth/login`, store JWT in context
- [ ] Show role-appropriate redirect after login (admin → `/admin`, doctor → `/doctor`, etc.)

### 5c — Admin Dashboard (Rupam)
- [ ] Build `AdminPage.jsx` with three panels:
  - [ ] **Users panel**: list users, add new user (role selector: admin/doctor/nurse/patient)
  - [ ] **Device panel**: form to assign `device_id` (ESP32 MAC) to room number
  - [ ] **Patient panel**: assign patient to room, assign doctor/nurse to patient

### 5d — Doctor Dashboard (Deepika)
- [ ] Build `DoctorPage.jsx`:
  - [ ] Left sidebar: list of assigned patients (fetched from `/api/doctor/patients`)
  - [ ] Main area: click a patient → connect to Socket.IO room `room:patient_<id>`
  - [ ] Real-time ECG waveform using Recharts `LineChart` (stream from Socket.IO `update` event)
  - [ ] BPM gauge and latest ML prediction badge (Normal / ABNORMAL)
  - [ ] Alert banner: appears and plays a browser audio ping when `prediction.label === "ABNORMAL"` and `consecutive_count >= 3`

### 5e — Patient Dashboard (Deepika)
- [ ] Build `PatientPage.jsx`:
  - [ ] Fetch personal info and ECG history from `/api/patients/me`
  - [ ] Display historical BPM chart (last 24 hours) using Recharts
  - [ ] Display last ML prediction and timestamp

### 5f — Deploy Frontend (Rupam)
- [ ] Push `frontend/react_app/` to GitHub
- [ ] Connect repo to [Vercel](https://vercel.com/) → New Project → select `frontend/react_app` as root
- [ ] Set env variable in Vercel: `VITE_API_BASE_URL=https://<your-render-url>`
- [ ] Verify login and dashboard work on deployed Vercel URL
- [ ] Commit: `git commit -m "feat(frontend): full React UI with RBAC and real-time dashboard"`

---

## 🚀 Phase 6 — Deployment & Integration Testing (Rupam)
- [ ] Push `backend/` to GitHub
- [ ] Connect repo to [Render](https://render.com/) → New Web Service → root dir: `backend/`
- [ ] Set all env variables on Render: `MONGO_URI`, `JWT_SECRET`, `IOT_API_KEY`, `FLASK_SECRET_KEY`
- [ ] Set start command on Render: `python server.py`
- [ ] Set up free cron job at [cron-job.org](https://cron-job.org) to ping `https://<render-url>/api/status` every 10 minutes (prevents cold start sleep)
- [ ] End-to-end test checklist:
  - [ ] ESP32 sends data → Render `/api/iot-data` receives it (check Render logs)
  - [ ] Data flows through `_raw_queue` → ML inference → MongoDB `ECG_Summaries`
  - [ ] Doctor dashboard on Vercel shows real-time waveform from live ESP32
  - [ ] ABNORMAL alert triggers browser notification on doctor screen AND buzzer fires on ESP32
  - [ ] Admin can create a new user, assign device, assign patient — all persisted in MongoDB
- [ ] Commit: `git commit -m "deploy: full system integration and Render+Vercel deployment"`

---

## 📚 Phase 7 — Documentation & Knowledge Base (All Members)
- [ ] Update `knowledge_base.md` with final API endpoint list and env variable descriptions
- [ ] Add system architecture diagram to `README.md`
- [ ] Document all MongoDB collection schemas in `knowledge_base.md`
- [ ] Write `backend/README.md` — how to run backend locally
- [ ] Write `frontend/react_app/README.md` — how to run frontend locally
- [ ] Write `firmware/ECG_Firmware/README.md` — how to flash firmware and configure Wi-Fi
- [ ] Commit: `git commit -m "docs: complete project documentation"`
