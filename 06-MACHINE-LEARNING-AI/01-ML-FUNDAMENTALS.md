# MACHINE LEARNING & AI FUNDAMENTALS
## Exam-Style Study Notes

---

## TOPIC: Machine Learning Engineering for Interviews

### WHY? (Problem Statement)

**Why ML in Interviews:**
- Every tech company needs ML engineers
- ML system design questions are standard
- Need to bridge theory → production
- Understanding tradeoffs, not just algorithms

**What ML Fundamentals Solve:**
| Problem | Solution |
|---------|----------|
| "When to use ML?" | Problem framing, ROI analysis |
| "Which algorithm?" | Problem type matching, constraints |
| "How to productionize?" | MLOps, monitoring, drift detection |
| "How to debug?" | Error analysis, bias/variance tradeoff |
| "How to scale?" | Distributed training, serving |

**Real-World Analogy:**
- ML Model = Recipe (ingredients → dish)
- Training = Test kitchen (experiment, iterate)
- Production = Restaurant kitchen (reliable, fast, consistent)
- MLOps = Restaurant operations (supply chain, quality control)

---

### HOW? (Core ML Pipeline)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ML PROJECT LIFECYCLE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. PROBLEM FRAMING          2. DATA ENGINEERING       3. MODEL DEVELOPMENT │
│  ─────────────────           ──────────────────        ───────────────────  │
│  • Business → ML problem     • Collection               • Architecture      │
│  • Success metrics           • Cleaning                 • Training          │
│  • Constraints (latency,     • Validation                • Tuning           │
│    privacy, cost)            • Feature engineering       • Validation       │
│  • Baseline                  • Versioning                • Error analysis   │
│                                                                             │
│  4. DEPLOYMENT               5. MONITORING              6. ITERATION       │
│  ──────────────              ─────────────              ────────────       │
│  • Serving (batch/real-time) • Data drift                • Retraining      │
│  • A/B testing               • Concept drift             • Feature adds    │
│  • Canary release            • Performance               • Architecture    │
│  • Rollback                  • Business metrics          • Model swap      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Core Concepts & Algorithms)

#### 1. **Problem Types & Matching**

| Problem Type | Output | Algorithms | Metrics |
|--------------|--------|------------|---------|
| **Binary Classification** | 0/1 | Logistic Reg, XGBoost, NN | AUC-ROC, F1, Precision/Recall |
| **Multi-class Classification** | Class label | Softmax, Random Forest | Accuracy, Macro F1, Confusion Matrix |
| **Regression** | Continuous | Linear Reg, GBRT, NN | MSE, MAE, R² |
| **Ranking** | Ordered list | LambdaMART, ListNet | NDCG, MAP, MRR |
| **Recommendation** | Items | CF, Matrix Factorization, Two-Tower | Recall@K, Coverage |
| **Clustering** | Groups | K-Means, DBSCAN, HDBSCAN | Silhouette, ARI |
| **Anomaly Detection** | Score/Flag | Isolation Forest, Autoencoder | Precision@Recall, F1 |
| **Time Series** | Future values | ARIMA, Prophet, LSTM, TFT | MAE, SMAPE |
| **NLP** | Text/Embeddings | BERT, GPT, Transformers | Perplexity, BLEU, F1 |
| **Computer Vision** | Classes/Masks | CNN, ViT, YOLO, SAM | mAP, IoU |

#### 2. **Key Algorithms - When to Use What**

