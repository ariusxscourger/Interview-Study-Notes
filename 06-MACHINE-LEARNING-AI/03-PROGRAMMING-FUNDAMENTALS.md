# PROGRAMMING FUNDAMENTALS FOR MACHINE LEARNING
## Exam-Style Study Notes

**Roadmap Section:** Programming Fundamentals
**Prerequisites:** Basic programming knowledge (any language)

---

## 1. PYTHON FOR ML

### WHY? (Problem Statement)
**Python is the lingua franca of ML.** Every major framework (PyTorch, TensorFlow, JAX, Scikit-learn) uses Python. You need fluency in:
- Core syntax + data structures
- NumPy/Pandas for data manipulation
- Type hints for maintainable code
- Testing and debugging

### HOW? (Core Python for ML)

#### Essential Syntax Patterns
```python
# List/Dict/Set comprehensions (idiomatic, fast)
squares = [x**2 for x in range(10)]
even_squares = {x: x**2 for x in range(10) if x % 2 == 0}

# Unpacking
a, b, *rest = [1, 2, 3, 4, 5]  # a=1, b=2, rest=[3,4,5]
first, *middle, last = range(10)

# Walrus operator (Python 3.8+)
if (n := len(data)) > 100:
    print(f"Large dataset: {n}")

# f-strings (formatting)
name = "model"; acc = 0.95
print(f"{name}: accuracy={acc:.2%}")

# Type hints (Python 3.5+)
from typing import List, Dict, Tuple, Optional, Union
def process_batch(batch: List[Dict[str, float]]) -> Tuple[np.ndarray, Optional[str]]:
    ...

# Context managers (resource management)
from contextlib import contextmanager

@contextmanager
def timer():
    start = time.time()
    yield
    print(f"Elapsed: {time.time() - start:.3f}s")

with timer():
    model.train()
```

#### Data Structures for ML
| Structure | Use Case | Complexity |
|-----------|----------|------------|
| **list** | Ordered, mutable sequences | O(1) append, O(n) search |
| **dict** | Key-value mapping (hash table) | O(1) lookup/insert |
| **set** | Unique elements | O(1) membership |
| **tuple** | Immutable sequences, keys | O(1) access, hashable |
| **collections.deque** | Queue/stack | O(1) both ends |
| **collections.Counter** | Counting | O(n) build, O(1) access |
| **collections.defaultdict** | Auto-initializing dict | Cleaner code |
| **heapq** | Priority queue | O(log n) push/pop |

```python
# heapq for top-k (common in ML)
import heapq

# Top-k largest
top_k = heapq.nlargest(k, iterable, key=lambda x: x.score)

# Min-heap for streaming top-k
heap = []
for item in stream:
    heapq.heappush(heap, (item.score, item))
    if len(heap) > k:
        heapq.heappop(heap)

# Counter for class balancing
from collections import Counter
class_counts = Counter(labels)
weights = {cls: 1.0/count for cls, count in class_counts.items()}
```

#### Functions & Functional Patterns
```python
# Higher-order functions
from functools import partial, reduce
from operator import add, mul

# Partial application (common in ML pipelines)
normalize = partial((lambda x, mean, std: (x - mean) / std), mean=0.5, std=0.25)

# Decorators (cross-cutting concerns)
import functools
import time

def retry(max_attempts=3, delay=1.0):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise
                    time.sleep(delay * (2 ** attempt))  # Exponential backoff
        return wrapper
    return decorator

# Lambda functions (inline, throwaway)
sort_by_loss = lambda x: x.loss
key_fn = lambda x: (x.category, -x.score)  # Sort by category, then score desc
```

#### Error Handling & Logging
```python
import logging
from typing import NoReturn

# Structured logging (JSON for production)
import json
import sys

class JsonFormatter(logging.Formatter):
    def format(self, record):
        log_obj = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno,
        }
        if record.exc_info:
            log_obj["exception"] = self.formatException(record.exc_info)
        return json.dumps(log_obj)

handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(JsonFormatter())
logger = logging.getLogger(__name__)
logger.addHandler(handler)
logger.setLevel(logging.INFO)

# Custom exceptions
class ModelError(Exception):
    """Base model exception"""
    pass

class DataValidationError(ModelError):
    """Data failed validation"""
    pass

class ModelNotFoundError(ModelError):
    """Model artifact not found"""
    pass

# Assertions for contracts
def train_model(X: np.ndarray, y: np.ndarray) -> Model:
    assert X.ndim == 2, f"Expected 2D array, got {X.ndim}D"
    assert len(X) == len(y), "X and y length mismatch"
    assert not np.any(np.isnan(X)), "NaN values in features"
    ...
```

---

## 2. OBJECT-ORIENTED PROGRAMMING FOR ML

### WHY? (Problem Statement)
**ML systems need clean abstractions.** OOP provides:
- Encapsulation (data + behavior together)
- Inheritance (reuse, polymorphism)
- Composition over inheritance (flexible pipelines)
- Design patterns (Strategy, Factory, Template Method)

### HOW? (OOP Patterns in ML)

