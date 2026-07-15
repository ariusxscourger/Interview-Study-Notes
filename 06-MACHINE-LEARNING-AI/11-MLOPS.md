# MLOPS & PRODUCTION ML
## Exam-Style Study Notes

**Roadmap Section:** MLOps (Deployment, Monitoring, CI/CD, Model Serving)
**Prerequisites:** Docker, Kubernetes, CI/CD, Cloud basics, ML fundamentals

---

## 1. MODEL DEPLOYMENT PATTERNS

### WHY? (Problem Statement)
**Models must serve predictions reliably.** Choose deployment pattern based on latency, throughput, and cost requirements.

### HOW? (Deployment Patterns)

#### Batch Inference
```python
# Scheduled jobs (Airflow, Prefect, Dagster)
def batch_predict(model_uri, input_path, output_path):
    model = mlflow.pyfunc.load_model(model_uri)
    df = pd.read_parquet(input_path)
    predictions = model.predict(df)
    pd.DataFrame({'prediction': predictions}).to_parquet(output_path)

# Spark for massive scale
from pyspark.ml import PipelineModel
model = PipelineModel.load(model_uri)
df = spark.read.parquet(input_path)
predictions = model.transform(df)
predictions.write.parquet(output_path)
```

#### Real-time API (REST/gRPC)
```python
# FastAPI (Python)
from fastapi import FastAPI
import mlflow.pyfunc

app = FastAPI()
model = mlflow.pyfunc.load_model("models:/my_model/Production")

@app.post("/predict")
async def predict(request: PredictRequest):
    features = np.array(request.features).reshape(1, -1)
    pred = model.predict(features)
    return {"prediction": float(pred[0])}

# gRPC (High performance)
# Define .proto, generate code, implement servicer
```

#### Async / Streaming
```python
# Kafka + Flink/Spark Streaming
# Request topic → Consumer group → Model → Response topic

# Redis for caching
import redis
r = redis.Redis()
def cached_predict(key, features):
    if r.exists(key):
        return json.loads(r.get(key))
    pred = model.predict(features)
    r.setex(key, 3600, json.dumps(pred))
    return pred
```

#### Edge / Mobile
```python
# ONNX Runtime / TensorRT / CoreML / TFLite
import onnxruntime as ort
session = ort.InferenceSession("model.onnx")
pred = session.run(None, {"input": features.astype(np.float32)})[0]

# Quantization
from onnxruntime.quantization import quantize_dynamic, QuantType
quantize_dynamic("model.onnx", "model_quant.onnx", weight_type=QuantType.QInt8)
```

---

## 2. MODEL SERVING INFRASTRUCTURE

### WHY? (Problem Statement)
**Production serving needs:** Autoscaling, model versioning, A/B testing, observability.

### HOW? (Serving Platforms)

| Platform | Best For | Key Features |
|----------|----------|--------------|
| **Triton Inference Server** | High-throughput, multi-framework | Dynamic batching, model pipeline, GPU/CPU, HTTP/gRPC |
| **TorchServe** | PyTorch native | Model management, metrics, multi-model |
| **TF Serving** | TensorFlow | Versioning, batching, GPU |
| **BentoML** | Python-first, flexible | Custom logic, runners, Yatai deployment |
| **Seldon Core** | Kubernetes-native | Canary, A/B, explainers, drift detection |
| **KServe** | K8s standard | Serverless, autoscaling, storage initializer |
| **Ray Serve** | Ray ecosystem | Scalable, flexible, ML-specific |

#### Triton Example
```yaml
# config.pbtxt
name: "my_model"
backend: "onnxruntime"
max_batch_size: 32
input [
  { name: "input", data_type: TYPE_FP32, dims: [10] }
]
output [
  { name: "output", data_type: TYPE_FP32, dims: [1] }
]
dynamic_batching {
  max_queue_delay_microseconds: 1000
}
instance_group [
  { kind: KIND_GPU, count: 1 }
]
```

#### Kubernetes Deployment
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-model
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ml-model
  template:
    spec:
      containers:
      - name: model
        image: my-registry/ml-model:v1.2.3
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
            nvidia.com/gpu: "1"
          limits:
            memory: "2Gi"
            cpu: "1000m"
            nvidia.com/gpu: "1"
        env:
        - name: MODEL_URI
          value: "models:/my_model/Production"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: ml-model
spec:
  selector:
    app: ml-model
  ports:
  - port: 80
    targetPort: 8000
```

---

## 3. CI/CD FOR ML

### WHY? (Problem Statement)
**Automate: test → train → evaluate → deploy.** Reproducible, auditable, fast.

### HOW? (Pipeline Stages)

#### GitHub Actions / GitLab CI / Jenkins
```yaml
# .github/workflows/ml-pipeline.yml
name: ML Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with: { python-version: '3.10' }
      - run: pip install -r requirements.txt
      - run: pytest tests/ --cov=src
      - run: python -m pytest tests/data_validation.py
      
  train:
    needs: test
    runs-on: [self-hosted, gpu]
    steps:
      - uses: actions/checkout@v3
      - name: Train
        run: |
          python train.py \
            --config configs/prod.yaml \
            --experiment-name ${{ github.repository }} \
            --run-name ${{ github.sha }}
      - uses: actions/upload-artifact@v3
        with:
          name: model
          path: mlruns/
          
  evaluate:
    needs: train
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v3
        with: { name: model }
      - run: |
          python evaluate.py \
            --model-path mlruns/ \
            --threshold-auc 0.85 \
            --threshold-f1 0.80
            
  deploy:
    needs: evaluate
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build & Push
        run: |
          docker build -t ${{ secrets.REGISTRY }}/ml-model:${{ github.sha }} .
          docker push ${{ secrets.REGISTRY }}/ml-model:${{ github.sha }}
      - name: Deploy to K8s
        run: |
          kubectl set image deployment/ml-model \
            ml-model=${{ secrets.REGISTRY }}/ml-model:${{ github.sha }}
          kubectl rollout status deployment/ml-model
