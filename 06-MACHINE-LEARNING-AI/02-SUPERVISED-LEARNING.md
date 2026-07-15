# SUPERVISED LEARNING
## Exam-Style Study Notes

---

## TOPIC: Supervised Learning Algorithms & Techniques

### WHY? (Problem Statement)

**Supervised Learning = Learning with Labels**
- Input: (X, y) pairs where y is the target
- Goal: Learn f: X → y that generalizes to unseen data
- Two main types: Classification (discrete y) and Regression (continuous y)

**What Supervised Learning Solves:**
| Problem | Business Example | Algorithm Family |
|---------|------------------|------------------|
| Will customer churn? | Retention campaigns | Binary Classification |
| What's the house price? | Real estate pricing | Regression |
| Which product to recommend? | E-commerce upsell | Ranking/Classification |
| Is this transaction fraud? | Financial security | Anomaly/Classification |
| How many items to stock? | Inventory optimization | Regression/Time Series |

**Real-World Analogy:**
- Supervised Learning = Student learning from answer key
- Features = Study materials
- Labels = Correct answers
- Model = Student's understanding
- Test set = Final exam (unseen questions)

---

### HOW? (Learning Mechanisms)

#### 1. **Empirical Risk Minimization (ERM)**

```
LOSS FUNCTION L(y, ŷ):
- Classification: Cross-Entropy, Hinge, Focal
- Regression: MSE, MAE, Huber, Quantile

OPTIMIZATION:
θ* = argmin_θ (1/n) Σ L(y_i, f_θ(x_i)) + λ R(θ)
        │                    │              │
    Empirical Risk      Regularization

ALGORITHM FAMILIES:
─────────────────────────────────────────────────────────────
LINEAR MODELS:
- Logistic/Linear Regression → Closed form or Gradient Descent
- SVM → Convex optimization (QP)

TREE-BASED:
- Decision Trees → Greedy splitting (Information Gain/Gini)
- Random Forest → Bagging + Feature Subsampling
- Gradient Boosting → Sequential residual fitting

NEURAL NETWORKS:
- MLP → Backpropagation + SGD/Adam
- CNN/RNN/Transformer → Specialized architectures
```

#### 2. **Bias-Variance Decomposition for Supervised Learning**

```
EXPECTED TEST ERROR = BIAS² + VARIANCE + IRREDUCIBLE ERROR

MODEL COMPLEXITY →
    │
    │  HIGH BIAS (Underfitting)
    │  ████████████████████████
    │
    │  OPTIMAL
    │        ████████
    │
    │  HIGH VARIANCE (Overfitting)
    │                    ████████████████████████
    └─────────────────────────────────────────────────►
```

#### 3. **Regularization Mechanisms**

| Technique | How It Works | Effect |
|-----------|--------------|--------|
| **L1 (Lasso)** | Add λΣ\|w\| to loss | Sparse weights, feature selection |
| **L2 (Ridge)** | Add λΣw² to loss | Small weights, stability |
| **Elastic Net** | L1 + L2 | Both benefits |
| **Dropout** | Randomly zero neurons | Ensemble effect, prevents co-adaptation |
| **Early Stopping** | Stop when val loss increases | Implicit regularization |
| **Data Augmentation** | Transform inputs | More effective data |
| **Weight Decay** | Multiply weights by (1-λ) each step | Equivalent to L2 in SGD |

---

### WHAT? (Algorithms Deep Dive)

#### 1. **Linear Models**

```python
# LOGISTIC REGRESSION (Binary Classification)
# P(y=1|x) = σ(w,b) = 1 / (1 + exp(-x^T w))
# Loss: -[y log(p) + (1-y) log(1-p)]

from sklearn.linear_model import LogisticRegression
from sklearn.linear_model import SGDClassifier

# Liblinear solver (small datasets, sparse)
lr = LogisticRegression(
    penalty='l2',      # 'l1', 'l2', 'elasticnet', 'none'
    C=1.0,             # Inverse regularization strength
    solver='lbfgs',    # 'liblinear', 'saga', 'newton-cg'
    max_iter=1000,
    class_weight='balanced',  # Handle imbalance
    random_state=42
)

# Linear SVM (Hinge Loss)
# min ½||w||² + C Σ max(0, 1 - y_i(w^T x_i + b))
from sklearn.svm import LinearSVC, SVC
svm = LinearSVC(C=1.0, class_weight='balanced', max_iter=5000)

# REGRESSION
from sklearn.linear_model import LinearRegression, Ridge, Lasso, ElasticNet
ridge = Ridge(alpha=1.0)           # L2
lasso = Lasso(alpha=0.1)           # L1 (sparse)
elastic = ElasticNet(alpha=0.1, l1_ratio=0.5)  # Both
```

