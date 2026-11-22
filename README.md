# Phase 1: Field Capture & Telemetry Edge Application (`pothole-field-capture`)

![Status](https://img.shields.io/badge/Phase-1%20Edge%20Capture-blue)
![Platform](https://img.shields.io/badge/Platform-Android%20%7C%20iOS%20%7C%20Edge-green)
![Offline](https://img.shields.io/badge/Architecture-Offline--First-orange)

## 1. Executive Summary
**`pothole-field-capture`** is the cross-platform mobile and vehicle-mounted edge application responsible for acquiring high-precision road surface defect data. Built for municipal field agents, road inspectors, and automated survey vehicles, this application detects potholes in real time, validates image clarity, estimates physical defect dimensions ($\text{cm}$), locks onto high-accuracy GNSS/GPS coordinates, and reliably synchronizes encrypted payloads to the central government database (`Phase 2: pothole-secure-core`)—even across zero-connectivity dead zones.

---

## 2. Core Capabilities & Engineering Principles
1. **Real-Time Edge Computer Vision:** Runs a quantized object detection model (`YOLOv8-nano` via TFLite/ONNX) on-device at 15–30 FPS to detect potholes and render live bounding boxes without requiring cloud inference.
2. **Automated Image Quality Gate:** Computes Laplacian variance sharpness and exposure histograms prior to capture, preventing blurry, dark, or motion-smeared images from entering the municipal pipeline.
3. **Sensor-Assisted Dimensional Estimation:** Converts 2D pixel bounding boxes into real-world physical measurements ($\text{length}$, $\text{width}$, $\text{surface area}$, and $\text{estimated depth}$) using device camera intrinsics, IMU pitch/roll angles, and ground-plane homography.
4. **High-Precision GNSS Telemetry & Local Geofence Warning:** Captures sub-meter GPS coordinates (where supported by hardware) alongside heading, altitude, and horizontal dilution of precision ($\text{HDOP}$). Cross-references a local SQLite geofence cache to alert agents if a pothole has already been logged within a $3\text{ m}$ radius.
5. **Store-and-Forward Offline Resilience:** Persists all captures locally in an ACID-compliant SQLite queue (`PENDING` $\rightarrow$ `UPLOADING` $\rightarrow$ `SYNCED`), guaranteeing zero data loss in tunnels, highways, or rural corridors.

---

## 3. System Architecture & Data Flow

```text
+-----------------------------------------------------------------------------------+
|                        PHASE 1: EDGE CAPTURE APPLICATION                          |
|                                                                                   |
|  [Camera Sensor] ---> [Laplacian Blur Check] ---> [YOLOv8n TFLite Inference]      |
|                                                               |                   |
|  [IMU Pitch/Roll] ---> [Ground-Plane Homography] <------------+                   |
|                                |                                                  |
|  [GNSS / GPS Lock] ---------> [Payload Builder & Deterministic UUID]              |
|                                |                                                  |
|                     +----------v-----------+                                      |
|                     | Local Geofence Check | (SQLite R-Tree / Bounding Box)       |
|                     +----------+-----------+                                      |
|                                |                                                  |
|                     +----------v-----------+                                      |
|                     | SQLite Offline Queue | (Persists JPEG + Telemetry JSON)     |
|                     +----------+-----------+                                      |
+--------------------------------|--------------------------------------------------+
                                 | (Background Sync Worker w/ Exponential Backoff)
                 +---------------+---------------+
                 |                               |
                 v                               v
   [Phase 2: POST /media/presign]    [Phase 2: POST /api/v1/ingest]
   (Direct KMS-Encrypted S3/MinIO)   (Idempotent JSON Telemetry Commit)
```

---

## 4. Deep-Dive: Technical Modules

### A. Two-Stage Vision & Quality Pipeline (`src/services/blurDetector.ts`)
* **Stage 1 (Live Detection):** Processes camera preview frames using `pothole_yolov8n.tflite` (INT8 quantized for low thermal footprint and extended battery life).
* **Stage 2 (Quality Gate):** On shutter trigger (or automated vehicle-mode trigger), the frame is converted to grayscale and convolved with a $3 \times 3$ Laplacian kernel:
  $$\nabla^2 I = \frac{\partial^2 I}{\partial x^2} + \frac{\partial^2 I}{\partial y^2}$$
  If the variance $\sigma^2(\nabla^2 I)$ falls below the configured threshold (e.g., $< 100.0$), the capture is flagged as blurry and the user is prompted to stabilize the camera.

### B. Sensor-Assisted Perspective Scaling (`src/services/dimensionEstimator.ts`)
To eliminate the need for physical reference markers on the road:
1. **Camera Intrinsics:** Reads focal length ($f_x, f_y$) and optical center ($c_x, c_y$) from device camera calibration metadata.
2. **IMU Orientation:** Captures device pitch ($\theta$) and roll ($\phi$) from the accelerometer/gyroscope at the exact millisecond of shutter release, combined with known mounting/holding height ($H$).
3. **Ground-Plane Projection:** Projects the 2D bounding box corners $(u, v)$ onto the road plane ($Z = 0$) to compute:
   * `length_cm`: Longitudinal span along the road axis.
   * `width_cm`: Transverse span across the lane.
   * `area_sq_cm`: Estimated elliptical/polygonal surface area ($\approx \frac{\pi}{4} \times \text{length} \times \text{width}$).
   * `depth_cm`: Derived via ARCore/ARKit depth APIs (on LiDAR/ToF-equipped devices) or shadow-gradient heuristic estimation, with a manual slider override in `ReviewDefectScreen.tsx`.

### C. Local SQLite Store-and-Forward Engine (`src/db/`)
Why SQLite is critical on the edge:
* **Zero Data Loss:** Captures are written atomically to disk before any network call is attempted.
* **Two-Step Zero-Trust Sync (`src/services/syncWorker.ts`):**
  1. Requests a short-lived, KMS-encrypted presigned `PUT` URL from Phase 2 (`POST /api/v1/media/presign`) and streams the raw image binary directly to S3/MinIO.
  2. Posts the lightweight telemetry JSON (referencing the uploaded `storage_key` and deterministic `capture_id`) to `POST /api/v1/ingest`.
  3. Marks the local SQLite record `SYNCED` and purges the cached image file after 24 hours to reclaim device storage.

---

## 5. Phase 1 $\rightarrow$ Phase 2 API Data Contract (`src/types/telemetry.ts`)
Every capture produces an immutable, schema-validated JSON payload matching Phase 2's ingestion endpoint:

```json
{
  "capture_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "captured_at": "2026-11-22T07:34:12.482Z",
  "device_metadata": {
    "device_id": "field-unit-042",
    "app_version": "1.0.0",
    "capture_mode": "MANUAL_AGENT"
  },
  "geolocation": {
    "latitude": 11.016844,
    "longitude": 76.955832,
    "altitude_m": 411.2,
    "horizontal_accuracy_m": 1.4,
    "heading_deg": 182.5,
    "speed_mps": 0.0
  },
  "dimensions": {
    "length_cm": 68.5,
    "width_cm": 44.0,
    "area_sq_cm": 2367.2,
    "depth_cm": 6.5,
    "estimation_method": "IMU_HOMOGRAPHY"
  },
  "vision_metadata": {
    "confidence_score": 0.94,
    "laplacian_sharpness": 248.6,
    "bounding_box_norm": [0.31, 0.44, 0.62, 0.71]
  },
  "media": {
    "storage_key": "raw/2026/11/22/9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d.jpg",
    "mime_type": "image/jpeg",
    "file_size_bytes": 412890
  }
}
```

---

## 6. Repository Directory Structure

```text
pothole-field-capture/
├── .github/
│   └── workflows/
│       └── mobile-ci.yml              # TypeScript linting, Jest unit tests, APK/IPA builds
├── assets/
│   └── models/
│       ├── pothole_yolov8n.tflite     # Quantized INT8 edge object-detection model
│       └── labels.txt                 # Defect class labels (pothole, alligator_crack)
├── src/
│   ├── api/
│   │   ├── client.ts                  # Axios HTTP client with OAuth2/JWT refresh logic
│   │   └── ingestionService.ts        # Presigned URL upload & telemetry JSON commit
│   ├── components/
│   │   ├── CameraViewfinder.tsx       # Live camera preview with SVG bounding-box overlay
│   │   ├── DimensionOverlay.tsx       # Interactive corner handles for dimension tuning
│   │   └── SyncStatusBanner.tsx       # Real-time offline queue counter & manual sync button
│   ├── db/
│   │   ├── schema.ts                  # SQLite tables: local_captures, geofence_cache
│   │   └── captureRepository.ts       # Transactional CRUD for offline store-and-forward
│   ├── navigation/
│   │   └── AppNavigator.tsx           # Navigation stack (Capture, Review, Queue, Settings)
│   ├── screens/
│   │   ├── CaptureScreen.tsx          # Viewfinder, GPS accuracy pill, blur warning, shutter
│   │   ├── ReviewDefectScreen.tsx     # Dimension verification, depth slider, severity notes
│   │   └── OfflineQueueScreen.tsx     # Audit view of PENDING, UPLOADING, FAILED, SYNCED items
│   ├── services/
│   │   ├── blurDetector.ts            # Laplacian variance sharpness calculation
│   │   ├── dimensionEstimator.ts      # IMU pitch/roll + camera intrinsics homography math
│   │   ├── geofenceCache.ts           # Local Haversine proximity check (3m duplicate alert)
│   │   ├── locationService.ts         # High-accuracy GNSS/GPS + HDOP filtering
│   │   └── syncWorker.ts              # Background queue processor with exponential backoff
│   ├── types/
│   │   └── telemetry.ts               # Strict TypeScript interfaces for Phase 2 contract
│   └── App.tsx
├── __tests__/
│   ├── dimensionEstimator.test.ts     # Unit tests for homography pixel-to-cm accuracy
│   ├── geofenceCache.test.ts          # Unit tests for 3m spatial duplicate alerts
│   └── syncWorker.test.ts             # Offline retry & idempotency queue tests
├── .env.example
├── package.json
├── tsconfig.json
└── README.md
```

---

## 7. Local Development & Setup

### Prerequisites
* Node.js $\ge 20.x$ & npm/yarn
* React Native CLI / Android Studio (SDK 34+) or Xcode 15+
* Physical Android/iOS device recommended (for hardware GNSS, IMU sensors, and camera testing)

### Installation
```bash
# 1. Clone the Phase 1 repository
git clone https://github.com/your-org/pothole-field-capture.git
cd pothole-field-capture

# 2. Install dependencies
npm install

# 3. Configure environment variables
cp .env.example .env
# Edit .env with your Phase 2 API Gateway URL and OAuth Client ID

# 4. Run unit tests (homography math, geofence cache, sync worker)
npm test

# 5. Launch on connected Android or iOS device
npx react-native run-android
# or
npx react-native run-ios
```

### Environment Variables (`.env.example`)
```env
API_BASE_URL=https://api.pothole-core.gov.local/v1
AUTH_TOKEN_ENDPOINT=https://auth.pothole-core.gov.local/oauth/token
DUPLICATE_WARNING_RADIUS_METERS=3.0
MIN_LAPLACIAN_SHARPNESS=100.0
MAX_GPS_ACCURACY_THRESHOLD_METERS=5.0
SYNC_BATCH_SIZE=10
```

---

## 8. Phase 1 Exit Criteria (Definition of Done)
- [ ] Field agent can detect a pothole on-device with live bounding-box overlay at $\ge 15\text{ FPS}$.
- [ ] Blurry frames ($\sigma^2 < 100$) and poor GPS locks ($\text{accuracy} > 5\text{ m}$) trigger clear UI warnings before submission.
- [ ] Calculated $\text{length}$ and $\text{width}$ measurements achieve within $\pm 10\%$ accuracy against physical ground-truth tape measurements at $1.5\text{ m}$–$2.5\text{ m}$ camera height.
- [ ] Captures recorded in Airplane Mode persist across app restarts in SQLite and automatically sync to Phase 2 once connectivity is restored without duplicate records.