```

#### MLflow Integration
```python
# train.py
import mlflow
import mlflow.sklearn

mlflow.set_tracking_uri("http://mlflow-server:5000")
mlflow.set_experiment("production-model")

with mlflow.start_run(run_name=f"train-{git_sha}") as run:
    mlflow.log_params(params)
    mlflow.log_metrics(metrics)
    mlflow.sklearn.log_model(model, "model", 
                             registered_model_name="my_model")
    
    # Model signature
    from mlflow.models.signature import infer_signature
    signature = infer_signature(X_train, predictions)
    mlflow.sklearn.log_model(model, "model", signature=signature)
```

---

## 4. MONITORING & OBSERVABILITY

### WHY? (Problem Statement)
**Detect: drift, degradation, outliers, system issues.** Before business impact.

### HOW? (Monitoring Stack)

#### Infrastructure Metrics (Prometheus + Grafana)
```yaml
# Prometheus alerts
groups:
- name: ml-model
  rules:
  - alert: HighLatency
    expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
    for: 5m
    labels: { severity: warning }
    annotations:
      summary: "High latency on {{ $labels.instance }}"
      
  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.01
    for: 2m
    labels: { severity: critical }
```

#### ML Metrics (Evidently / WhyLabs / Arize)
```python
# Evidently AI - Data Drift Report
from evidently.dashboard import Dashboard
from evidently.tabs import DataDriftTab, CatTargetDriftTab

dashboard = Dashboard(tabs=[DataDriftTab(), CatTargetDriftTab()])
dashboard.calculate(reference_data, current_data, column_mapping)
dashboard.save("drift_report.html")

# Custom monitoring
def check_drift(reference, current, threshold=0.2):
    from evidently.metrics import DatasetDriftMetric
    from evidently.report import Report
    
    report = Report(metrics=[DatasetDriftMetric()])
    report.run(reference_data=reference, current_data=current)
    
    drift_score = report.as_dict()['metrics'][0]['result']['drift_score']
    if drift_score > threshold:
        alert_oncall(f"Data drift detected: {drift_score:.3f}")
```

#### Business Metrics
```python
# Track in DataDog / custom
def log_business_metrics(prediction, actual, user_id, segment):
    metrics = {
        "prediction": prediction,
        "actual": actual,
        "user_segment": segment,
        "timestamp": datetime.utcnow()
    }
    # Send to metrics store
    metrics_client.write("predictions", metrics)
    
    # Rolling window alerts
    if segment == "premium":
        check_segment_performance("premium", window_hours=24)
```

---

## 5. MODEL VERSIONING & REGISTRY

### WHY? (Problem Statement)
**Reproducibility, rollback, audit trail, governance.**

### HOW? (Versioning Strategy)

#### MLflow Model Registry
```python
# Register model
mlflow.register_model(
    model_uri=f"runs:/{run_id}/model",
    name="fraud-detector"
)

# Transition stages
client = mlflow.tracking.MlflowClient()
client.transition_model_version_stage(
    name="fraud-detector",
    version=3,
    stage="Staging"
)
client.transition_model_version_stage(
    name="fraud-detector",
    version=3,
    stage="Production",
    archive_existing_versions=True
)

# Load from stage
model = mlflow.pyfunc.load_model("models:/fraud-detector/Production")
```

#### DVC for Data Versioning
```bash
# Initialize
dvc init
dvc remote add -d myremote s3://my-bucket/dvc-store

# Track data
dvc add data/raw/train.parquet
git add data/raw/train.parquet.dvc .gitignore
git commit -m "Add training data v1"

# Pipeline
dvc repro  # Reproduce pipeline
dvc push   # Push to remote
```

#### Feature Store (Feast)
```python
# feature_definitions.py
from feast import Entity, FeatureView, Field
from feast.types import Float32, Int64

user = Entity(name="user_id", join_keys=["user_id"])

user_features = FeatureView(
    name="user_features",
    entities=[user],
    ttl=timedelta(days=30),
    schema=[
        Field(name="age", dtype=Int64),
        Field(name="income", dtype=Float32),
        Field(name="tenure_days", dtype=Int64),
    ],
    online=True,
    source=FileSource(
        path="s3://feature-store/user_features.parquet",
        timestamp_field="event_timestamp"
    )
)

# Training: point-in-time correct
training_data = client.get_historical_features(
    entity_df=entity_df,
    features=[
        "user_features:age",
        "user_features:income",
        "user_features:tenure_days"
    ]
)