```
LINEAR MODELS (Interpretable, Fast):
├── Linear Regression          → Baseline regression
├── Logistic Regression        → Baseline classification
├── Ridge/Lasso/ElasticNet     → Regularization, feature selection
└── SGD Classifier/Regressor   → Large-scale, online learning

TREE-BASED (Tabular Data King):
├── Decision Tree              → Interpretable, non-linear
├── Random Forest              → Robust, handles mixed types
├── Gradient Boosting (XGBoost, LightGBM, CatBoost) → SOTA tabular
└── Histogram Gradient Boosting → Fast, native missing values

NEURAL NETWORKS (Complex Patterns):
├── MLP                        → Tabular with interactions
├── CNN                        → Images, sequences (1D)
├── RNN/LSTM/GRU               → Sequential (deprecated for transformers)
├── Transformer                → NLP, Vision, Multimodal
├── Attention Mechanism        → Long-range dependencies
└── Embedding Layers           → Categorical, text, graphs

UNSUPERVISED:
├── PCA / SVD                  → Dimensionality reduction
├── K-Means / MiniBatch        → Large-scale clustering
├── DBSCAN / HDBSCAN           → Arbitrary shapes, noise
├── Isolation Forest           → Anomaly detection
└── Autoencoders               → Representation learning, anomaly
```

#### 3. **Model Evaluation - Beyond Accuracy**

```python
# Classification Metrics
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score, f1_score,
    roc_auc_score, average_precision_score,
    confusion_matrix, classification_report,
    precision_recall_curve, roc_curve
)

# Regression Metrics
from sklearn.metrics import (
    mean_squared_error, mean_absolute_error,
    r2_score, mean_absolute_percentage_error
)

# Ranking Metrics
from sklearn.metrics import ndcg_score, ndcg_score

# KEY CONCEPTS:
# ─────────────────────────────────────────────────────────────
# PRECISION = TP / (TP + FP)  → Of predicted positive, how many correct?
# RECALL = TP / (TP + FN)     → Of actual positive, how many found?
# F1 = 2 * P * R / (P + R)    → Harmonic mean
# AUC-ROC = Area under ROC curve (threshold independent)
# AUC-PR = Area under Precision-Recall (better for imbalanced)
# CONFUSION MATRIX → TN, FP, FN, TP breakdown

# WHEN TO USE WHICH:
# ─────────────────────────────────────────────────────────────
# Balanced, cost equal          → Accuracy
# Imbalanced, FP costly         → Precision
# Imbalanced, FN costly         → Recall
# Need balance                  → F1
# Threshold tuning needed       → AUC-ROC, AUC-PR
# Ranking quality               → NDCG, MAP
# Calibration needed            → Brier score, reliability diagram
```

#### 4. **Validation Strategies**

```python
from sklearn.model_selection import (
    train_test_split, KFold, StratifiedKFold,
    TimeSeriesSplit, GroupKFold, LeaveOneGroupOut
)

# STRATEGIES:
# ─────────────────────────────────────────────────────────────
# Random Split              → IID data, baseline
# Stratified K-Fold         → Classification, maintain class ratio
# Time Series Split         → Temporal data, NO random shuffle!
# Group K-Fold              → Grouped data (user, session) - no leakage!
# Nested CV                 → Hyperparameter tuning + unbiased estimate

# DATA LEAKAGE PREVENTION:
# ─────────────────────────────────────────────────────────────
# ❌ Fit scaler on full dataset → Transform train/test
# ✅ Fit scaler on TRAIN only → Transform train, test

# ❌ Feature selection on full data
# ✅ Feature selection inside CV loop

# ❌ Target encoding using full data
# ✅ Target encoding with smoothing, fit on train only

# ❌ Using future data for past prediction (time series)
# ✅ Strict temporal splits, expanding window
```

#### 5. **Hyperparameter Optimization**