#### Base Classes & Interfaces
```python
from abc import ABC, abstractmethod
from typing import Protocol

# Abstract Base Class (ABC) - for inheritance
class BaseModel(ABC):
    @abstractmethod
    def fit(self, X: np.ndarray, y: np.ndarray) -> "BaseModel":
        pass
    
    @abstractmethod
    def predict(self, X: np.ndarray) -> np.ndarray:
        pass
    
    def fit_predict(self, X: np.ndarray, y: np.ndarray) -> np.ndarray:
        return self.fit(X, y).predict(X)
    
    def score(self, X: np.ndarray, y: np.ndarray) -> float:
        pred = self.predict(X)
        return accuracy_score(y, pred)

# Protocol (Structural Typing) - for duck typing
class Predictor(Protocol):
    def predict(self, X: np.ndarray) -> np.ndarray: ...
    def predict_proba(self, X: np.ndarray) -> np.ndarray: ...

# Runtime checkable
from typing import runtime_checkable

@runtime_checkable
class Trainable(Protocol):
    def fit(self, X: np.ndarray, y: np.ndarray) -> "Trainable": ...
```

#### Composition over Inheritance (Pipeline Pattern)
```python
from sklearn.base import BaseEstimator, TransformerMixin
from sklearn.pipeline import Pipeline

# Transformer interface (fit/transform)
class StandardScaler(BaseEstimator, TransformerMixin):
    def fit(self, X, y=None):
        self.mean_ = X.mean(axis=0)
        self.scale_ = X.std(axis=0)
        return self
    
    def transform(self, X):
        return (X - self.mean_) / self.scale_

# Pipeline composes transformers + estimator
pipeline = Pipeline([
    ("imputer", SimpleImputer(strategy="median")),
    ("scaler", StandardScaler()),
    ("selector", SelectKBest(f_classif, k=20)),
    ("model", LogisticRegression())
])

# Fit/transform flows through pipeline
pipeline.fit(X_train, y_train)
pred = pipeline.predict(X_test)
```

#### Design Patterns in ML

**Strategy Pattern** (Algorithm selection)
```python
from abc import ABC, abstractmethod

class Optimizer(ABC):
    @abstractmethod
    def update(self, params: List[np.ndarray], grads: List[np.ndarray]) -> None:
        pass

class SGD(Optimizer):
    def __init__(self, lr=0.01, momentum=0.9):
        self.lr = lr
        self.momentum = momentum
        self.velocity = None
    
    def update(self, params, grads):
        if self.velocity is None:
            self.velocity = [np.zeros_like(p) for p in params]
        for i, (p, g) in enumerate(zip(params, grads)):
            self.velocity[i] = self.momentum * self.velocity[i] - self.lr * g
            params[i] += self.velocity[i]

class Adam(Optimizer):
    def __init__(self, lr=0.001, beta1=0.9, beta2=0.999, eps=1e-8):
        ...
    
    def update(self, params, grads):
        ...

# Model uses optimizer strategy
class NeuralNetwork:
    def __init__(self, optimizer: Optimizer):
        self.optimizer = optimizer
    
    def train_step(self, X, y):
        grads = self.compute_gradients(X, y)
        self.optimizer.update(self.params, grads)
```

**Factory Pattern** (Model creation)
```python
from typing import Dict, Type
from enum import Enum

class ModelType(Enum):
    LOGISTIC = "logistic"
    XGBOOST = "xgboost"
    NEURAL_NET = "neural_net"

class ModelFactory:
    _registry: Dict[ModelType, Type[BaseModel]] = {}
    
    @classmethod
    def register(cls, model_type: ModelType):
        def decorator(model_class: Type[BaseModel]):
            cls._registry[model_type] = model_class
            return model_class
        return decorator
    
    @classmethod
    def create(cls, model_type: ModelType, **kwargs) -> BaseModel:
        if model_type not in cls._registry:
            raise ValueError(f"Unknown model type: {model_type}")
        return cls._registry[model_type](**kwargs)

# Usage
@ModelFactory.register(ModelType.LOGISTIC)
class LogisticModel(BaseModel):
    ...

@ModelFactory.register(ModelType.XGBOOST)
class XGBoostModel(BaseModel):
    ...

model = ModelFactory.create(ModelType.XGBOOST, n_estimators=100)
```

**Template Method Pattern** (Training loop skeleton)
```python
from abc import ABC, abstractmethod

class BaseTrainer(ABC):
    def train(self, train_loader, val_loader, epochs=100):
        for epoch in range(epochs):
            self.train_epoch(train_loader)
            val_metrics = self.validate(val_loader)
            self.on_epoch_end(epoch, val_metrics)
            
            if self.should_stop(val_metrics):
                break
    
    @abstractmethod
    def train_epoch(self, loader): ...
    
    @abstractmethod
    def validate(self, loader): ...
    
    def on_epoch_end(self, epoch, metrics):  # Hook
        pass
    
    def should_stop(self, metrics) -> bool:  # Hook
        return False

class EarlyStoppingTrainer(BaseTrainer):
    def __init__(self, patience=10):
        self.patience = patience
        self.best_score = -np.inf
        self.counter = 0
    
    def on_epoch_end(self, epoch, metrics):
        if metrics["val_auc"] > self.best_score:
            self.best_score = metrics["val_auc"]
            self.counter = 0
        else:
            self.counter += 1
    
    def should_stop(self, metrics):
        return self.counter >= self.patience
```

---

## 3. SQL FOR ML