# Serving: online features
online_features = client.get_online_features(
    entity_rows=[{"user_id": 123}],
    features=["user_features:age", "user_features:income"]
)
```

---

## 6. A/B TESTING & CANARY DEPLOYMENT

### WHY? (Problem Statement)
**Safe rollout, measure impact, quick rollback.**

### HOW? (Strategies)

#### Canary Deployment (K8s)
```yaml
# Argo Rollouts / Flagger
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: ml-model
spec:
  replicas: 10
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 10m}
      - setWeight: 30
      - pause: {duration: 30m}
      - setWeight: 50
      - pause: {duration: 1h}
      - setWeight: 100
      canaryMetadata:
        labels:
          role: canary
      stableMetadata:
        labels:
          role: stable
      trafficRouting:
        istio:
          virtualService:
            name: ml-model
```

#### A/B Testing Framework
```python
class ABTest:
    def __init__(self, model_a_uri, model_b_uri, traffic_split=0.5):
        self.model_a = mlflow.pyfunc.load_model(model_a_uri)
        self.model_b = mlflow.pyfunc.load_model(model_b_uri)
        self.split = traffic_split
    
    def predict(self, features, user_id):
        # Deterministic assignment
        variant = "A" if hash(user_id) % 100 < self.split * 100 else "B"
        model = self.model_a if variant == "A" else self.model_b
        pred = model.predict(features)
        
        # Log for analysis
        self.log_experiment(user_id, variant, pred)
        return pred, variant
    
    def analyze(self, start_date, end_date):
        # Statistical test
        from scipy import stats
        a_outcomes = self.get_outcomes("A", start_date, end_date)
        b_outcomes = self.get_outcomes("B", start_date, end_date)
        
        # Chi-square for conversion
        # t-test for continuous
        stat, p = stats.ttest_ind(a_outcomes, b_outcomes)
        return {"p_value": p, "significant": p < 0.05}
```

---

## 7. DATA PIPELINES & ORCHESTRATION

### WHY? (Problem Statement)
**Reliable, reproducible data flow.** From raw to features to training.

### HOW? (Tools)

#### Airflow (Batch)
```python
# dags/training_pipeline.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'ml-team',
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
}

with DAG(
    'ml_training_pipeline',
    default_args=default_args,
    schedule_interval='@daily',
    start_date=datetime(2024, 1, 1),
    catchup=False
) as dag:

    def extract_data(**context):
        # Extract from warehouse
        pass
    
    def validate_data(**context):
        # Great Expectations
        pass
    
    def feature_engineering(**context):
        # Generate features
        pass
    
    def train_model(**context):
        # Train + log to MLflow
        pass
    
    def evaluate_model(**context):
        # Check thresholds
        pass
    
    def deploy_model(**context):
        # Deploy if passes
        pass

    t1 = PythonOperator(task_id='extract', python_callable=extract_data)
    t2 = PythonOperator(task_id='validate', python_callable=validate_data)
    t3 = PythonOperator(task_id='features', python_callable=feature_engineering)
    t4 = PythonOperator(task_id='train', python_callable=train_model)
    t5 = PythonOperator(task_id='evaluate', python_callable=evaluate_model)
    t6 = PythonOperator(task_id='deploy', python_callable=deploy_model)

    t1 >> t2 >> t3 >> t4 >> t5 >> t6
```

#### Dagster (Modern, Asset-based)
```python
# assets.py
from dagster import asset, AssetIn, Output, MetadataValue
import pandas as pd

@asset
def raw_data() -> pd.DataFrame:
    return pd.read_parquet("s3://bucket/raw/data.parquet")

@asset(ins={"raw_data": AssetIn()})
def validated_data(raw_data: pd.DataFrame) -> pd.DataFrame:
    # Great Expectations validation
    return raw_data

@asset(ins={"validated_data": AssetIn()})
def features(validated_data: pd.DataFrame) -> pd.DataFrame:
    return feature_engineering(validated_data)

@asset(ins={"features": AssetIn()})
def model(features: pd.DataFrame):
    model, metrics = train(features)
    return Output(model, metadata={
        "auc": MetadataValue.float(metrics["auc"]),
        "f1": MetadataValue.float(metrics["f1"])
    })
```

---

## 8. SCALING & PERFORMANCE

### WHY? (Problem Statement)
**Handle load, reduce latency, optimize cost.**

### HOW? (Techniques)

#### Model Optimization
```python
# 1. Quantization (Post-training)
import torch.quantization
model_q = torch.quantization.quantize_dynamic(
    model, {nn.Linear}, dtype=torch.qint8
)

# 2. Quantization Aware Training
model_qat = torch.quantization.prepare_qat(model)
# train...
model_qat = torch.quantization.convert(model_qat)

# 3. ONNX Export + Optimization
import onnxruntime as ort
session = ort.InferenceSession("model.onnx",
    providers=['CUDAExecutionProvider', 'CPUExecutionProvider'])
# Enable graph optimization
sess_options = ort.SessionOptions()
sess_options.graph_optimization_level = ort.GraphOptimizationLevel.ORT_ENABLE_ALL

# 4. TensorRT (NVIDIA GPU)
from torch2trt import torch2trt
model_trt = torch2trt(model, [input_tensor], fp16_mode=True)

# 5. Distillation
# Teacher (large) → Student (small)
student = SmallModel()
loss = distillation_loss(student_output, teacher_output, labels)
```

#### Serving Optimization
```python
# Batching (Triton / custom)
async def batched_predict(requests):
    batch = np.stack([r.features for r in requests])
    preds = model.predict(batch)
    return [{"prediction": float(p)} for p in preds]