```python
# Libraries: Optuna, Hyperopt, Ray Tune, Scikit-Optimize

import optuna

def objective(trial):
    # Define search space
    params = {
        'n_estimators': trial.suggest_int('n_estimators', 100, 1000),
        'max_depth': trial.suggest_int('max_depth', 3, 15),
        'learning_rate': trial.suggest_float('learning_rate', 0.01, 0.3, log=True),
        'subsample': trial.suggest_float('subsample', 0.5, 1.0),
        'colsample_bytree': trial.suggest_float('colsample_bytree', 0.5, 1.0),
        'reg_alpha': trial.suggest_float('reg_alpha', 1e-8, 10.0, log=True),
        'reg_lambda': trial.suggest_float('reg_lambda', 1e-8, 10.0, log=True),
        'min_child_weight': trial.suggest_int('min_child_weight', 1, 10),
    }
    
    model = XGBClassifier(**params, random_state=42, n_jobs=-1)
    scores = cross_val_score(model, X, y, cv=StratifiedKFold(5), scoring='roc_auc')
    return scores.mean()

study = optuna.create_study(direction='maximize', sampler=optuna.samplers.TPESampler())
study.optimize(objective, n_trials=100, timeout=3600)

# Best params
print(study.best_params)
print(study.best_value)

# Pruning (early stopping unpromising trials)
study.optimize(objective, n_trials=100, 
               callbacks=[optuna.study.MaxTrialsCallback(100)])
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Start Simple** | Deep learning first | Overkill, uninterpretable, data hungry |
| **Baseline First** | No baseline | Can't measure improvement |
| **Cross-Validation** | Single train/test split | High variance, unreliable |
| **Feature Engineering** | Raw data → Model | Misses signal, poor performance |
| **Error Analysis** | Only look at metrics | Don't know WHY it fails |
| **Monitoring** | Deploy and forget | Silent degradation |
| **Version Everything** | Manual tracking | Can't reproduce, debug |
| **A/B Test** | Ship to all | Risk, no causal evidence |

---

### INTERVIEW QUESTIONS (Top 20)

#### 1. **Conceptual**: "Bias-Variance Tradeoff"

```
ERROR = BIAS² + VARIANCE + IRREDUCIBLE ERROR

HIGH BIAS (Underfitting):
- Model too simple
- Train error HIGH, Test error HIGH
- Fix: More features, complex model, less regularization

HIGH VARIANCE (Overfitting):
- Model too complex
- Train error LOW, Test error HIGH
- Fix: More data, simpler model, regularization, dropout

IRREDUCIBLE ERROR:
- Noise in data, label errors
- Theoretical minimum

VISUAL:
┌─────────────────────────────────────────────────────────────┐
│                    TOTAL ERROR                              │
│     ┌─────────────────────┐                                 │
│     │      BIAS²          │                                 │
│     │        ▼            │                                 │
│     │    ┌─────────┐      │                                 │
│     │    │ VARIANCE  │      │                                 │
│     │    └─────────┘      │                                 │
│     └─────────────────────┘                                 │
│           MODEL COMPLEXITY →                                │
└─────────────────────────────────────────────────────────────┘

REGULARIZATION KNOBS:
- L1 (Lasso): Feature selection, sparse weights
- L2 (Ridge): Weight decay, small weights
- Elastic Net: L1 + L2
- Dropout: Random neuron deactivation (NN)
- Early Stopping: Stop before overfit
- Data Augmentation: More effective data
- Ensemble: Average predictions (RF, Boosting)
```

#### 2. **Code**: "Implement Gradient Descent from Scratch"

```python
import numpy as np

class LinearRegressionGD:
    def __init__(self, learning_rate=0.01, n_iterations=1000, lambda_l2=0.0):
        self.lr = learning_rate
        self.n_iter = n_iterations
        self.lambda_l2 = lambda_l2
        self.weights = None
        self.bias = None
        self.loss_history = []
    
    def fit(self, X, y):
        n_samples, n_features = X.shape
        self.weights = np.zeros(n_features)
        self.bias = 0
        
        for i in range(self.n_iter):
            # Forward pass
            y_pred = np.dot(X, self.weights) + self.bias
            
            # Compute gradients (MSE + L2)
            dw = (2/n_samples) * np.dot(X.T, (y_pred - y)) + (2 * self.lambda_l2 / n_samples) * self.weights
            db = (2/n_samples) * np.sum(y_pred - y)
            
            # Update
            self.weights -= self.lr * dw
            self.bias -= self.lr * db
            
            # Track loss
            loss = np.mean((y_pred - y) ** 2) + self.lambda_l2 * np.sum(self.weights ** 2)
            self.loss_history.append(loss)
            
            # Early stopping check
            if i > 10 and abs(self.loss_history[-1] - self.loss_history[-2]) < 1e-6:
                break
    
    def predict(self, X):
        return np.dot(X, self.weights) + self.bias