### WHY? (Problem Statement)
**Data lives in databases.** ML engineers must extract, transform, and load data efficiently.

### HOW? (SQL Patterns for ML)

#### Essential Queries
```sql
-- Feature engineering in SQL
SELECT 
    user_id,
    COUNT(*) as total_orders,
    AVG(order_value) as avg_order_value,
    MAX(order_date) - MIN(order_date) as customer_lifetime_days,
    COUNT(DISTINCT product_category) as num_categories
FROM orders
WHERE order_date >= '2023-01-01'
GROUP BY user_id
HAVING COUNT(*) >= 5;

-- Time series features (window functions)
SELECT 
    user_id,
    order_date,
    order_value,
    AVG(order_value) OVER (PARTITION BY user_id ORDER BY order_date 
                           ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as rolling_7day_avg,
    LAG(order_value) OVER (PARTITION BY user_id ORDER BY order_date) as prev_order_value
FROM orders;

-- Train/test split by time (NO random split for time series!)
WITH ranked AS (
    SELECT *, 
           ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_date) as rn,
           COUNT(*) OVER (PARTITION BY user_id) as total
    FROM orders
)
SELECT * FROM ranked WHERE rn <= total * 0.8  -- Train (first 80%)
-- vs
SELECT * FROM ranked WHERE rn > total * 0.8   -- Test (last 20%)

-- Sampling for large tables
SELECT * FROM large_table TABLESAMPLE BERNOULLI (10)  -- 10% sample
SELECT * FROM large_table TABLESAMPLE SYSTEM (10)     -- Block sampling (faster)

-- Pivot for wide features
SELECT 
    user_id,
    SUM(CASE WHEN category = 'electronics' THEN 1 ELSE 0 END) as electronics_count,
    SUM(CASE WHEN category = 'clothing' THEN 1 ELSE 0 END) as clothing_count,
    SUM(CASE WHEN category = 'books' THEN 1 ELSE 0 END) as books_count
FROM orders
GROUP BY user_id;
```

#### Connecting from Python
```python
import pandas as pd
from sqlalchemy import create_engine, text
from sqlalchemy.pool import NullPool

# Connection with connection pooling
engine = create_engine(
    "postgresql://user:pass@host:5432/db",
    poolclass=NullPool,  # Avoid pool issues in multiprocessing
    connect_args={"connect_timeout": 10}
)

# Read in chunks (memory efficient)
chunk_iter = pd.read_sql(
    "SELECT * FROM large_table WHERE date >= '2023-01-01'",
    engine,
    chunksize=100000
)

dfs = []
for chunk in chunk_iter:
    # Process chunk
    processed = feature_engineer(chunk)
    dfs.append(processed)

df = pd.concat(dfs, ignore_index=True)

# Parameterized queries (prevent SQL injection)
query = text("SELECT * FROM orders WHERE user_id = :uid AND date >= :start_date")
df = pd.read_sql(query, engine, params={"uid": 123, "start_date": "2023-01-01"})

# Write back (predictions)
df.to_sql("predictions", engine, if_exists="append", index=False, chunksize=1000)
```

---

## 4. APIs & WEBSERVICES

### WHY? (Problem Statement)
**ML models serve via APIs.** Need to build, consume, and version APIs.

### HOW? (API Patterns)

#### REST API Design (FastAPI Example)
```python
from fastapi import FastAPI, HTTPException, Depends, BackgroundTasks
from pydantic import BaseModel, Field
from typing import List, Optional
import mlflow.pyfunc

app = FastAPI(title="ML Model API", version="1.0.0")

# Request/Response schemas
class PredictionRequest(BaseModel):
    features: List[float] = Field(..., min_length=10, max_length=10)
    user_id: Optional[str] = None

class PredictionResponse(BaseModel):
    prediction: float
    probability: float
    model_version: str
    request_id: str

class BatchPredictionRequest(BaseModel):
    instances: List[PredictionRequest] = Field(..., min_length=1, max_length=1000)

# Dependency injection for model loading
def get_model():
    if not hasattr(get_model, "_model"):
        get_model._model = mlflow.pyfunc.load_model("models:/my_model/Production")
    return get_model._model

@app.post("/predict", response_model=PredictionResponse)
async def predict(
    request: PredictionRequest,
    model=Depends(get_model),
    background_tasks: BackgroundTasks = None
):
    import uuid
    import time
    
    request_id = str(uuid.uuid4())
    start = time.time()
    
    # Prepare features
    features = np.array(request.features).reshape(1, -1)
    
    # Predict
    prob = model.predict_proba(features)[0, 1]
    pred = float(prob > 0.5)
    
    # Log async
    if background_tasks:
        background_tasks.add_task(
            log_prediction, request_id, request.features, pred, prob, time.time() - start
        )
    
    return PredictionResponse(
        prediction=pred,
        probability=prob,
        model_version=model.metadata.get("version", "unknown"),
        request_id=request_id
    )

@app.post("/batch_predict")
async def batch_predict(request: BatchPredictionRequest, model=Depends(get_model)):
    features = np.array([inst.features for inst in request.instances])
    probs = model.predict_proba(features)[:, 1]
    preds = (probs > 0.5).astype(int)
    
    return [
        PredictionResponse(
            prediction=float(p), 
            probability=float(prob),
            model_version="1.0",
            request_id=str(uuid.uuid4())
        )
        for p, prob in zip(preds, probs)
    ]

# Health check
@app.get("/health")
async def health():
    return {"status": "healthy", "model_loaded": hasattr(get_model, "_model")}
```