# Caching
from functools import lru_cache
@lru_cache(maxsize=10000)
def cached_predict(feature_hash):
    return model.predict(features)

# Async inference
import asyncio
async def async_predict(features):
    loop = asyncio.get_event_loop()
    return await loop.run_in_executor(None, model.predict, features)
```

---

## 9. SECURITY & COMPLIANCE

### WHY? (Problem Statement)
**Protect data, models, and meet regulations (GDPR, HIPAA, SOC2).**

### HOW? (Security Practices)

#### Data Protection
```python
# Encryption at rest
# S3: SSE-KMS, SSE-S3
# Database: TDE

# Encryption in transit
# TLS 1.2+ for all connections
# mTLS for service-to-service

# PII Handling
from presidio_analyzer import AnalyzerEngine
from presidio_anonymizer import AnonymizerEngine

analyzer = AnalyzerEngine()
anonymizer = AnonymizerEngine()

def anonymize_pii(text):
    results = analyzer.analyze(text, language='en')
    return anonymizer.anonymize(text, results).text
```

#### Model Security
```python
# Adversarial robustness
import foolbox as fb
fmodel = fb.PyTorchModel(model, bounds=(0, 1))
attack = fb.attacks.LinfPGD()
adversarials = attack(fmodel, images, labels, epsilons=0.03)

# Membership inference defense
# Differential privacy
from opacus import PrivacyEngine
privacy_engine = PrivacyEngine()
model, optimizer, data_loader = privacy_engine.make_private(
    module=model,
    optimizer=optimizer,
    data_loader=data_loader,
    noise_multiplier=1.0,
    max_grad_norm=1.0
)

# Model watermarking
def embed_watermark(model, trigger_set):
    # Train on trigger set with specific labels
    pass
```

#### Audit & Compliance
```python
# MLflow for audit trail
mlflow.log_param("gdpr_consent", True)
mlflow.log_param("data_version", "v2.3.1")
mlflow.log_param("pii_removed", True)

# Model cards
model_card = f"""
# Model Card: Fraud Detector v3.2
## Intended Use: Real-time transaction scoring
## Training Data: 2023-01 to 2023-12, no PII
## Metrics: AUC=0.97, F1=0.92
## Fairness: Tested across demographics
## Limitations: Not for high-value (>$10k) transactions
"""
```

---

## PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Model Registry** | Files on shared drive | No versioning, no lineage |
| **Automated Retraining** | Manual retrain | Stale models, human error |
| **Canary Deployment** | Direct production push | No rollback, blast radius |
| **Feature Store** | CSV files per project | Inconsistent features, leakage |
| **Data Contracts** | Schema evolution without notice | Broken pipelines |
| **Monitoring Drift** | Deploy and forget | Silent degradation |
| **Infrastructure as Code** | Manual K8s commands | Not reproducible |
| **Secrets Management** | Hardcoded in code | Security breach |
| **Model Cards** | No documentation | Compliance risk, tribal knowledge |
| **Chaos Engineering** | No resilience testing | Unknown failure modes |

---

## INTERVIEW QUESTIONS (Top 15)

### 1. **Deployment**: "How do you deploy a model for real-time inference with <50ms latency?"
```
ARCHITECTURE:
1. MODEL OPTIMIZATION:
   - Export to ONNX → TensorRT (GPU) / ONNX Runtime (CPU)
   - FP16/INT8 quantization
   - Graph optimization (fusion, constant folding)
   
2. SERVING:
   - Triton Inference Server (dynamic batching, model pipeline)
   - OR custom FastAPI + ONNX Runtime + batching
   - gRPC for lower overhead than REST
   
3. INFRASTRUCTURE:
   - GPU: T4/A10G for ML, H100 for large models
   - Right-size: 1-2 CPU cores, 4-8GB RAM per replica
   - HPA: scale on CPU > 70% or queue depth > 100
   
4. CACHING:
   - Redis for repeated queries (feature vectors)
   - LRU cache for recent predictions
   
5. OPTIMIZATIONS:
   - Async I/O (asyncio)
   - Connection pooling
   - Warm-up on startup
   - Profile with py-spy / nsight

TARGET: <10ms model forward, <40ms overhead = <50ms p99
```

### 2. **CI/CD**: "Design a CI/CD pipeline for ML."
```
STAGES:
1. CODE QUALITY (Parallel):
   - Lint (black, flake8, mypy)
   - Unit tests (pytest, >80% coverage)
   - Security scan (bandit, safety)
   - Data validation tests (Great Expectations)
   
2. TRAIN (GPU runner):
   - Reproducible environment (Docker/conda lock)
   - MLflow tracking (params, metrics, artifacts)
   - Model registry (versioning)
   
3. EVALUATE (Gate):
   - Threshold checks: AUC > 0.85, F1 > 0.80
   - Slice-based evaluation (per segment)
   - Bias/fairness checks
   - Comparison vs production (A/B if possible)
   
4. VALIDATE (Staging):
   - Deploy to staging namespace
   - Smoke tests (health, prediction shape)
   - Integration tests (full request flow)
   - Load test (locust/k6) - verify latency SLA
   
5. DEPLOY (Production):
   - Canary (10% → 100% over 2h)
   - Automated rollback on error rate > 1%
   - Post-deploy monitoring (15 min)
   
