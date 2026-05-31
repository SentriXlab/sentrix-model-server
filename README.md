# sentrix-model-server

ML inference server for SentriX.

sentrix-model-server receives Common Feature Schema vectors from sentrix-core, runs anomaly detection and fault classification, and returns diagnosis results.

## Architecture

```text
Spring Boot Demo Server
        ↓
Prometheus / LGTM
        ↓
sentrix-core
        ↓
sentrix-model-server
        ↓
Diagnosis Result
(anomaly score, fault type, evidence features)
```

## Responsibilities

- Validate incoming feature vectors against the Common Feature Schema
- Run Isolation Forest anomaly detection
- Run XGBoost fault classification
- Return anomaly score, fault type, confidence, and evidence features

## Model Info

| Item | Value |
|---|---|
| Training Dataset | RCAEval RE2-TT (Train Ticket) |
| Cases | 60 (5 services × 3 runs × 4 fault types) |
| Train Samples | 1,920 |
| Test Samples | 480 |
| Feature Count | 55 (11 raw features × 5 statistics) |
| Detection AUROC | 0.9437 |
| Detection Recall | 0.9333 |
| Classification Accuracy | 0.8479 |
| Classification Macro F1 | 0.7938 |
| Anomaly Threshold | 0.2590 (NORMAL 80th percentile) |
| Schema Version | v1 |

## Fault Types

```text
NORMAL
HIGH_CPU
MEMORY_PRESSURE
LATENCY_SPIKE
ERROR_SPIKE
```

## Feature Schema

### Raw Features (11)

| Feature | Unit | Related Fault |
|---|---|---|
| `request_rate` | req/sec | - |
| `latency_p95` | seconds | LATENCY_SPIKE |
| `latency_p99` | seconds | LATENCY_SPIKE |
| `error_rate` | ratio | ERROR_SPIKE |
| `process_cpu_usage` | ratio | HIGH_CPU |
| `system_cpu_usage` | ratio | HIGH_CPU |
| `jvm_memory_used` | bytes | MEMORY_PRESSURE |
| `jvm_memory_max_ratio` | ratio | MEMORY_PRESSURE |
| `hikaricp_active` | count | - |
| `hikaricp_pending` | count | - |
| `executor_active_threads` | count | - |

### Window Settings

```text
window_size : 300s
step_size   : 30s
statistics  : mean, std, max, min, slope
→ 11 × 5 = 55 model features
```

### Feature Naming Convention

```text
{raw_feature}_{statistic}

Examples:
  latency_p99_mean
  latency_p99_max
  process_cpu_usage_slope
```

## API

### GET /health

```json
{
  "status": "UP",
  "modelLoaded": true,
  "featureSchemaVersion": "v1",
  "models": {
    "detector": "IsolationForest",
    "classifier": "XGBoost"
  }
}
```

### POST /diagnose

**Request**

```json
{
  "timestamp": "2026-05-30T12:00:30",
  "featureSchemaVersion": "v1",
  "features": {
    "request_rate_mean": 120.5,
    "request_rate_std": 10.3,
    "latency_p99_mean": 0.41,
    "latency_p99_max": 0.92,
    "process_cpu_usage_mean": 0.72,
    "..."
  }
}
```

**Response**

```json
{
  "timestamp": "2026-05-30T12:00:30",
  "featureSchemaVersion": "v1",
  "detection": {
    "status": "ANOMALY",
    "anomalyScore": 0.84,
    "threshold": 0.2716
  },
  "classification": {
    "faultType": "LATENCY_SPIKE",
    "confidence": 0.88,
    "top3": [
      { "faultType": "LATENCY_SPIKE", "confidence": 0.88 },
      { "faultType": "ERROR_SPIKE",   "confidence": 0.07 },
      { "faultType": "HIGH_CPU",      "confidence": 0.03 }
    ]
  },
  "evidenceFeatures": [
    { "featureName": "latency_p99_max",  "score": 0.91 },
    { "featureName": "latency_p95_mean", "score": 0.86 }
  ]
}
```

### Error Responses

| errorCode | Description |
|---|---|
| `INVALID_SCHEMA_VERSION` | Unsupported feature schema version |
| `MISSING_FEATURE` | Required feature is missing from request |
| `MODEL_NOT_LOADED` | Model files are missing or failed to load |
| `INFERENCE_FAILED` | Unexpected error during inference |

## Repository Structure

```text
sentrix-model-server/
 ├── app/
 │   ├── main.py
 │   ├── api/
 │   │   └── diagnose.py
 │   ├── core/
 │   │   └── config.py
 │   ├── schemas/
 │   │   ├── request.py
 │   │   └── response.py
 │   ├── services/
 │   │   ├── model_loader.py
 │   │   ├── inference_service.py
 │   │   └── evidence_service.py
 │   └── features/
 │       ├── feature_schema.yml
 │       └── feature_validator.py
 ├── models/
 │   ├── detector.pkl
 │   ├── classifier.pkl
 │   ├── scaler.pkl
 │   └── metadata.json
 ├── Dockerfile
 ├── requirements.txt
 └── README.md
```

## Tech Stack

| Area | Tech |
|---|---|
| Language | Python 3.10+ |
| Framework | FastAPI |
| Server | Uvicorn |
| ML | scikit-learn, XGBoost |
| Data | Pandas, NumPy |
| Model Load | joblib |
| Deploy | Docker |

## Getting Started

### Prerequisites

- Python 3.10+
- Docker
- Model files (`detector.pkl`, `classifier.pkl`, `scaler.pkl`, `metadata.json`) in `models/`

### Run with Docker

```bash
docker build -t sentrix-model-server .
docker run -p 8000:8000 sentrix-model-server
```

### Run locally

```bash
pip install -r requirements.txt
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

## Configuration

```yaml
# config.py defaults
MODEL_DIR: ./models
FEATURE_SCHEMA_VERSION: v1
HOST: 0.0.0.0
PORT: 8000
```

## Related Repositories

- `sentrix-core` — Backend service for metric collection, diagnosis orchestration, and dashboard APIs
- `sentrix-demo-server` — Target Spring Boot application for generating normal and fault scenarios