#### Async Patterns & Caching
```python
import asyncio
from functools import lru_cache
from cachetools import TTLCache, cached

# Async prediction for high throughput
@app.post("/async_predict")
async def async_predict(request: PredictionRequest, model=Depends(get_model)):
    loop = asyncio.get_event_loop()
    prob = await loop.run_in_executor(
        None, 
        lambda: model.predict_proba(np.array(request.features).reshape(1, -1))[0, 1]
    )
    return {"probability": float(prob)}

# Caching for repeated requests
prediction_cache = TTLCache(maxsize=10000, ttl=300)  # 5 min TTL

@cached(prediction_cache)
def cached_predict(features_hash: str, features: tuple) -> float:
    model = get_model()
    return float(model.predict_proba(np.array(features).reshape(1, -1))[0, 1])

@app.post("/cached_predict")
async def cached_predict_endpoint(request: PredictionRequest):
    features_hash = hashlib.md5(str(request.features).encode()).hexdigest()
    prob = cached_predict(features_hash, tuple(request.features))
    return {"probability": prob, "cached": True}
```

#### gRPC for High-Performance Internal Services
```protobuf
// model.proto
syntax = "proto3";

package ml;

service ModelService {
  rpc Predict(PredictRequest) returns (PredictResponse);
  rpc BatchPredict(BatchPredictRequest) returns (BatchPredictResponse);
  rpc HealthCheck(HealthCheckRequest) returns (HealthCheckResponse);
}

message PredictRequest {
  repeated float features = 1;
  string user_id = 2;
}

message PredictResponse {
  float prediction = 1;
  float probability = 2;
  string model_version = 3;
  string request_id = 3;
}

message BatchPredictRequest {
  repeated PredictRequest instances = 1;
}

message BatchPredictResponse {
  repeated PredictResponse predictions = 1;
}
```

```python
# gRPC server implementation
import grpc
from concurrent import futures
import model_pb2
import model_pb2_grpc

class ModelServicer(model_pb2_grpc.ModelServiceServicer):
    def __init__(self, model):
        self.model = model
    
    def Predict(self, request, context):
        features = np.array(request.features).reshape(1, -1)
        prob = self.model.predict_proba(features)[0, 1]
        pred = float(prob > 0.5)
        
        return model_pb2.PredictResponse(
            prediction=pred,
            probability=prob,
            model_version="1.0",
            request_id=str(uuid.uuid4())
        )
    
    def BatchPredict(self, request, context):
        features = np.array([list(inst.features) for inst in request.instances])
        probs = self.model.predict_proba(features)[:, 1]
        preds = (probs > 0.5).astype(int)
        
        return model_pb2.BatchPredictResponse(
            predictions=[
                model_pb2.PredictResponse(
                    prediction=float(p), probability=float(prob), model_version="1.0"
                )
                for p, prob in zip(preds, probs)
            ]
        )

def serve(model, port=50051):
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    model_pb2_grpc.add_ModelServiceServicer_to_server(ModelServicer(model), server)
    server.add_insecure_port(f"[::]:{port}")
    server.start()
    server.wait_for_termination()
```

---

## PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Type hints everywhere** | No type hints | Bugs caught late, poor IDE support |
| **List/dict comprehensions** | `map`/`filter` with lambdas | More readable, faster |
| **Context managers** | Manual open/close | Resource leaks |
| **Pipeline composition** | Monolithic functions | Hard to test, reuse |
| **Protocol/ABC** | Concrete class coupling | Hard to test, swap |
| **Vectorized NumPy** | Python loops on arrays | 100x slower |
| **Parameterized SQL** | String formatting | SQL injection |
| **Async for I/O** | Sync blocking calls | Low throughput |
| **Structured logging** | Print statements | Unsearchable logs |

---

## INTERVIEW QUESTIONS (Top 15)

### 1. **Python**: "What's the difference between list and generator? When to use each?"

```
LIST:
- Eager evaluation: computes all values immediately
- Memory: O(n) stores all elements
- Reusable: can iterate multiple times
- Use when: need random access, multiple passes, small/medium data

GENERATOR:
- Lazy evaluation: yields one value at a time
- Memory: O(1) constant
- Single use: exhausted after one iteration
- Use when: large/infinite sequences, memory constrained, streaming

EXAMPLES:
# List comprehension - eager
squares = [x**2 for x in range(1000000)]  # Allocates ~8MB

# Generator expression - lazy
squares_gen = (x**2 for x in range(1000000))  # ~200 bytes

# When to use generator:
def stream_data(file_path):
    with open(file_path) as f:
        for line in f:
            yield process(line)  # Memory efficient

# Chain generators (pipeline)
def pipeline(data):
    yield from filter(valid, map(transform, data))
```

### 2. **OOP**: "Composition vs Inheritance - when to use which?"