6. POST-DEPLOY:
   - Drift monitoring (Evidently)
   - Performance dashboards
   - Retraining trigger (weekly/monthly)

TOOLS: GitHub Actions / GitLab CI, MLflow, ArgoCD/Flux, Prometheus/Grafana
```

### 3. **Monitoring**: "How do you detect model drift in production?"
```
DRIFT TYPES & DETECTION:

1. COVARIATE SHIFT (P(X) changes):
   - Feature-level: PSI (Population Stability Index) per feature
   - PSI < 0.1: stable, 0.1-0.2: moderate, >0.2: significant
   - KS test per feature (p < 0.05 = drift)
   - Multivariate: MMD, PCA reconstruction error
   - Frequency: Daily batch job
   
2. PREDICTION DRIFT (P(Ŷ) changes):
   - Monitor prediction distribution (mean, std, quantiles)
   - Compare to training distribution (KL divergence)
   - Alert on shift > threshold
   - Frequency: Hourly
   
3. CONCEPT DRIFT (P(Y|X) changes):
   - Need ground truth labels (delayed)
   - Rolling window metrics (AUC, F1, accuracy)
   - Statistical process control (CUSUM, Page-Hinkley)
   - Frequency: When labels available
   
4. DATA QUALITY:
   - Missing rate, schema violations, range checks
   - Great Expectations / Deequ
   - Frequency: Per batch

RESPONSE:
- Alert → Investigate → Retrain / Feature update / Rollback
- Automated retraining trigger (weekly if drift > threshold)
- Champion/Challenger framework for safe rollout

TOOLS: Evidently AI, WhyLabs, Arize, Fiddler, custom (Prometheus + PSI)
```

### 4. **Scaling**: "Your model serves 1000 req/s with 200ms p99. Need 10K req/s with <50ms."
```
OPTIMIZATION HIERARCHY:

1. MODEL (Biggest impact):
   - Quantize: FP32 → INT8 (2-4x speedup)
   - Distill: Large → Small (10x smaller, 5x faster)
   - Export: PyTorch → ONNX → TensorRT (GPU) / ONNX Runtime (CPU)
   - Prune: Remove redundant weights
   
2. SERVING:
   - Triton: Dynamic batching (batch 32-128)
   - GPU: Maximize utilization (CUDA graphs, persistent kernels)
   - CPU: ONNX Runtime + OpenVINO + thread pooling
   - gRPC + protobuf (vs REST/JSON): 3-5x throughput
   
3. INFRASTRUCTURE:
   - Horizontal: HPA on custom metric (queue depth, GPU util)
   - Vertical: Right-size containers (requests=limits)
   - Node pools: GPU nodes for ML, CPU for preprocessing
   - Placement: Pod affinity to same zone
   
4. APPLICATION:
   - Async I/O (FastAPI + asyncio)
   - Connection pooling (HTTP, DB, Redis)
   - Caching: Redis for embeddings, repeated features
   - Batching: Accumulate requests (1-5ms window)
   
5. ARCHITECTURE:
   - Separate preprocessing (CPU) from inference (GPU)
   - Feature store (online) for low-latency feature lookup
   - Edge caching for geo-distributed users

EXPECTED: 1000 req/s @ 200ms → 10000 req/s @ 30ms (after all optimizations)
```

### 5. **Versioning**: "How do you version data, features, and models together?"
```
VERSIONING STRATEGY:

1. CODE: Git (semantic versioning: v1.2.3)
   - Every commit = potential artifact
   - Tags for releases

2. DATA: DVC / LakeFS / Delta Lake
   - DVC: .dvc files in Git, data in S3/GCS
   - LakeFS: Git-like semantics on object store
   - Delta Lake: Time travel, ACID on Parquet
   - Version: commit hash / timestamp / semantic

3. FEATURES: Feature Store (Feast/Tecton)
   - Feature definitions versioned in Git
   - Feature values: offline (Parquet partitions) + online (Redis/DynamoDB)
   - Point-in-time correctness for training

4. MODELS: MLflow Model Registry
   - Model name + version (1, 2, 3...)
   - Stages: None → Staging → Production → Archived
   - Aliases: champion, challenger
   - Lineage: code version → data version → model version

5. PIPELINE: Dagster / Airflow
   - Asset-based (Dagster) or task-based (Airflow)
   - Each run produces materialized assets with metadata
   - Reproducible: re-execute with same inputs

INTEGRATION:
- Training pipeline: Git SHA + Data Version → Model Version
- Serving: Model Version → Deployed Artifact (Docker image tag)
- Rollback: Docker tag / MLflow stage transition
```

### 6. **A/B Testing**: "How do you run an A/B test for a new ranking model?"
```
A/B TEST DESIGN:

1. HYPOTHESIS:
   - New model improves CTR by ≥1% relative
   - Null: No difference
   
2. RANDOMIZATION:
   - User-level (not session/request!)
   - Consistent assignment: hash(user_id) % 100 < split_pct
   - Pre-period: AA test (both get control) for 1 week

3. SAMPLE SIZE:
   - Baseline CTR: 5%
   - MDE: 1% relative (0.05% absolute)
   - α=0.05, Power=0.8
   - n ≈ 1.5M users per variant (using proportions_ztest)
   