#### 2. **Tree-Based Models (Tabular SOTA)**

```python
# DECISION TREE
# Splitting Criteria:
# - Classification: Gini = 1 - Σ p², Entropy = -Σ p log p
# - Regression: MSE, MAE, Friedman MSE

from sklearn.tree import DecisionTreeClassifier, DecisionTreeRegressor
dt = DecisionTreeClassifier(
    criterion='gini',          # 'entropy', 'log_loss'
    max_depth=10,              # Prevent overfitting
    min_samples_split=20,      # Min samples to split
    min_samples_leaf=10,       # Min samples in leaf
    min_impurity_decrease=0.0, # Min gain to split
    class_weight='balanced',
    random_state=42
)

# RANDOM FOREST (Bagging)
# - Bootstrap samples (with replacement)
# - Feature subsampling (sqrt(n_features) for classification)
# - Average predictions

from sklearn.ensemble import RandomForestClassifier, RandomForestRegressor
rf = RandomForestClassifier(
    n_estimators=500,
    max_depth=15,
    min_samples_split=10,
    min_samples_leaf=5,
    max_features='sqrt',       # 'sqrt', 'log2', None, int, float
    bootstrap=True,
    oob_score=True,            # Out-of-bag validation
    class_weight='balanced',
    n_jobs=-1,
    random_state=42
)

# GRADIENT BOOSTING (Sequential, Residual Fitting)
# F_m = F_{m-1} + η * h_m (h_m fits residuals)
# XGBoost / LightGBM / CatBoost / HistGradientBoosting

# XGBOOST (Extreme Gradient Boosting)
import xgboost as xgb
xgb_clf = xgb.XGBClassifier(
    n_estimators=500,
    max_depth=6,
    learning_rate=0.1,
    subsample=0.8,             # Row sampling
    colsample_bytree=0.8,      # Feature sampling
    reg_alpha=0.1,             # L1
    reg_lambda=1.0,            # L2
    min_child_weight=5,
    gamma=0,                   # Min split gain
    scale_pos_weight=1,        # Imbalance handling
    eval_metric='auc',
    early_stopping_rounds=50,
    n_jobs=-1,
    random_state=42
)

# Fit with early stopping
xgb_clf.fit(X_train, y_train,
            eval_set=[(X_val, y_val)],
            verbose=False)

# LIGHTGBM (Leaf-wise growth, faster)
import lightgbm as lgb
lgb_clf = lgb.LGBMClassifier(
    n_estimators=500,
    max_depth=-1,              # No limit
    num_leaves=63,             # Max leaves
    learning_rate=0.1,
    subsample=0.8,
    colsample_bytree=0.8,
    reg_alpha=0.1,
    reg_lambda=1.0,
    min_child_samples=20,
    objective='binary',
    metric='auc',
    verbosity=-1,
    n_jobs=-1,
    random_state=42
)

# CATBOOST (Categorical features native)
from catboost import CatBoostClassifier
cat_clf = CatBoostClassifier(
    iterations=500,
    depth=6,
    learning_rate=0.1,
    l2_leaf_reg=3,
    loss_function='Logloss',
    eval_metric='AUC',
    early_stopping_rounds=50,
    verbose=False,
    random_state=42
)

# HISTOGRAM-BASED GRADIENT BOOSTING (Scikit-learn native, fast)
from sklearn.ensemble import HistGradientBoostingClassifier
hgb = HistGradientBoostingClassifier(
    max_iter=500,
    max_depth=10,
    learning_rate=0.1,
    max_leaf_nodes=31,
    min_samples_leaf=20,
    l2_regularization=1.0,
    early_stopping=True,
    validation_fraction=0.1,
    n_iter_no_change=20,
    random_state=42
)
```

#### 3. **Neural Networks for Tabular Data**

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, TensorDataset