```
INHERITANCE (Is-A relationship):
✓ Base class defines common interface/behavior
✓ Subclasses specialize behavior
✓ Liskov Substitution Principle holds
Example: BaseModel → LogisticModel, XGBoostModel

COMPOSITION (Has-A relationship):
✓ Flexible, runtime configuration
✓ Single Responsibility Principle
✓ Avoids fragile base class problem
Example: Pipeline(steps=[Imputer(), Scaler(), Model()])

RULE: Prefer composition. Use inheritance only when:
- Clear "is-a" relationship
- Need polymorphism (same interface, different impl)
- Base class is stable (won't change often)

ANTI-PATTERN: Deep inheritance hierarchies
BaseModel → TreeModel → XGBoostModel → TunedXGBoostModel → ...
→ Fragile, hard to test, hard to change
```

### 3. **SQL**: "How do you prevent data leakage in train/test split?"

```
TIME SERIES: Never use random split!
✓ Use time-based split: train on past, test on future
✓ Use expanding window or sliding window CV

GROUPED DATA: Never split groups!
✓ Use GroupKFold (user_id, session_id, customer_id)
✓ All rows of a group must be in same split

FEATURE ENGINEERING:
✗ Fit scaler on full dataset → transform train/test
✓ Fit scaler on TRAIN only → transform train AND test

✗ Feature selection on full data
✓ Feature selection INSIDE CV loop

TARGET ENCODING:
✗ Compute target mean on full data
✓ Compute on train fold only, apply to val fold
✓ Use smoothing (add prior) to prevent overfitting

PIPELINE PATTERN (prevents leakage):
pipeline = Pipeline([
    ("imputer", SimpleImputer()),
    ("scaler", StandardScaler()),
    ("selector", SelectKBest(k=10)),
    ("model", LogisticRegression())
])
cross_val_score(pipeline, X, y, cv=StratifiedKFold(5))  # Safe!
```

### 4. **APIs**: "REST vs gRPC - when to use which?"

```
REST (HTTP/JSON):
✓ Universal, firewall-friendly, human-readable
✓ Caching, CDN support
✓ Browser/JavaScript native
✓ Public APIs, webhooks, external integrations
✗ No built-in contract, verbose, no streaming

gRPC (HTTP/2/Protobuf):
✓ Contract-first (proto file), code generation
✓ 5-10x faster, smaller payloads
✓ Streaming (unary, server, client, bidirectional)
✓ Strong typing, IDE support
✗ No browser support (needs grpc-web), debugging harder
✗ No CDN caching

USE REST WHEN:
- Public APIs, mobile/web clients
- Need caching/CDN
- Simple CRUD, low latency not critical

USE gRPC WHEN:
- Internal microservices
- High throughput, low latency required
- Streaming (real-time, large data)
- Polyglot environments (Python, Go, Java, etc.)

HYBRID: REST gateway (Envoy) → gRPC internal services
```

### 5. **Python**: "Explain GIL and its impact on ML."

```
GLOBAL INTERPRETER LOCK (GIL):
- Only one thread executes Python bytecode at a time
- CPU-bound multi-threading = NO speedup (can be slower due to context switching)
- I/O-bound multi-threading = OK (GIL released during I/O)

IMPACT ON ML:
- Data loading: Use multiprocessing (DataLoader num_workers > 0)
- Training: Use multiprocessing or release GIL (NumPy, PyTorch, TensorFlow do this in C)
- Inference serving: Use multiprocessing (gunicorn workers) or async (FastAPI)

WORKAROUNDS:
1. Multiprocessing (separate processes, each has own GIL)
   from multiprocessing import Pool
   with Pool(4) as p: results = p.map(func, data)

2. Release GIL in C extensions (NumPy, Pandas, PyTorch do this)
   - NumPy operations release GIL → true parallelism

3. Thread pools for I/O only
   from concurrent.futures import ThreadPoolExecutor

4. Alternative interpreters: PyPy (JIT, no GIL in some cases), Sub-interpreters (3.12+)
```

### 6. **OOP**: "Explain the Strategy Pattern with ML example."

```
STRATEGY PATTERN:
- Define family of algorithms
- Encapsulate each one
- Make them interchangeable
- Client uses strategy interface

ML EXAMPLE: Optimizer Strategy
class Optimizer(ABC):
    @abstractmethod
    def update(self, params, grads): ...

class SGD(Optimizer):
    def update(self, params, grads):
        for p, g in zip(params, grads):
            p -= self.lr * g

class Adam(Optimizer):
    def update(self, params, grads):
        # Adam update logic

class Model:
    def __init__(self, optimizer: Optimizer):
        self.optimizer = optimizer  # Inject strategy
    
    def train_step(self, X, y):
        grads = self.compute_gradients(X, y)
        self.optimizer.update(self.params, grads)

# Usage
model = Model(optimizer=Adam(lr=0.001))
model.train_step(X, y)

# Easy to swap
model.optimizer = SGD(lr=0.01)
```

### 7. **SQL**: "Explain window functions and their ML use cases."