4. METRICS:
   - Primary: CTR (conversion)
   - Secondary: Revenue/session, dwell time, bounce rate
   - Guardrails: Latency, error rate, revenue

5. DURATION:
   - Minimum: 2 weeks (weekly seasonality)
   - Maximum: 4 weeks (peeking penalty)
   - Sequential testing (O'Brien-Fleming) for early stopping

6. ANALYSIS:
   - Intention-to-treat (ITT)
   - Per-protocol (exclude non-compliance)
   - Segment analysis (new/returning, mobile/desktop, geo)
   - Multiple testing correction (Bonferroni)

7. DECISION:
   - p < 0.05 AND practical significance
   - No guardrail violations
   - Consistent across segments

TOOLS: Internal experimentation platform / Eppo / Statsig / GrowthBook
```

### 7. **Feature Store**: "Why do you need a feature store? Build vs Buy?"
```
PROBLEM WITHOUT FEATURE STORE:
- Training-serving skew (different feature logic)
- Duplicate feature engineering across teams
- No feature reuse / discovery
- No point-in-time correctness for training
- Online/offline inconsistency

FEATURE STORE SOLVES:
1. SINGLE SOURCE OF TRUTH:
   - Feature definitions (SQL/Python) versioned in Git
   - Same code for training + serving

2. POINT-IN-TIME CORRECTNESS:
   - Historical features at exact timestamp
   - No leakage from future

3. ONLINE/OFFLINE SYNC:
   - Offline: Parquet in data lake (training)
   - Online: Redis/DynamoDB/Cassandra (serving, <10ms)
   - Automated materialization

4. GOVERNANCE:
   - Ownership, documentation, tags
   - Monitoring (freshness, distribution)
   - Access control

BUILD VS BUY:

BUILD (Feast + Redis + Spark/Flink):
✓ Full control, open source, no vendor lock-in
✓ Custom integrations
✗ Operational burden (infra, scaling, monitoring)
✗ Team needs strong data engineering

BUY (Tecton, Hopsworks, Databricks FS, SageMaker FS):
✓ Managed infra, SLAs, support
✓ Built-in monitoring, CI/CD, GitOps
✓ Advanced: transformations, backfills, streaming
✗ Cost ($$$), vendor lock-in, less flexibility

DECISION:
- <10 ML engineers, standard needs → Buy (Tecton/Databricks)
- >10 ML engineers, custom needs, strong DE team → Build (Feast)
- Start with Feast OSS, migrate if needed
```

### 8. **Security**: "How do you protect PII in ML pipelines?"
```
PII PROTECTION LAYERS:

1. DATA INGESTION:
   - Schema validation (Great Expectations)
   - PII detection (AWS Macie, Google DLP, Presidio)
   - Fail pipeline if unexpected PII

2. PREPROCESSING:
   - Remove: Drop PII columns not needed
   - Hash: Deterministic hash for IDs (join keys)
   - Tokenize: Reversible encryption for needed PII
   - Aggregate: k-anonymity, differential privacy

3. TRAINING:
   - No PII in training features
   - DP-SGD (Opacus) for formal privacy guarantees
   - Federated learning (data never leaves device)

4. INFERENCE:
   - Input validation (reject PII in requests)
   - Output filtering (no PII in predictions)
   - Audit logging (who, what, when)

5. GOVERNANCE:
   - Data catalog with PII tags (Amundsen, DataHub)
   - Access control (RBAC + ABAC)
   - Retention policies (auto-delete)
   - DPIA (Data Protection Impact Assessment)

TOOLS:
- Presidio (Microsoft) - detection/anonymization
- AWS Macie / Google Cloud DLP - cloud-native
- OpenMined PySyft - federated learning
- Opacus (Facebook) - differential privacy
- Great Expectations - validation
```

### 9. **Cost Optimization**: "Your ML inference costs $50K/month. Reduce by 50%."
```
COST REDUCTION STRATEGIES (Priority order):

1. MODEL OPTIMIZATION (Highest ROI):
   - INT8 Quantization: 2-4x throughput, ~50% cost reduction
   - Distillation: 70B → 7B model, 10x cheaper
   - Pruning + Quantization: 50% smaller, minimal accuracy loss
   - Compile: ONNX → TensorRT (GPU) / OpenVINO (CPU)

2. SERVING EFFICIENCY:
   - Dynamic batching (Triton): 5-10x GPU utilization
   - Right-size instances: T4 vs A10G vs A100
   - Spot/Preemptible GPUs: 60-90% discount (fault-tolerant workloads)
   - Scale-to-zero: Knative/KServe for intermittent traffic

3. INFERENCE ARCHITECTURE:
   - Cache: Embeddings, repeated predictions (Redis)
   - Async batching: Accumulate 5-50ms windows
   - Precompute: Offline scoring for batch-eligible use cases
   - Cascade: Small model → Large model (only uncertain)

4. INFRASTRUCTURE:
   - Reserved Instances / Savings Plans: 30-70% off
   - Multi-AZ only for production
   - GPU sharing (MIG on A100): 7x A100 slices
   - ARM-based (Graviton) for CPU workloads: 20% cheaper

5. DATA & TRAINING:
   - Spot instances for training (checkpointing)
   - Gradient accumulation (simulate large batch on small GPU)
   - Mixed precision (FP16/BF16): 2x speed, 50% memory
   - Distributed training (FSDP/DeepSpeed) vs single large GPU

QUICK WINS (Week 1):
- Enable quantization + ONNX Runtime
- Add dynamic batching
- Move batch workloads to spot
- Enable Savings Plans

EXPECTED: 50-70% reduction with minimal accuracy impact
```

### 10. **Reliability**: "Design a self-healing ML system."
```
SELF-HEALING ARCHITECTURE:

1. DETECTION (Multi-layer):
   - INFRA: K8s liveness/readiness, Prometheus alerts
   - MODEL: Drift (Evidently), performance (when labels), latency
   - DATA: Schema validation, freshness, volume anomalies
   - BUSINESS: Conversion drop, revenue anomaly

2. DIAGNOSIS:
   - Automated runbook: "If X alert → check Y dashboard → run Z query"
   - Correlation: Deploy time vs metric change
   - Canary analysis: Compare canary vs stable metrics

3. REMEDIATION (Automated):
   - ROLLBACK: Argo Rollouts / Flagger auto-rollback on SLO breach
   - SCALE: HPA + VPA + Cluster Autoscaler
   - RETRY: Dead letter queues, exponential backoff
   - FAILOVER: Multi-region, active-passive
   - CIRCUIT BREAKER: Hystrix/Resilience4j - fail fast, fallback
   
4. RECOVERY:
   - AUTO-RETRAIN: Trigger when drift > threshold
   - FEATURE STORE: Rebuild materialization
   - DATA: Reprocess from source (idempotent pipelines)
   
5. CHAOS ENGINEERING:
   - LitmusChaos / Chaos Mesh
   - Experiments: Pod kill, network latency, CPU stress, GPU OOM
   - Game days: Monthly, production-like staging

SLOs TO DEFINE:
- Availability: 99.9% (43 min downtime/month)
- Latency: p99 < 100ms
- Freshness: Features < 1 hour old
- Accuracy: AUC > threshold (when labels available)

TOOLS: Prometheus, Grafana, Argo Rollouts, Flagger, LitmusChaos, Evidently
```

### 11. **Orchestration**: "Airflow vs Dagster vs Prefect - when to use which?"
```
AIRFLOW:
✓ Mature, huge community, many operators
✓ Task-based, good for ETL
✗ DAGs as code but not software engineering friendly
✗ No data awareness (tasks don't know about data)
✗ Scaling issues at high concurrency
→ USE: Traditional ETL, team knows Airflow, simple DAGs

DAGSTER:
✓ Asset-based (data-knowedges data lineage
✓ Software-defined assets (SDAs) - testable, reusable
✓ Type system, partitions, backfills
✓ Integrated lineage, observability
✓ Better testing (unit test assets)
✗ Newer, smaller ecosystem
✗ Learning curve
→ USE: Modern ML pipelines, data platforms, need lineage/testing

PREFECT:
✓ Python-first, dynamic workflows
✓ Easy local development, great UI
✓ Hybrid execution (local → cloud)
✓ Good for event-driven, parameterized flows
✗ Less mature orchestration features
→ USE: Dynamic workflows, ML experiments, team prefers Pythonic API

RECOMMENDATION:
- ML-heavy, need lineage/testing → Dagster
- Traditional ETL, stable team → Airflow
- Dynamic, event-driven, fast iteration → Prefect
- Can combine: Airflow for infra, Dagster for ML assets
```

### 12. **Testing**: "How do you test ML code?"
```
TEST PYRAMID FOR ML:

1. UNIT TESTS (Fast, isolated):
   - Preprocessing functions (pure functions)
   - Feature engineering (known input → expected output)
   - Model utilities (loss, metrics, helpers)
   - Config validation
   - Target: >90% coverage, <1 min

2. DATA TESTS (Great Expectations / Pandera):
   - Schema validation (columns, types, nulls)
   - Distribution checks (range, quantiles)
   - Referential integrity (FK, uniqueness)
   - Freshness (max age)
   - Run: Every pipeline run

3. MODEL TESTS (Behavioral):
   - Invariance: Permute irrelevant features → same prediction
   - Directional: Increase income → higher credit score
   - Minimum functionality: Edge cases (empty, NaN, extremes)
   - Adversarial: Known failure cases
   - Slice tests: Performance per segment
   - Tools: Deepchecks, Giskard, custom

4. INTEGRATION TESTS:
   - Training pipeline (mini dataset, 1 epoch)
   - Serving: Load model → predict → validate schema
   - Feature store: Write → Read → Verify
   - End-to-end: Raw data → Features → Model → Prediction

5. CONTRACT TESTS:
   - Model signature (input/output schema)
   - API contract (request/response)
   - Feature store schema

6. A/B TEST VALIDATION:
   - SRM (Sample Ratio Mismatch) check
   - Guardrail metrics
   - Power analysis

CI INTEGRATION:
- Unit + Data tests: Every PR (<5 min)
- Model tests: Nightly / on model change
- Integration: Pre-merge to main
- Contract: Every deploy

TOOLS: pytest, Great Expectations, Pandera, Deepchecks, Giskard, MLflow Projects
```

### 13. **Governance**: "How do you implement ML governance?"
```
GOVERNANCE FRAMEWORK:

1. MODEL LIFECYCLE POLICIES:
   - Development → Validation → Staging → Production → Archived
   - Required approvals per stage
   - Max time in staging (e.g., 30 days)
   - Mandatory retirement review

2. DOCUMENTATION (Model Cards):
   - Intended use, limitations
   - Training data, preprocessing
   - Metrics (overall + slices)
   - Fairness/bias assessment
   - Maintenance plan

3. RISK TIERING:
   - Tier 1 (High): Credit, medical, hiring → Full review
   - Tier 2 (Medium): Recommendations, fraud → Standard review
   - Tier 3 (Low): Internal tools → Lightweight review

4. COMPLIANCE:
   - GDPR: Right to explanation, deletion, portability
   - Fair lending: ECOA, Reg B adverse action codes
   - Model risk: SR 11-7 (banking) - independent validation
   - Audit trail: Who, what, when, why for every change

4. MONITORING & ALERTING:
   - Drift alerts → Auto-create Jira ticket
   - Performance degradation → PagerDuty
   - Quarterly model review calendar

5. ORGANIZATION:
   - Model Review Board (monthly)
   - ML Platform team (paved roads)
   - Data Science → Platform handoff checklist
   - Incident response playbook

TOOLS: MLflow, Kubeflow, Seldon, custom governance layer
```

### 14. **Edge ML**: "Deploy model to mobile/IoT with 10MB size limit."
```
EDGE DEPLOYMENT STRATEGY:

1. MODEL COMPRESSION PIPELINE:
   PyTorch → ONNX → TFLite / CoreML / ONNX Runtime Mobile
   
   Steps:
   a) Pruning: Remove 50-90% weights (magnitude/structured)
   b) Quantization: FP32 → INT8 (post-training or QAT)
   c) Knowledge Distillation: Large → MobileNet/ShuffleNet
   d) Architecture: Depthwise separable convs, inverted residuals
   e) Compile: TFLite delegate (GPU/DSP/NPU)