class TabularNN(nn.Module):
    def __init__(self, input_dim, hidden_dims=[256, 128, 64], 
                 dropout=0.3, num_classes=1):
        super().__init__()
        
        layers = []
        prev_dim = input_dim
        
        for hidden_dim in hidden_dims:
            layers.extend([
                nn.Linear(prev_dim, hidden_dim),
                nn.BatchNorm1d(hidden_dim),
                nn.ReLU(),
                nn.Dropout(dropout)
            ])
            prev_dim = hidden_dim
        
        layers.append(nn.Linear(prev_dim, num_classes))
        
        if num_classes == 1:
            layers.append(nn.Sigmoid())
        else:
            layers.append(nn.Softmax(dim=1))
        
        self.network = nn.Sequential(*layers)
    
    def forward(self, x):
        return self.network(x)

# Training Loop
def train_model(model, train_loader, val_loader, epochs=100, lr=1e-3):
    criterion = nn.BCELoss()  # BCEWithLogitsLoss if no sigmoid
    optimizer = optim.AdamW(model.parameters(), lr=lr, weight_decay=1e-4)
    scheduler = optim.lr_scheduler.ReduceLROnPlateau(optimizer, patience=10)
    
    best_val_auc = 0
    patience = 20
    patience_counter = 0
    
    for epoch in range(epochs):
        # Train
        model.train()
        train_loss = 0
        for X_batch, y_batch in train_loader:
            optimizer.zero_grad()
            y_pred = model(X_batch).squeeze()
            loss = criterion(y_pred, y_batch.float())
            loss.backward()
            optimizer.step()
            train_loss += loss.item()
        
        # Validate
        model.eval()
        with torch.no_grad():
            val_preds = []
            val_targets = []
            for X_batch, y_batch in val_loader:
                y_pred = model(X_batch).squeeze()
                val_preds.append(y_pred)
                val_targets.append(y_batch)
            
            val_preds = torch.cat(val_preds).numpy()
            val_targets = torch.cat(val_targets).numpy()
            val_auc = roc_auc_score(val_targets, val_preds)
        
        scheduler.step(val_auc)
        
        if val_auc > best_val_auc:
            best_val_auc = val_auc
            patience_counter = 0
            torch.save(model.state_dict(), 'best_model.pth')
        else:
            patience_counter += 1
            if patience_counter >= patience:
                break
    
    return best_val_auc
```

#### 4. **Support Vector Machines**

```python
from sklearn.svm import SVC, LinearSVC
from sklearn.kernel_approximation import RBFSampler, Nystroem

# LINEAR SVM (Large scale, fast)
linear_svm = LinearSVC(
    C=1.0,
    class_weight='balanced',
    max_iter=10000,
    dual='auto',           # Auto choose dual/primal
    random_state=42
)

# KERNEL SVM (Non-linear, small/medium data)
# RBF Kernel: K(x, x') = exp(-γ ||x - x'||²)
rbf_svm = SVC(
    C=1.0,
    kernel='rbf',
    gamma='scale',         # 'scale', 'auto', float
    class_weight='balanced',
    probability=True,      # Enable predict_proba
    random_state=42
)

# LARGE SCALE KERNEL APPROXIMATION
# Random Fourier Features (RBF kernel approximation)
rbf_feature = RBFSampler(gamma=0.1, n_components=1000, random_state=42)
X_transformed = rbf_feature.fit_transform(X)
linear_svm_on_rbf = LinearSVC(C=1.0, max_iter=5000)
linear_svm_on_rbf.fit(X_transformed, y)

# Nyström Method (Subset of samples)
nystroem = Nystroem(kernel='rbf', gamma=0.1, n_components=500, random_state=42)
X_nystroem = nystroem.fit_transform(X)
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Start with Baseline** | Jump to XGBoost/NN | Can't measure improvement, overkill |
| **Cross-Validation** | Single Train/Test | High variance, unreliable |
| **Feature Engineering** | Raw Data → Model | Misses signal, poor performance |
| **Hyperparameter Tuning** | Default params | Suboptimal, unfair comparison |
| **Class Weights** | Oversampling/Undersampling | Distorts distribution, overfits |
| **Threshold Tuning** | Default 0.5 | Business cost ignored |
| **Calibration** | Raw probabilities | Unreliable confidence |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "Logistic Regression vs Linear SVM"

```
LOGISTIC REGRESSION:
- Probabilistic output (calibrated probabilities)
- Loss: Log Loss (Cross-Entropy)
- Optimization: Gradient Descent (convex)
- Output: P(y=1|x) ∈ [0,1]
- Regularization: L1/L2/ElasticNet
- Interpretable coefficients

LINEAR SVM:
- Geometric margin maximization
- Loss: Hinge Loss max(0, 1 - y(w·x + b))
- No native probabilities (need Platt Scaling)
- Support Vectors only (sparse solution)
- Better for high-dimensional sparse data
- C parameter controls margin violation

WHEN TO USE:
- Need probabilities → Logistic Regression
- High-dimensional sparse (text) → Linear SVM
- Need interpretability → Logistic Regression
- Large scale → Linear SVM (Liblinear)
```