```
WINDOW FUNCTIONS: Perform calculation across related rows
Syntax: FUNCTION() OVER (PARTITION BY col ORDER BY col 
                         ROWS/RANGE BETWEEN ...)

KEY FUNCTIONS:
- ROW_NUMBER() / RANK() / DENSE_RANK() - Ranking
- LAG() / LEAD() - Access previous/next row
- FIRST_VALUE() / LAST_VALUE() - First/last in window
- AVG() / SUM() / MIN() / MAX() - Aggregations over window
- NTILE(n) - Bucket into n groups

ML USE CASES:
1. Time series features:
   SELECT user_id, 
          order_date,
          LAG(amount) OVER (PARTITION BY user_id ORDER BY date) as prev_amount,
          AVG(amount) OVER (PARTITION BY user_id 
                            ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as rolling_7day_avg
   FROM transactions;

2. Sessionization:
   SELECT *,
          SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY timestamp) as session_id
   FROM (
     SELECT *,
            CASE WHEN timestamp - LAG(timestamp) OVER (PARTITION BY user_id ORDER BY timestamp) > 1800 
                 THEN 1 ELSE 0 END as is_new_session
     FROM events
   );

3. Feature ranking:
   SELECT *,
          ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY prediction_score DESC) as rank
   FROM predictions;
```

### 8. **Python**: "How do you handle memory efficiently with large datasets?"

```
MEMORY EFFICIENCY TECHNIQUES:

1. CHUNKED PROCESSING:
   for chunk in pd.read_sql(query, engine, chunksize=100000):
       process(chunk)
       write_output(chunk)  # Don't accumulate!

2. GENERATORS (lazy evaluation):
   def data_generator(file_paths):
       for path in file_paths:
           with open(path) as f:
               for line in f:
                   yield parse(line)

3. DASK / POLARS (out-of-core):
   import polars as pl
   df = pl.scan_parquet("s3://bucket/data/*.parquet")
   result = df.filter(pl.col("value") > 100).collect()

4. NUMPY MEMORY VIEWS (avoid copies):
   arr = np.memmap("large_file.dat", dtype=np.float32, mode="r", shape=(n, d))
   # Access slices without loading entire file

5. DTYPE OPTIMIZATION:
   df["category"] = df["category"].astype("category")  # Saves memory
   df["flag"] = df["flag"].astype("bool")
   df["id"] = df["id"].astype("int32")  # vs int64

6. DELETE UNUSED VARIABLES:
   del large_df
   import gc; gc.collect()

7. CHUNKED MODEL TRAINING:
   for chunk in pd.read_csv("large.csv", chunksize=10000):
       model.partial_fit(chunk[X_cols], chunk[y_col])
```

### 9. **APIs**: "How do you version ML model APIs?"

```
VERSIONING STRATEGIES:

1. URL VERSIONING (explicit, clear):
   /v1/predict, /v2/predict
   Pros: Clear, cacheable, easy to route
   Cons: URL pollution

2. HEADER VERSIONING:
   Accept: application/vnd.company.v2+json
   Pros: Clean URLs, flexible
   Cons: Harder to test/debug

3. QUERY PARAMETER:
   /predict?version=2
   Pros: Simple
   Cons: Not cacheable by default

MODEL VERSIONING (separate from API version):
- Model registry (MLflow, Weights & Biases, custom)
- Each model has: version, metadata, artifacts
- API serves specific model version
- Canary: route 5% traffic to new version
- Rollback: switch traffic back

FASTAPI EXAMPLE:
app = FastAPI()
v1_router = APIRouter(prefix="/v1")
v2_router = APIRouter(prefix="/v2")

@v1_router.post("/predict")
def predict_v1(request: PredictRequestV1): ...

@v2_router.post("/predict")
def predict_v2(request: PredictRequestV2): ...

app.include_router(v1_router)
app.include_router(v2_router)
```

### 10. **Python**: "Explain context managers and their ML use cases."

```
CONTEXT MANAGERS: Setup/teardown guaranteed execution
with open("file.txt") as f:  # __enter__ opens, __exit__ closes
    data = f.read()

ML USE CASES:

1. TIMER / PROFILING:
class Timer:
    def __enter__(self): self.start = time.time(); return self
    def __exit__(self, *args): self.elapsed = time.time() - self.start

with Timer() as t:
    model.train()
print(f"Training took {t.elapsed:.2f}s")

2. GPU MEMORY MANAGEMENT:
import torch
class GPUMemory:
    def __enter__(self): torch.cuda.empty_cache(); return self
    def __exit__(self, *args): torch.cuda.empty_cache()

with GPUMemory():
    large_tensor = torch.randn(10000, 10000, device="cuda")

3. TEMPORARY FILES/DIRS:
import tempfile
with tempfile.TemporaryDirectory() as tmpdir:
    model.save(tmpdir)
    upload_to_s3(tmpdir)

4. MLFLOW RUNS:
import mlflow
with mlflow.start_run() as run:
    mlflow.log_param("lr", 0.01)
    mlflow.log_metric("auc", 0.95)
    mlflow.sklearn.log_model(model, "model")

5. CUSTOM CONTEXT MANAGER:
from contextlib import contextmanager

@contextmanager
def timer(name: str):
    start = time.time()
    try:
        yield
    finally:
        logger.info(f"{name} took {time.time() - start:.3f}s")

with timer("data_loading"):
    df = pd.read_parquet("s3://bucket/data.parquet")
```

### 11. **SQL**: "How do you handle categorical variables with high cardinality?"