# Mini-batch SGD
def sgd_minibatch(X, y, batch_size=32, epochs=100, lr=0.01):
    n = len(X)
    w = np.zeros(X.shape[1])
    b = 0
    
    for epoch in range(epochs):
        indices = np.random.permutation(n)
        for i in range(0, n, batch_size):
            batch_idx = indices[i:i+batch_size]
            X_batch, y_batch = X[batch_idx], y[batch_idx]
            
            y_pred = X_batch @ w + b
            dw = (2/len(X_batch)) * X_batch.T @ (y_pred - y_batch)
            db = (2/len(X_batch)) * np.sum(y_pred - y_batch)
            
            w -= lr * dw
            b -= lr * db
    
    return w, b

# Adam Optimizer (Adaptive Moment Estimation)
class Adam:
    def __init__(self, lr=0.001, beta1=0.9, beta2=0.999, eps=1e-8):
        self.lr = lr
        self.beta1 = beta1
        self.beta2 = beta2
        self.eps = eps
        self.m = 0
        self.v = 0
        self.t = 0
    
    def update(self, params, grads):
        self.t += 1
        self.m = self.beta1 * self.m + (1 - self.beta1) * grads
        self.v = self.beta2 * self.v + (1 - self.beta2) * (grads ** 2)
        
        m_hat = self.m / (1 - self.beta1 ** self.t)
        v_hat = self.v / (1 - self.beta2 ** self.t)
        
        params -= self.lr * m_hat / (np.sqrt(v_hat) + self.eps)
        return params
```

#### 3. **Design**: "Build a Recommendation System"

```
REQUIREMENTS:
- 10M users, 1M items, 1B interactions
- Real-time recommendations (<50ms)
- Cold start for new users/items
- Explainability

ARCHITECTURE:
─────────────────────────────────────────────────────────────
┌─────────────────────────────────────────────────────────────┐
│                    TWO-TOWER MODEL                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   USER TOWER                    ITEM TOWER                  │
│   ────────────                  ───────────                 │
│   User ID  ──┐                  Item ID  ──┐               │
│   Features ──┤ Embedding → Dense → Dot ───┤ Embedding       │
│   History  ──┤ (128-d)  → Layers → Score  │ (128-d)         │
│              │                          └───────────┘       │
│              └──────────────────────────────┘               │
│                         │                                    │
│                    Similarity Score                          │
│                         │                                    │
│                    Top-K Retrieval                           │
│                         │                                    │
│                    Reranking (LightGBM)                      │
│                         │                                    │
│                    Final Top-N                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘

TRAINING:
─────────────────────────────────────────────────────────────
1. POSITIVE SAMPLES: (user, item) from interactions
2. NEGATIVE SAMPLING: Random items + hard negatives
3. LOSS: Binary Cross-Entropy / BPR / Sampled Softmax
4. FEATURES:
   - User: demographics, embeddings from history, context
   - Item: category, textual embeddings (BERT, imageCNN, popularity

SERVING:
─────────────────────────────────────────────────────────────
1. CANDIDATE GENERATION (Recall):
   - Annoy / FAISS / ScaNN for ANN search
   - User embedding → Top 1000 items
   - ~10ms

2. RERANKING (Precision):
   - LightGBM / DNN with cross features
   - Score top 100 → Top 10
   - ~30ms

3. BUSINESS LOGIC:
   - Diversity, freshness, monetization
   - Filter ineligible items

COLD START:
─────────────────────────────────────────────────────────────
NEW USER:
- Demographics → User tower (no history)
- Popular items + category preferences
- Onboarding quiz → Explicit preferences

NEW ITEM:
- Content features (text, image, metadata) → Item tower
- Explore: Thompson Sampling / UCB
- Impression-weighted sampling

SCALING:
─────────────────────────────────────────────────────────────
- User embeddings: Redis (10M × 128 = 1.6GB)
- Item embeddings: FAISS Index (1M × 128)
- Two-tower: TensorRT / ONNX Runtime
- Batch inference for offline scores
```

#### 4. **Code**: "Feature Engineering Pipeline"

```python
import pandas as pd
import numpy as np
from sklearn.base import BaseEstimator, TransformerMixin
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.compose import ColumnTransformer
from category_encoders import TargetEncoder
import hashlib

class FeatureEngineer(BaseEstimator, TransformerMixin):
    """Automated feature engineering with leakage prevention"""
    
    def __init__(self, target_col='target', datetime_cols=None, 
                 text_cols=None, high_cardinality_threshold=50):
        self.target_col = target_col
        self.datetime_cols = datetime_cols or []
        self.text_cols = text_cols or []
        self.high_cardinality_threshold = high_cardinality_threshold
        self.fitted_encoders = {}
        self.target_means = {}
    
    def fit(self, X, y=None):
        df = X.copy()
        if y is not None:
            df[self.target_col] = y
        
        # 1. Datetime features
        for col in self.datetime_cols:
            if col in df.columns:
                df[col] = pd.to_datetime(df[col])
                df[f'{col}_hour'] = df[col].dt.hour
                df[f'{col}_dayofweek'] = df[col].dt.dayofweek
                df[f'{col}_month'] = df[col].dt.month
                df[f'{col}_is_weekend'] = df[col].dt.dayofweek >= 5
        
        # 2. Target Encoding for High Cardinality Categoricals
        cat_cols = df.select_dtypes(include=['object', 'category']).columns
        for col in cat_cols:
            if col in self.text_cols: continue
            if col == self.target_col: continue
            
            nunique = df[col].nunique()
            if nunique > self.high_cardinality_threshold:
                # Smooth target encoding
                encoder = TargetEncoder(smoothing=10, min_samples_leaf=20)
                encoder.fit(df[col], df[self.target_col])
                self.fitted_encoders[col] = encoder
                self.target_means[col] = df[self.target_col].mean()
            else:
                # One-hot for low cardinality
                encoder = OneHotEncoder(handle_unknown='ignore', sparse_output=False)
                encoder.fit(df[[col]])
                self.fitted_encoders[col] = encoder
        
        # 3. Numerical Scaling
        num_cols = df.select_dtypes(include=[np.number]).columns
        num_cols = [c for c in num_cols if c != self.target_col]
        self.scaler = StandardScaler()
        self.scaler.fit(df[num_cols])
        
        return self
    
    def transform(self, X):
        df = X.copy()
        
        # Datetime
        for col in self.datetime_cols:
            if col in df.columns:
                df[col] = pd.to_datetime(df[col])
                df[f'{col}_hour'] = df[col].dt.hour
                df[f'{col}_dayofweek'] = df[col].dt.dayofweek
                df[f'{col}_month'] = df[col].dt.month
                df[f'{col}_is_weekend'] = df[col].dt.dayofweek >= 5
        
        # Categorical
        for col, encoder in self.fitted_encoders.items():
            if col not in df.columns: continue
            
            if isinstance(encoder, TargetEncoder):
                encoded = encoder.transform(df[col])
                # Handle unseen: use global mean
                encoded = encoded.fillna(self.target_means.get(col, 0))
                df[f'{col}_te'] = encoded
                df.drop(columns=[col], inplace=True)
            elif isinstance(encoder, OneHotEncoder):
                encoded = encoder.transform(df[[col]])
                feature_names = encoder.get_feature_names_out([col])
                encoded_df = pd.DataFrame(encoded, columns=feature_names, index=df.index)
                df = pd.concat([df.drop(columns=[col]), encoded_df], axis=1)
        
        # Numerical Scaling
        num_cols = df.select_dtypes(include=[np.number]).columns
        df[num_cols] = self.scaler.transform(df[num_cols])
        
        # Hash encoding for unseen categorical (alternative)
        for col in self.text_cols:
            if col in df.columns:
                df[f'{col}_hash'] = df[col].apply(
                    lambda x: int(hashlib.md5(str(x).encode()).hexdigest(), 16) % 1000
                )
        
        return df

# Interaction Features
class InteractionFeatures(BaseEstimator, TransformerMixin):
    def __init__(self, feature_pairs=None, operations=None):
        self.feature_pairs = feature_pairs or []
        self.operations = operations or ['multiply', 'divide', 'add', 'subtract']
    
    def fit(self, X, y=None): return self
    
    def transform(self, X):
        df = X.copy()
        for f1, f2 in self.feature_pairs:
            if f1 in df.columns and f2 in df.columns:
                if 'multiply' in self.operations:
                    df[f'{f1}_x_{f2}'] = df[f1] * df[f2]
                if 'divide' in self.operations:
                    df[f'{f1}_div_{f2}'] = df[f1] / (df[f2] + 1e-8)
                if 'add' in self.operations:
                    df[f'{f1}_plus_{f2}'] = df[f1] + df[f2]
                if 'subtract' in self.operations:
                    df[f'{f1}_minus_{f2}'] = df[f1] - df[f2]
        return df

# Complete Pipeline
def build_feature_pipeline(X, y, cat_cols, num_cols, datetime_cols):
    preprocessor = ColumnTransformer([
        ('cat', Pipeline([
            ('target_enc', TargetEncoder(smoothing=10)),
        ]), cat_cols),
        ('num', StandardScaler(), num_cols),
        ('dt', DateTimeFeatures(), datetime_cols),
    ])
    
    pipeline = Pipeline([
        ('features', FeatureEngineer(target_col='target', datetime_cols=datetime_cols)),
        ('interactions', InteractionFeatures([
            ('age', 'income'), ('price', 'quantity')
        ])),
        ('preprocess', preprocessor),
        ('select', SelectKBest(f_classif, k=50)),
    ])
    
    return pipeline.fit(X, y)
```

---

### GOTCHAS & EDGE CASES

- [ ] **Data Leakage** - Fit transformers on train only, never test
- [ ] **Class Imbalance** - Use class_weight, sampling, threshold tuning
- [ ] **Target Leakage** - Don't use future data to predict past
- [ ] **Categorical Encoding** - High cardinality → target/hash encoding
- [ ] **Numerical Stability** - Log transform skewed features, clip outliers
- [ ] **Reproducibility** - Set seeds, pin versions, log everything
- [ ] **Concept Drift** - Monitor feature distributions, retrain regularly
- [ ] **Feedback Loops** - Model affects data → biased retraining
- [ ] **Cold Start** - Content-based fallback, exploration strategies
- [ ] **Evaluation ≠ Business Metric** - Optimize proxy, measure business

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete Training Loop with MLflow**
```python
import mlflow
import mlflow.sklearn
from sklearn.model_selection import cross_val_score

mlflow.set_experiment("customer-churn")

with mlflow.start_run(run_name="xgboost-baseline"):
    # Log params
    params = {'n_estimators': 500, 'max_depth': 6, 'learning_rate': 0.1}
    mlflow.log_params(params)
    
    # Train with CV
    model = XGBClassifier(**params, random_state=42, n_jobs=-1)
    cv_scores = cross_val_score(model, X_train, y_train, 
                                cv=StratifiedKFold(5), scoring='roc_auc')
    
    # Log metrics
    mlflow.log_metric("cv_auc_mean", cv_scores.mean())
    mlflow.log_metric("cv_auc_std", cv_scores.std())
    
    # Fit full model
    model.fit(X_train, y_train)
    
    # Evaluate
    y_pred_proba = model.predict_proba(X_test)[:, 1]
    test_auc = roc_auc_score(y_test, y_pred_proba)
    mlflow.log_metric("test_auc", test_auc)
    
    # Log model
    mlflow.sklearn.log_model(model, "model", 
                             registered_model_name="churn-predictor")
    
    # Log artifacts
    mlflow.log_artifact("feature_importance.png")
```

#### 2. **Model Monitoring Dashboard**
```python
# Drift Detection
from scipy.stats import ks_2samp, chi2_contingency
import numpy as np

def detect_data_drift(reference, current, threshold=0.05):
    """Detect distribution shift using KS test (numerical) / Chi-square (categorical)"""
    drift_results = {}
    
    for col in reference.columns:
        if reference[col].dtype in ['int64', 'float64']:
            # KS Test for numerical
            stat, p_value = ks_2samp(reference[col].dropna(), current[col].dropna())
            drift_results[col] = {'test': 'KS', 'statistic': stat, 'p_value': p_value, 
                                  'drift': p_value < threshold}
        else:
            # Chi-square for categorical
            ref_counts = reference[col].value_counts()
            cur_counts = current[col].value_counts()
            # Align categories
            all_cats = set(ref_counts.index) | set(cur_counts.index)
            ref_aligned = [ref_counts.get(c, 0) for c in all_cats]
            cur_aligned = [cur_counts.get(c, 0) for c in all_cats]
            
            if sum(ref_aligned) > 0 and sum(cur_aligned) > 0:
                stat, p_value, _, _ = chi2_contingency([ref_aligned, cur_aligned])
                drift_results[col] = {'test': 'Chi2', 'statistic': stat, 'p_value': p_value,
                                      'drift': p_value < threshold}
    
    return pd.DataFrame(drift_results).T

# Feature Importance Stability
def check_importance_stability(model, X_ref, X_cur, top_k=20):
    ref_imp = pd.Series(model.feature_importances_, index=X_ref.columns).nlargest(top_k)
    # Need retrained model on current data for comparison
    # Track rank correlation over time
```

---

### PRACTICE PROBLEMS

1. **Implement** end-to-end ML pipeline: Data → Features → Train → Evaluate → Deploy → Monitor
2. **Debug** a model with 99% training accuracy, 60% test accuracy
3. **Design** A/B test for new ranking model (sample size, metrics, duration)
4. **Build** feature store with Feast or custom (offline + online stores)
5. **Implement** distributed training with PyTorch DDP / Horovod

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| Bias-Variance Tradeoff | Bias = underfitting (high train/test error). Variance = overfitting (low train, high test). |
| When to Use XGBoost | Tabular data, structured, SOTA baseline. Not for images/text/raw signals. |
| Cross-Validation Types | Random, Stratified (class balance), TimeSeries (temporal), Group (no leakage). |
| Data Leakage Prevention | Fit transformers on TRAIN only. Feature selection inside CV. |
| Precision vs Recall | Precision: Of predicted +, how many true? Recall: Of actual +, how many found? |
| AUC-ROC vs AUC-PR | ROC: Balanced classes. PR: Imbalanced, focuses on positive class. |
| Regularization L1 vs L2 | L1: Sparse (feature selection). L2: Small weights. Elastic Net: Both. |
| Embedding Dimension Rule | 4th root of cardinality, or min(50, cardinality/2). |
| Two-Tower Retrieval | User tower + Item tower → Dot product → ANN search → Rerank. |
| Concept Drift | P(Y|X) changes. Data Drift: P(X) changes. Monitor both. |

---

## NEXT TOPIC: `08-INTERVIEW-PREP/01-BEHAVIORAL.md`

> **Study Tip**: Build a mini ML project end-to-end: Kaggle competition → Feature engineering → Optuna tuning → MLflow tracking → FastAPI serving → Docker → Monitoring dashboard.