#### 2. **Code**: "XGBoost Early Stopping & Custom Metric"

```python
import xgboost as xgb
from sklearn.metrics import roc_auc_score

def custom_auc(preds, dtrain):
    labels = dtrain.get_label()
    return 'auc', roc_auc_score(labels, preds)

# Custom Objective (Focal Loss for imbalance)
def focal_loss_objective(preds, dtrain):
    labels = dtrain.get_label()
    preds = 1 / (1 + np.exp(-preds))  # Sigmoid
    alpha = 0.25
    gamma = 2.0
    
    # Focal Loss: -α(1-p)^γ y log(p) - (1-α)p^γ (1-y) log(1-p)
    pt = np.where(labels == 1, preds, 1 - preds)
    grad = -alpha * (1 - pt)**gamma * (labels - preds) * gamma * np.log(pt) - alpha * (1 - pt)**gamma * (labels - preds)
    hess = alpha * (1 - pt)**gamma * (gamma * pt * np.log(pt) + pt) + ...
    
    return grad, hess

# Early Stopping with Custom Metric
xgb_clf = xgb.XGBClassifier(
    n_estimators=10000,
    max_depth=6,
    learning_rate=0.05,
    eval_metric=custom_auc,
    early_stopping_rounds=100,
    verbose=50
)

xgb_clf.fit(
    X_train, y_train,
    eval_set=[(X_train, y_train), (X_val, y_val)],
    eval_metric=custom_auc,
    verbose=True
)

# Best iteration
print(f"Best iteration: {xgb_clf.best_iteration}")
print(f"Best score: {xgb_clf.best_score}")
```

#### 3. **Design**: "Handle Imbalanced Classification (1:1000)"

```
STRATEGIES (In Order of Preference):

1. CLASS WEIGHTS (Easiest, No Data Change)
   - LogisticRegression(class_weight='balanced')
   - XGBoost: scale_pos_weight = neg/pos
   - Loss function handles imbalance

2. THRESHOLD TUNING (Post-training)
   - Find optimal threshold on validation
   - Precision-Recall curve → max F1
   - Business cost matrix optimization

3. ENSEMBLE OF BALANCED BAGGING
   - Each tree trained on balanced bootstrap
   - BalancedRandomForestClassifier

4. SMOTE / ADASYN (Oversampling)
   - Generate synthetic minority samples
   - Risk: Overfitting, noise amplification
   - Better: SMOTE + Tomek Links (clean)

5. FOCAL LOSS (Deep Learning)
   - Down-weights easy examples
   - Focuses on hard negatives

6. ANOMALY DETECTION FRAMEWORK
   - One-Class SVM, Isolation Forest
   - Treat minority as anomalies

EVALUATION METRICS FOR IMBALANCE:
- NEVER use Accuracy
- AUC-ROC (threshold independent)
- AUC-PR (better for extreme imbalance)
- F1, Precision@K, Recall@K
- Matthews Correlation Coefficient (MCC)

XGBOOST SPECIFIC:
scale_pos_weight = n_negative / n_positive
# Or per-sample weights
sample_weight = np.where(y == 1, 1000, 1)
```

#### 4. **Code**: "Nested Cross-Validation for Unbiased Estimate"

```python
from sklearn.model_selection import cross_val_score, GridSearchCV, StratifiedKFold
from sklearn.pipeline import Pipeline

# NESTED CV: Outer = Performance Estimate, Inner = Hyperparameter Tuning

outer_cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
inner_cv = StratifiedKFold(n_splits=3, shuffle=True, random_state=42)

# Pipeline with Preprocessing
pipe = Pipeline([
    ('scaler', StandardScaler()),
    ('clf', LogisticRegression(max_iter=1000, random_state=42))
])

# Hyperparameter Grid
param_grid = {
    'clf__C': np.logspace(-3, 3, 7),
    'clf__penalty': ['l1', 'l2'],
    'clf__solver': ['liblinear', 'saga']
}

# Inner CV (Grid Search)
inner_search = GridSearchCV(
    pipe, param_grid=param_grid,
    cv=inner_cv, scoring='roc_auc',
    n_jobs=-1, verbose=0
)

# Outer CV (Performance Estimate)
outer_scores = cross_val_score(
    inner_search, X, y,
    cv=outer_cv, scoring='roc_auc',
    n_jobs=-1
)

print(f"Nested CV AUC: {outer_scores.mean():.4f} (+/- {outer_scores.std()*2:.4f})")

# Results: Unbiased estimate of generalization performance
# inner_search.best_params_ from last fold is not representative!
# Refit on full data with best params from nested CV analysis
```