```
HIGH CARDINALITY PROBLEMS:
- One-hot: Explodes dimensions (100k categories → 100k cols)
- Target encoding: Leakage risk if not done properly
- Embedding: Requires NN, fixed vocab size

SOLUTIONS:

1. TARGET ENCODING (with smoothing):
   mean_target = y.mean()
   smooth = (count * category_mean + α * mean_target) / (count + α)
   
   # sklearn
   from category_encoders import TargetEncoder
   encoder = TargetEncoder(smoothing=10, min_samples_leaf=20)

2. FEATURE HASHING (fixed dim):
   from sklearn.feature_extraction import FeatureHasher
   hasher = FeatureHasher(n_features=256, input_type="string")
   X_hashed = hasher.transform(categorical_series)
   # Collisions possible but bounded memory

3. EMBEDDING (learned):
   # In NN: nn.Embedding(num_categories, embedding_dim)
   # Handle OOV with reserved index 0

4. FREQUENCY ENCODING:
   freq_map = df[col].value_counts(normalize=True)
   df[col + "_freq"] = df[col].map(freq_map)

5. LEAVE-ONE-OUT ENCODING (CV-safe):
   # Compute target mean excluding current row
   # Use within CV loop only!

BEST PRACTICE: 
- Low cardinality (<20): One-hot
- Medium (20-1000): Target encoding + smoothing
- High (>1000): Feature hashing or embedding
- Always: Fit on train fold only, apply to val/test
```

### 12. **APIs**: "How do you handle model updates without downtime?"

```
ZERO-DOWNTIME DEPLOYMENT PATTERNS:

1. BLUE/GREEN:
   - Two identical environments
   - Deploy to idle, test, switch traffic (DNS/LB)
   - Instant rollback
   - Cost: 2x infrastructure

2. CANARY (Progressive Delivery):
   - Deploy new version alongside old
   - Route 5% → 25% → 50% → 100% traffic
   - Automated rollback on SLO breach
   - Tools: Flagger, Argo Rollouts, Istio, Flux

3. FEATURE FLAGS:
   - Deploy code dark (flag OFF)
   - Enable for internal → beta → 10% → 100%
   - Decouple deploy from release
   - Tools: LaunchDarkly, Unleash, OpenFeature

4. ROLLING UPDATE (K8s default):
   - maxSurge: 25%, maxUnavailable: 25%
   - New pods ready → old pods terminate
   - Requires: readiness probes, graceful shutdown

REQUIREMENTS FOR ALL:
✓ Backward-compatible API changes (additive only)
✓ Database migrations: Expand → Migrate → Contract
✓ Idempotent consumers (safe replay)
✓ Automated rollback on SLO breach
✓ Observability: deployment annotations in metrics

K8s EXAMPLE (Canary with Argo Rollouts):
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  replicas: 10
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 5m}
      - setWeight: 30
      - pause: {duration: 10m}
      - setWeight: 50
      - pause: {duration: 10m}
      - setWeight: 100
      analysis:
        templates:
        - templateName: success-rate
```

### 13. **Python**: "How do you test ML code?"

```
TESTING PYRAMID FOR ML:

1. UNIT TESTS (70-80%):
   - Pure functions: preprocessing, metrics, utilities
   - Fast (<1ms), isolated, deterministic
   - pytest + hypothesis (property-based)

2. INTEGRATION TESTS (15-20%):
   - Model + real dependencies (DB, Redis)
   - Testcontainers for ephemeral infra
   - Contract tests (Pact)

3. CONTRACT TESTS:
   - Consumer defines expectations
   - Provider verifies
   - Prevents breaking changes

4. DATA TESTS:
   - Great Expectations / Pandera
   - Schema validation, distribution checks

4. MODEL TESTS:
   - Behavioral tests (invariance, directionality)
   - Slice-based evaluation (fairness)
   - Adversarial examples

PYTEST EXAMPLES:
import pytest
import numpy as np

# Unit test
def test_preprocessing_scaler():
    scaler = StandardScaler()
    X = np.array([[1, 2], [3, 4], [5, 6]])
    scaled = scaler.fit_transform(X)
    assert np.allclose(scaled.mean(axis=0), 0)
    assert np.allclose(scaled.std(axis=0), 1)

# Parametrized test
@pytest.mark.parametrize("input,expected", [
    ([1, 2, 3], 2.0),
    ([10, 20, 30], 20.0),
])
def test_mean(input, expected):
    assert np.mean(input) == expected

# Property-based (Hypothesis)
from hypothesis import given, strategies as st
@given(st.lists(st.floats(), min_size=1))
def test_mean_positive(data):
    assert np.mean(data) >= min(data)

# Fixtures
@pytest.fixture
def sample_data():
    return pd.DataFrame({"x": [1,2,3], "y": [0,1,0]})

def test_model_training(sample_data):
    model = LogisticRegression()
    model.fit(sample_data[["x"]], sample_data["y"])
    assert model.score(sample_data[["x"]], sample_data["y"]) >= 0
```

### 14. **APIs**: "How do you implement authentication/authorization in ML APIs?"