2. SIZE BUDGET ALLOCATION:
   - Model weights: 8MB (INT8)
   - Metadata/labels: 0.5MB
   - Runtime: 1.5MB (TFLite flatbuffer)
   - Buffer: 0.5MB

3. OPTIMIZATION TECHNIQUES:
   - Weight clustering (k-means on weights)
   - Palettization (1-4 bit)
   - Sparsity + compression (RLE)
   - Operator fusion (Conv+BN+ReLU)

4. HARDWARE ACCELERATION:
   - iOS: Core ML → Neural Engine
   - Android: TFLite GPU delegate / NNAPI / Hexagon DSP
   - Embedded: TensorRT, TVM, STM32Cube.AI
   - WASM: ONNX Runtime Web / TensorFlow.js

5. TESTING:
   - Accuracy: Compare FP32 vs INT8 on validation set
   - Latency: Profile on target devices (p50, p95, p99)
   - Memory: Peak RAM during inference
   - Battery: Energy per inference

FRAMEWORKS:
- TFLite Model Maker / Model Optimization Toolkit
- Core ML Tools (Apple)
- ONNX Runtime Mobile
- PyTorch Mobile / ExecuTorch
- TVM / Apache TVM Unity

TARGET: <10MB, <50ms on 3-year-old phone, <1% accuracy drop
```

### 15. **Future-Proofing**: "How do you design ML systems for the next 3 years?"
```
ARCHITECTURAL PRINCIPLES:

1. MODULARITY & ABSTRACTION:
   - Feature store abstraction (swap backend)
   - Model registry interface (MLflow/Sagemaker/Vertex)
   - Serving interface (Triton/BentoML/KServe)
   - Orchestration interface (Dagster/Airflow/Prefect)
   
2. CONFIGURATION OVER CODE:
   - All hyperparams, paths, versions in config (YAML/Hydra)
   - Environment-specific overlays
   - GitOps for infra + config

3. OBSERVABILITY BY DEFAULT:
   - Structured logging (JSON)
   - Distributed tracing (OpenTelemetry)
   - Metrics (Prometheus) on every component
   - Automated dashboards

4. DATA CONTRACTS:
   - Schema registry (Avro/Protobuf)
   - Breaking change detection in CI
   - Consumer-driven contracts

5. GRADUAL ROLLOUT INFRASTRUCTURE:
   - Canary/Blue-Green built into platform
   - Feature flags for model features
   - Automated rollback

6. MULTI-CLOUD / HYBRID READY:
   - Kubernetes-native (KServe, Kubeflow)
   - Storage abstraction (S3/GCS/Azure compatible)
   - Compute abstraction (Ray, Spark on K8s)

7. CONTINUOUS LEARNING:
   - Automated retraining triggers
   - Shadow deployment for new models
   - Online learning infrastructure (River, Vowpal Wabbit)

8. GENERATIVE AI READY:
   - LLM serving (vLLM, TGI, Triton)
   - RAG pipeline (vector DB, reranker)
   - Prompt management & versioning
   - Guardrails (NeMo Guardrails, Guardrails AI)

TECH STACK 2024-2027:
- Orchestration: Dagster / Airflow 3.0
- Feature Store: Feast / Tecton
- Model Registry: MLflow / Unity Catalog
- Serving: Triton / KServe / vLLM
- Monitoring: Evidently / WhyLabs / custom
- Infra: Terraform + ArgoCD / Flux
- Compute: Ray / Spark on K8s
- Vector DB: Pinecone / Weaviate / Milvus / pgvector
```