#### 5. **Code**: "Calibration for Reliable Probabilities"

```python
from sklearn.calibration import CalibratedClassifierCV, calibration_curve
from sklearn.metrics import brier_score_loss

# PLATT SCALING (Logistic Regression on scores)
# ISOTONIC REGRESSION (Non-parametric, more flexible)

# Method 1: Post-hoc Calibration
calibrated = CalibratedClassifierCV(
    base_estimator=xgb_clf,
    method='isotonic',  # 'sigmoid' (Platt) or 'isotonic'
    cv=3,              # Use CV for calibration (avoid overfit)
    ensemble=True      # Average multiple calibrations
)

calibrated.fit(X_train, y_train)

# Method 2: Pre-fit Calibration (if base estimator already trained)
calibrated = CalibratedClassifierCV(
    base_estimator=xgb_clf,
    method='isotonic',
    cv='prefit'  # Use pre-trained estimator
)
calibrated.fit(X_cal, y_cal)  # Separate calibration set!

# Evaluate Calibration
prob_true, prob_pred = calibration_curve(y_test, calibrated.predict_proba(X_test)[:, 1], n_bins=10)

# Plot Reliability Diagram
import matplotlib.pyplot as plt
plt.plot(prob_pred, prob_true, marker='o', label='Calibrated')
plt.plot([0, 1], [0, 1], 'k--', label='Perfect')
plt.xlabel('Mean Predicted Probability')
plt.ylabel('Fraction of Positives')
plt.legend()
plt.show()

# Brier Score (Lower = Better Calibration)
brier = brier_score_loss(y_test, calibrated.predict_proba(X_test)[:, 1])
print(f"Brier Score: {brier:.4f}")

# WHEN TO CALIBRATE:
# - Logistic Regression: Usually well-calibrated
- Random Forest: Often overconfident → Calibrate
- XGBoost/LightGBM: Often overconfident → Calibrate
- SVM: Not calibrated (needs Platt/Isotonic)
- Neural Networks: Often miscalibrated → Calibrate

# NEVER CALIBRATE ON TRAINING DATA!
# Use separate calibration set or CV
```

---

### GOTCHAS & EDGE CASES

- [ ] **Multicollinearity** - Ridge handles, Lasso picks one, trees unaffected
- [ ] **Categorical Variables** - Target encoding leakage, use CV within encoding
- [ ] **Missing Values** - XGBoost/LightGBM/CatBoost handle natively
- [ ] **Outliers** - Tree-based robust, linear models sensitive → Winsorize
- [ ] **Feature Scaling** - Trees/SVM(Kernel) need it, Linear/SVM(Linear)/NN need it
- [ ] **Class Imbalance** - Never use accuracy, use AUC/AUC-PR/F1
- [ ] **Data Leakage** - Target encoding, feature selection outside CV
- [ ] **Reproducibility** - Set random_state everywhere, pin versions
- [ ] **Interpretability** - SHAP values for tree models, coefficients for linear
- [ ] **Production Mismatch** - Train/Serve skew, feature store needed

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete Model Training Template**
```python
def train_evaluate_model(X, y, model, params, cv=5):
    """Train with CV, return best model and scores"""
    
    # Cross-Validation
    cv_scores = cross_val_score(
        model, X, y, cv=StratifiedKFold(cv, shuffle=True, random_state=42),
        scoring='roc_auc', n_jobs=-1
    )
    
    # Hyperparameter Tuning
    search = GridSearchCV(model, params, cv=3, scoring='roc_auc', n_jobs=-1)
    search.fit(X, y)
    
    # Best Model
    best_model = search.best_estimator_
    
    # Feature Importance (if available)
    if hasattr(best_model, 'feature_importances_'):
        importance = pd.Series(
            best_model.feature_importances_, 
            index=X.columns
        ).sort_values(ascending=False)
    
    return {
        'model': best_model,
        'cv_scores': cv_scores,
        'best_params': search.best_params_,
        'best_score': search.best_score_,
        'importance': importance if 'importance' in locals() else None
    }
```