```
AUTHENTICATION (Who are you?):

1. API KEYS (Simple, service-to-service):
   X-API-Key: sk_live_abc123
   - Store hashed in DB
   - Rate limit per key
   - Rotate periodically

2. JWT / BEARER TOKENS (Users, SPAs):
   Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
   - Short expiry (15-30min) + refresh token
   - RS256 (asymmetric) for verification
   - Claims: sub, roles, permissions, exp

3. MUTUAL TLS (Service-to-service):
   - Both client/server present certs
   - SPIFFE/SPIRE for identity
   - No secrets in code

AUTHORIZATION (What can you do?):

1. RBAC (Role-Based):
   @require_permission("model:predict")
   def predict(request): ...

2. ABAC (Attribute-Based):
   policy = """
   allow(principal, action, resource) if 
     principal.role == "admin" or
     (principal.team == resource.owner_team and action == "read")
   """
   # OPA/Gatekeeper evaluates

3. SCOPE-BASED (OAuth2):
   scope: "model:predict model:train"
   token contains scopes, validate per endpoint

IMPLEMENTATION (FastAPI):
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

async def get_current_user(credentials: HTTPAuthorizationCredentials = Depends(security)):
    token = credentials.credentials
    payload = jwt.decode(token, SECRET_KEY, algorithms=["RS256"])
    if payload["exp"] < time.time():
        raise HTTPException(401, "Token expired")
    return User(**payload)

@app.post("/predict")
async def predict(request: PredictRequest, user: User = Depends(get_current_user)):
    # Check permission
    if "model:predict" not in user.permissions:
        raise HTTPException(403, "Insufficient permissions")
    ...
```

### 15. **Architecture**: "How do you design an ML platform?"

```
ML PLATFORM ARCHITECTURE:

COMPONENTS:
┌─────────────────────────────────────────────────────────────┐
│                    ML PLATFORM                              │
├─────────────────────────────────────────────────────────────┤
│  DATA LAYER          │  FEATURE STORE    │  MODEL REGISTRY  │
│  - Raw/Processed     │  - Offline/Online │  - Versions      │
│  - Catalog           │  - Lineage        │  - Staging/Prod  │
│  - Quality           │  - Serving        │  - Lineage       │
├─────────────────────────────────────────────────────────────┤
│  TRAINING            │  DEPLOYMENT       │  MONITORING      │
│  - Pipeline (Kubeflow│  - Batch/Real-time│  - Drift         │
│   / Airflow / Dagster)│  - Canary/Blue-Green│ - Performance │
│  - Hyperopt          │  - A/B Testing    │  - Alerting      │
│  - Experiment Track  │  - Rollback       │  - Audit         │
└─────────────────────────────────────────────────────────────┘

KEY PRINCIPLES:
1. GitOps for everything (infrastructure, models, config)
2. Self-service for data scientists (paved roads)
3. Contract-first (schemas, APIs, SLAs)
4. Observability by default (metrics, logs, traces)
5. Security by default (encryption, authZ, audit)

TECHNOLOGY CHOICES:
- Orchestration: Airflow, Dagster, Prefect, Kubeflow
- Feature Store: Feast, Tecton, Hopsworks
- Model Registry: MLflow, Weights & Biases, Neptune
- Deployment: KServe, Seldon, BentoML, Triton
- Monitoring: Prometheus/Grafana, Evidently, WhyLabs

TEAM TOPOLOGY:
- Platform Team: Builds/maintains platform
- ML Teams: Build models using platform
- Embedded: Platform engineers rotate into ML teams
```

---

## QUICK REFERENCE: PROGRAMMING FUNDAMENTALS CHEAT SHEET

### Python Patterns
| Task | Idiomatic | Avoid |
|------|-----------|-------|
| Filter + transform | `[f(x) for x in items if cond(x)]` | `map(f, filter(cond, items))` |
| Dict from pairs | `{k: v for k, v in pairs}` | `dict(pairs)` |
| Merge dicts | `{**d1, **d2}` (3.5+) | `d1.update(d2)` |
| Default dict | `defaultdict(list)` | `if k not in d: d[k]=[]` |
| Count items | `Counter(items)` | `for x in items: d[x]=d.get(x,0)+1` |
| Group by | `itertools.groupby(sorted, key)` | Manual loop |
| Top-k | `heapq.nlargest(k, iterable)` | `sorted()[-k:]` |

### SQL for ML
| Task | Pattern |
|------|---------|
| Time split | `WHERE date < '2023-01-01'` train, `>=` test |
| Group split | `GROUP BY user_id HAVING RAND() < 0.8` |
| Rolling features | `AVG() OVER (PARTITION BY user ORDER BY date ROWS 6 PRECEDING)` |
| Sessionization | `SUM(is_new) OVER (PARTITION BY user ORDER BY time)` |
| Sampling | `TABLESAMPLE BERNOULLI (10)` |

### API Patterns
| Scenario | Approach |
|----------|----------|
| High throughput | gRPC + connection pooling |
| Public API | REST + OpenAPI spec |
| Streaming | gRPC bidirectional / Server-Sent Events |
| Batch async | Background tasks + webhook callback |
| Model versioning | `/v1/`, `/v2/` or header `Accept: vnd.app.v2+json` |
| Auth | JWT (users) + API Keys (services) + mTLS (internal) |

### Git Workflow for ML
```
main ←── release/v1.2 ←── feature/experiment-42
                 │
                 └── hotfix/critical-bug
```

---

*This document covers all topics from the Programming Fundamentals section of the roadmap.sh Machine Learning roadmap with exam-style depth.*