#### 2. **SHAP Explanation for Tree Models**
```python
import shap

# TreeExplainer (fast for tree models)
explainer = shap.TreeExplainer(xgb_clf)
shap_values = explainer.shap_values(X_test)

# Summary Plot (Feature Importance + Direction)
shap.summary_plot(shap_values, X_test, plot_type='bar')
shap.summary_plot(shap_values, X_test)

# Dependence Plot (Feature Interaction)
shap.dependence_plot('feature_name', shap_values, X_test)

# Force Plot (Individual Prediction)
shap.force_plot(explainer.expected_value, shap_values[0], X_test.iloc[0])

# SHAP for Linear Models
explainer = shap.LinearExplainer(lr_model, X_train)
```

#### 3. **Model Comparison Framework**
```python
from sklearn.model_selection import cross_validate

models = {
    'LogisticRegression': LogisticRegression(max_iter=1000, class_weight='balanced'),
    'RandomForest': RandomForestClassifier(n_estimators=200, class_weight='balanced'),
    'XGBoost': xgb.XGBClassifier(n_estimators=200, scale_pos_weight=100),
    'LightGBM': lgb.LGBMClassifier(n_estimators=200, class_weight='balanced'),
    'HistGradientBoosting': HistGradientBoostingClassifier(class_weight='balanced'),
}

scoring = ['roc_auc', 'average_precision', 'f1', 'precision', 'recall']

results = {}
for name, model in models.items():
    cv_results = cross_validate(
        model, X, y, 
        cv=StratifiedKFold(5, shuffle=True, random_state=42),
        scoring=scoring, n_jobs=-1, return_train_score=True
    )
    results[name] = {
        'test_auc_mean': cv_results['test_roc_auc'].mean(),
        'test_auc_std': cv_results['test_roc_auc'].std(),
        'test_ap_mean': cv_results['test_average_precision'].mean(),
        'fit_time': cv_results['fit_time'].mean(),
    }

# Compare
df_results = pd.DataFrame(results).T
print(df_results.sort_values('test_auc_mean', ascending=False))
```

---

### PRACTICE PROBLEMS

1. **Build** end-to-end churn prediction: EDA → Features → Baseline → Tuning → Calibration → Explainability
2. **Debug** model with 0.99 train AUC, 0.55 test AUC
3. **Implement** custom XGBoost objective for quantile regression
4. **Design** A/B test for new ranking model (power analysis, metrics, guardrails)
5. **Compare** Random Forest vs XGBoost vs Neural Net on same dataset

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| Logistic Regression Loss | Log Loss (Cross-Entropy): -[y log p + (1-y) log(1-p)] |
| SVM Loss | Hinge Loss: max(0, 1 - y(w·x + b)) |
| XGBoost Key Params | n_estimators, max_depth, learning_rate, subsample, colsample_bytree, reg_alpha, reg_lambda |
| LightGBM vs XGBoost | LightGBM: Leaf-wise (faster), handles categorical. XGBoost: Level-wise, more stable |
| CatBoost Advantage | Native categorical handling, ordered boosting (no target leak) |
| HistGradientBoosting | Scikit-learn native, histogram-based, fast, handles missing |
| scale_pos_weight | n_negative / n_positive for binary imbalance |
| Calibration Methods | Platt (sigmoid) - parametric. Isotonic - non-parametric, more flexible |
| Nested CV | Outer: Performance estimate. Inner: Hyperparameter tuning. Unbiased. |
| Class Weight Balanced | n_samples / (n_classes * n_samples_per_class) |
| Random Forest OOB Score | Unbiased validation using out-of-bag samples |
| Gradient Boosting | Sequential: each tree fits residuals of previous ensemble |
| Tree Splitting Criteria | Gini = 1-Σp². Entropy = -Σp log p. MSE for regression. |
| Early Stopping | Monitor validation metric, stop when no improvement for N rounds |
| SHAP Values | Shapley values from game theory. Consistent, locally accurate. |

---

## NEXT TOPIC: `03-UNSUPERVISED-LEARNING.md`

> **Study Tip**: Pick a Kaggle tabular competition. Build 5 models (LogReg, RF, XGB, LGBM, CatBoost). Compare with nested CV. Calibrate best. Explain with SHAP. Write 1-page summary.