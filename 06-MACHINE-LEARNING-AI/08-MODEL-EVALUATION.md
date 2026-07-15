# MODEL EVALUATION & VALIDATION
## Exam-Style Study Notes

**Roadmap Section:** Model Evaluation
**Prerequisites:** Statistics, Python, Scikit-learn, basic ML concepts

---

## 1. EVALUATION METRICS

### WHY? (Problem Statement)
**You can't improve what you can't measure.** Choosing the right metric aligns model optimization with business objectives.

### HOW? (Classification Metrics)

#### Threshold-Independent Metrics
| Metric | Formula | Range | Best For |
|--------|---------|-------|----------|
| **AUC-ROC** | ∫ TPR dFPR | [0,1] | General ranking quality |
| **AUC-PR** | ∫ Precision dRecall | [0,1] | Imbalanced data |
| **Log Loss** | -Σ y log(p) | [0,∞) | Probability calibration |
| **Brier Score** | Σ (y - p)² / n | [0,1] | Calibration + discrimination |

```python
from sklearn.metrics import roc_auc_score, average_precision_score, log_loss, brier_score_loss

auc_roc = roc_auc_score(y_true, y_prob)
auc_pr = average_precision_score(y_true, y_prob)
logloss = log_loss(y_true, y_prob)
brier = brier_score_loss(y_true, y_prob)
```

#### Threshold-Dependent Metrics
| Metric | Formula | When to Use |
|--------|---------|-------------|
| **Accuracy** | (TP+TN)/(TP+TN+FP+FN) | Balanced classes, equal cost |
| **Precision** | TP/(TP+FP) | FP costly (spam, fraud alert) |
| **Recall** | TP/(TP+FN) | FN costly (cancer, safety) |
| **F1** | 2PR/(P+R) | Balance P/R, imbalanced |
| **Fβ** | (1+β²)PR/(β²P+R) | β>1: recall focus, β<1: precision focus |
| **MCC** | (TP×TN-FP×FN)/√(...) | [-1,1], all classes matter |

```python
from sklearn.metrics import (accuracy_score, precision_score, recall_score, 
                             f1_score, fbeta_score, matthews_corrcoef, 
                             confusion_matrix, classification_report)

y_pred = (y_prob > threshold).astype(int)
print(classification_report(y_true, y_pred))

# Optimal threshold selection
from sklearn.metrics import precision_recall_curve
prec, rec, thresholds = precision_recall_curve(y_true, y_prob)
f1_scores = 2 * prec * rec / (prec + rec + 1e-10)
best_idx = np.argmax(f1_scores)
best_threshold = thresholds[best_idx]
```

#### Multi-class_idx]
```

#### Regression Metrics
| Metric | Formula | Robust? | When |
|--------|---------|---------|------|
| **MSE** | Σ(y-ŷ)²/n | No | General, differentiable |
| **RMSE** | √MSE | No | Same units as target |
| **MAE** | Σ|y-ŷ|/n | Yes | Outliers present |
| **Huber** | Quadratic near 0, linear far | Yes | Best of both |
| **Quantile** | Asymmetric loss | Yes | Prediction intervals |
| **R²** | 1 - SS_res/SS_tot | No | Explained variance |
| **MAPE** | Σ|y-ŷ|/|y|/n | No | Percentage errors |
| **SMAPE** | 2|y-ŷ|/(|y|+|ŷ|)/n | Partial | Scale-independent |

```python
from sklearn.metrics import (mean_squared_error, mean_absolute_error, 
                             r2_score, mean_absolute_percentage_error)

mse = mean_squared_error(y_true, y_pred)
rmse = np.sqrt(mse)
mae = mean_absolute_error(y_true, y_pred)
r2 = r2_score(y_true, y_pred)
mape = mean_absolute_percentage_error(y_true, y_pred)

# Huber loss
from sklearn.metrics import mean_squared_error
def huber_loss(y_true, y_pred, delta=1.0):
    error = y_true - y_pred
    is_small = np.abs(error) <= delta
    loss = np.where(is_small, 0.5 * error**2, delta * (np.abs(error) - 0.5 * delta))
    return loss.mean()
```

#### Ranking Metrics
```python
from sklearn.metrics import ndcg_score, average_precision_score

# NDCG (Normalized Discounted Cumulative Gain)
# y_true: relevance scores, y_pred: predicted scores
ndcg = ndcg_score([y_true], [y_pred], k=10)

# MAP (Mean Average Precision)
# For binary relevance
map_score = average_precision_score(y_true, y_pred)

# MRR (Mean Reciprocal Rank)
def mrr_score(y_true, y_pred):
    order = np.argsort(y_pred)[::-1]
    for rank, idx in enumerate(order):
        if y_true[idx] == 1:
            return 1.0 / (rank + 1)
    return 0.0
```

---

## 2. VALIDATION TECHNIQUES

### WHY? (Problem Statement)
**Unbiased performance estimation.** Prevent overfitting to validation set, detect data leakage.

### HOW? (Cross-Validation Strategies)

#### Standard CV
```python
from sklearn.model_selection import (
    KFold, StratifiedKFold, TimeSeriesSplit, 
    GroupKFold, LeaveOneGroupOut, ShuffleSplit
)

# Classification
cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)

# Regression
cv = KFold(n_splits=5, shuffle=True, random_state=42)

# Time Series (NO SHUFFLE!)
cv = TimeSeriesSplit(n_splits=5)

# Grouped Data (user, session, customer)
cv = GroupKFold(n_splits=5)
scores = cross_val_score(model, X, y, cv=cv, groups=groups)

# Leave-One-Group-Out (max bias reduction)
cv = LeaveOneGroupOut()
```

#### Nested CV (Unbiased Hyperparameter Selection)
```python
from sklearn.model_selection import GridSearchCV, cross_val_score

# Inner CV: Hyperparameter tuning
inner_cv = StratifiedKFold(n_splits=3, shuffle=True, random_state=42)
grid = GridSearchCV(model, param_grid, cv=inner_cv, scoring='roc_auc')

# Outer CV: Unbiased performance estimate
outer_cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
nested_scores = cross_val_score(grid, X, y, cv=outer_cv, scoring='roc_auc')

print(f"Nested CV: {nested_scores.mean():.4f} ± {nested_scores.std():.4f}")
```

#### Bootstrap Validation
```python
def bootstrap_validate(model, X, y, n_bootstrap=1000):
    """Bootstrap .632+ estimator"""
    n = len(X)
    scores = []
    
    for _ in range(n_bootstrap):
        # Sample with replacement
        idx = np.random.choice(n, n, replace=True)
        oob_idx = np.setdiff1d(np.arange(n), idx)  # Out-of-bag
        
        if len(oob_idx) == 0:
            continue
            
        model.fit(X[idx], y[idx])
        pred = model.predict(X[oob_idx])
        scores.append(roc_auc_score(y[oob_idx], pred))
    
    return np.mean(scores), np.std(scores)

# .632+ Bootstrap (Efron & Tibshirani)
def bootstrap_632plus(model, X, y, n_bootstrap=1000):
    n = len(X)
    boot_scores = []
    app_scores = []  # Apparent (training) scores
    
    for _ in range(n_bootstrap):
        idx = np.random.choice(n, n, replace=True)
        oob_idx = np.setdiff1d(np.arange(n), idx)
        
        if len(oob_idx) == 0:
            continue
        
        model.fit(X[idx], y[idx])
        
        # OOB score
        boot_scores.append(roc_auc_score(y[oob_idx], model.predict(X[oob_idx])))
        # Apparent score
        app_scores.append(roc_auc_score(y[idx], model.predict(X[idx])))
    
    # .632+ weighting
    rel_overfit = (np.mean(app_scores) - np.mean(boot_scores)) / (1 - np.mean(app_scores))
    w = 0.632 / (1 - 0.368 * rel_overfit)
    score = (1 - w) * np.mean(app_scores) + w * np.mean(boot_scores)
    
    return score
```

---

## 3. HYPERPARAMETER TUNING

### WHY? (Problem Statement)
**Default parameters ≠ optimal.** Hyperparameter optimization finds best configuration.

### HOW? (Optimization Methods)

#### Grid Search
```python
from sklearn.model_selection import GridSearchCV

param_grid = {
    'n_estimators': [100, 200, 500],
    'max_depth': [3, 5, 7, None],
    'learning_rate': [0.01, 0.1, 0.3],
    'subsample': [0.8, 1.0],
    'colsample_bytree': [0.8, 1.0],
}

grid = GridSearchCV(
    XGBClassifier(random_state=42),
    param_grid,
    cv=5,
    scoring='roc_auc',
    n_jobs=-1,
    verbose=1
)
grid.fit(X, y)
print(grid.best_params_, grid.best_score_)
```

#### Random Search (More Efficient)
```python
from sklearn.model_selection import RandomizedSearchCV
from scipy.stats import randint, uniform, loguniform

param_dist = {
    'n_estimators': randint(100, 1000),
    'max_depth': randint(3, 20),
    'learning_rate': loguniform(1e-3, 0.3),
    'subsample': uniform(0.5, 0.5),
    'colsample_bytree': uniform(0.5, 0.5),
    'reg_alpha': loguniform(1e-8, 10),
    'reg_lambda': loguniform(1e-8, 10),
    'min_child_weight': randint(1, 20),
    'gamma': uniform(0, 5),
}

rand = RandomizedSearchCV(
    XGBClassifier(random_state=42),
    param_dist,
    n_iter=200,
    cv=5,
    scoring='roc_auc',
    n_jobs=-1,
    random_state=42
)
rand.fit(X, y)
```

#### Bayesian Optimization (Optuna)
```python
import optuna

def objective(trial):
    params = {
        'n_estimators': trial.suggest_int('n_estimators', 100, 2000),
        'max_depth': trial.suggest_int('max_depth', 3, 12),
        'learning_rate': trial.suggest_float('learning_rate', 1e-3, 0.3, log=True),
        'subsample': trial.suggest_float('subsample', 0.5, 1.0),
        'colsample_bytree': trial.suggest_float('colsample_bytree', 0.5, 1.0),
        'reg_alpha': trial.suggest_float('reg_alpha', 1e-8, 10.0, log=True),
        'reg_lambda': trial.suggest_float('reg_lambda', 1e-8, 10.0, log=True),
        'min_child_weight': trial.suggest_int('min_child_weight', 1, 20),
        'gamma': trial.suggest_float('gamma', 0, 5.0),
        'random_state': 42,
        'n_jobs': -1,
    }
    
    model = XGBClassifier(**params)
    scores = cross_val_score(model, X, y, cv=StratifiedKFold(5), 
                             scoring='roc_auc', n_jobs=-1)
    return scores.mean()

study = optuna.create_study(
    direction='maximize',
    sampler=optuna.samplers.TPESampler(seed=42),
    pruner=optuna.pruners.MedianPruner(n_warmup_steps=10)
)
study.optimize(objective, n_trials=200, timeout=3600)

print(f"Best: {study.best_value:.4f}")
print(f"Params: {study.best_params}")
```

#### Successive Halving (Fast)
```python
from sklearn.model_selection import HalvingGridSearchCV, HalvingRandomSearchCV

# HalvingGridSearchCV: successive halving with grid
halving = HalvingGridSearchCV(
    XGBClassifier(random_state=42),
    param_grid,
    cv=5,
    factor=3,           # Eliminate 2/3 each round
    min_resources=50,   # Start with 50 samples
    max_resources='auto',
    scoring='roc_auc',
    n_jobs=-1
)
halving.fit(X, y)
```

---

## 4. DATA LEAKAGE PREVENTION

### WHY? (Problem Statement)
**Leakage = overly optimistic results.** Model learns from information not available at prediction time.

### HOW? (Common Leakage Sources & Prevention)

| Leakage Type | Example | Prevention |
|--------------|---------|------------|
| **Target leakage** | Using future target as feature | Remove target-derived features |
| **Train-test leakage** | Preprocessing on full data | Pipeline: fit on train, transform test |
| **Feature selection leakage** | Select on full data | Select inside CV |
| **Time leakage** | Future data in training | Time-series split, expanding window |
| **Group leakage** | Same user in train & test | GroupKFold |
| **Target encoding leakage** | Encode with full target mean | CV-safe encoding (leave-one-out, smoothing) |

```python
# WRONG: Leakage!
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)  # Fits on ALL data!
X_train, X_test = train_test_split(X_scaled, ...)

# CORRECT: Pipeline
pipeline = Pipeline([
    ('scaler', StandardScaler()),
    ('model', LogisticRegression())
])
cross_val_score(pipeline, X, y, cv=5)  # Safe!

# WRONG: Feature selection on full data
selector = SelectKBest(f_classif, k=10).fit(X, y)
X_selected = selector.transform(X)

# CORRECT: Inside pipeline
pipeline = Pipeline([
    ('selector', SelectKBest(f_classif, k=10)),
    ('model', LogisticRegression())
])

# Time series: NO random shuffle!
tscv = TimeSeriesSplit(n_splits=5)
cross_val_score(pipeline, X, y, cv=tscv)

# Grouped: GroupKFold
gkf = GroupKFold(n_splits=5)
cross_val_score(pipeline, X, y, cv=gkf, groups=user_ids)
```

---

## 5. MODEL COMPARISON & STATISTICAL TESTING

### WHY? (Problem Statement)
**Is model A actually better than B?** Statistical tests distinguish real improvements from noise.

### HOW? (Tests)

#### Paired t-test (Same CV Folds)
```python
from scipy import stats

# Cross-validate both models on SAME folds
cv = StratifiedKFold(5, shuffle=True, random_state=42)
scores_a = cross_val_score(model_a, X, y, cv=cv, scoring='roc_auc')
scores_b = cross_val_score(model_b, X, y, cv=cv, scoring='roc_auc')

# Paired t-test
t_stat, p_val = stats.ttest_rel(scores_a, scores_b)
print(f"t={t_stat:.3f}, p={p_val:.4f}")

# Wilcoxon signed-rank (non-parametric)
w_stat, p_val = stats.wilcoxon(scores_a, scores_b)
```

#### 5x2 CV Test (Dietterich)
```python
from sklearn.model_selection import train_test_split
from sklearn.base import clone
from sklearn.metrics import roc_auc_score

def five_two_cv(model1, model2, X, y):
    diffs = []
    for i in range(5):
        # Split 1
        X1, X2, y1, y2 = train_test_split(X, y, test_size=0.5, random_state=i)
        
        m1 = clone(model1).fit(X1, y1)
        m2 = clone(model2).fit(X1, y1)
        diff1 = roc_auc_score(y2, m1.predict_proba(X2)[:,1]) - \
                roc_auc_score(y2, m2.predict_proba(X2)[:,1])
        
        # Split 2 (swap)
        m1 = clone(model1).fit(X2, y2)
        m2 = clone(model2).fit(X2, y2)
        diff2 = roc_auc_score(y1, m1.predict_proba(X1)[:,1]) - \
                roc_auc_score(y1, m2.predict_proba(X1)[:,1])
        
        diffs.extend([diff1, diff2])
    
    diffs = np.array(diffs)
    # Variance estimator
    var = np.sum((diffs - np.mean(diffs))**2) / 5
    t_stat = np.mean(diffs) / np.sqrt(var/5)
    p_val = 2 * (1 - stats.t.cdf(abs(t_stat), df=5))
    return t_stat, p_val
```

#### McNemar's Test (Same Test Set)
```python
from statsmodels.stats.contingency_tables import mcnemar

# Binary predictions on SAME test set
correct1 = (pred1 == y_test).astype(int)
correct2 = (pred2 == y_test).astype(int)

# Contingency table
#                Model 2 Correct | Model 2 Wrong
# Model 1 Correct      a              b
# Model 1 Wrong        c              d

a = np.sum((correct1 == 1) & (correct2 == 1))
b = np.sum((correct1 == 1) & (correct2 == 0))
c = np.sum((correct1 == 0) & (correct2 == 1))
d = np.sum((correct1 == 0) & (correct2 == 0))

table = [[a, b], [c, d]]
result = mcnemar(table, exact=True)
print(f"McNemar: χ²={result.statistic:.4f}, p={result.pvalue:.4f}")
```

#### Friedman + Nemenyi (Multiple Models)
```python
from scipy.stats import friedmanchisquare

# Rank models on each fold
cv = StratifiedKFold(5, shuffle=True, random_state=42)
n_models = len(models)
n_splits = 5
ranks = np.zeros((n_splits, n_models))

for fold_idx, (train_idx, val_idx) in enumerate(cv.split(X, y)):
    scores = []
    for name, model in models.items():
        model.fit(X[train_idx], y[train_idx])
        score = roc_auc_score(y[val_idx], model.predict_proba(X[val_idx])[:,1])
        scores.append(score)
    ranks[fold_idx] = len(scores) - np.argsort(np.argsort(scores))

# Friedman test
friedman_stat, p = friedmanchisquare(*[ranks[:,i] for i in range(n_models)])

if p < 0.05:
    # Nemenyi post-hoc
    avg_ranks = np.mean(ranks, axis=0)
    cd = 1.96 * np.sqrt(n_models * (n_models + 1) / (6 * n_splits))
    # Models with rank diff > cd are significantly different
```

---

## PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Metric aligned with business** | Always use accuracy | Wrong optimization target |
| **CV for performance** | Single train/test split | High variance, unreliable |
| **Nested CV for selection** | CV on tuned model | Optimistic bias |
| **Pipeline for preprocessing** | Manual preprocessing | Leakage, inconsistency |
| **Statistical testing** | Compare means only | Can't distinguish noise |
| **TimeSeriesSplit for temporal** | Random split for time series | Look-ahead bias |
| **GroupKFold for grouped data** | Random split for groups | Data leakage |
| **Multiple testing correction** | Many tests uncorrected | False discoveries |

---

## INTERVIEW QUESTIONS (Top 15)

### 1. **Conceptual**: "What's the difference between AUC-ROC and AUC-PR?"

```
AUC-ROC:
- Plots TPR vs FPR at all thresholds
- Insensitive to class imbalance (uses TNR in denominator)
- Good for balanced data, comparing models

AUC-PR:
- Plots Precision vs Recall at all thresholds
- Sensitive to class imbalance (focuses on positive class)
- Better for imbalanced data (fraud, rare disease)
- Baseline = positive class ratio (not 0.5)

WHEN TO USE:
- Balanced / general → AUC-ROC
- Imbalanced / positive class focus → AUC-PR
- Both → Report both

EXAMPLE:
Fraud: 0.1% positive
AUC-ROC = 0.99 (looks great!)
AUC-PR = 0.15 (reveals reality)
```

### 2. **Code**: "Implement optimal threshold selection for F1."

```python
def find_optimal_threshold(y_true, y_prob, metric='f1'):
    """Find threshold that maximizes metric"""
    from sklearn.metrics import precision_recall_curve, f1_score
    
    prec, rec, thresholds = precision_recall_curve(y_true, y_prob)
    
    if metric == 'f1':
        f1_scores = 2 * prec * rec / (prec + rec + 1e-10)
        best_idx = np.argmax(f1_scores)
        best_score = f1_scores[best_idx]
    elif metric == 'fbeta':
        beta = 2  # F2
        fbeta = (1 + beta**2) * prec * rec / (beta**2 * prec + rec + 1e-10)
        best_idx = np.argmax(fbeta)
        best_score = fbeta[best_idx]
    elif metric == 'precision_at_recall':
        target_recall = 0.9
        valid = rec >= target_recall
        if np.any(valid):
            best_idx = np.argmax(prec[valid])
            best_score = prec[best_idx]
        else:
            best_idx = 0
            best_score = 0
    elif metric == 'business_cost':
        # Cost matrix: FP_cost, FN_cost
        fp_cost, fn_cost = 10, 100
        costs = fp_cost * (1 - prec) + fn_cost * (1 - rec)
        best_idx = np.argmin(costs)
        best_score = -costs[best_idx]
    
    best_threshold = thresholds[best_idx] if best_idx < len(thresholds) else 1.0
    return best_threshold, best_score

# Youden's J statistic (max TPR - FPR)
def youden_threshold(y_true, y_prob):
    from sklearn.metrics import roc_curve
    fpr, tpr, thresholds = roc_curve(y_true, y_prob)
    j = tpr - fpr
    best_idx = np.argmax(j)
    return thresholds[best_idx], j[best_idx]
```

### 3. **Design**: "How do you evaluate a model for imbalanced classification (1:1000)?"

```
IMBALANCED EVALUATION STRATEGY:

1. NEVER USE ACCURACY
   - 99.9% accuracy by predicting all negative!

2. PRIMARY METRICS:
   - AUC-PR (not AUC-ROC)
   - F1, Precision@K, Recall@K
   - Matthews Correlation Coefficient (MCC)
   - Balanced Accuracy = (Recall + Specificity) / 2

3. THRESHOLD TUNING:
   - Default 0.5 is wrong
   - Optimize for business cost
   - Precision-Recall curve → best F1

4. EVALUATION PROTOCOL:
   - StratifiedKFold (preserve class ratio)
   - Multiple metrics reported
   - Confusion matrix at operating threshold
   - Calibration check (reliability diagram)

5. BASELINES:
   - Random classifier
   - Always predict majority
   - Simple heuristic (e.g., recent purchasers)

6. CODE:
scoring = ['roc_auc', 'average_precision', 'f1', 'precision', 'recall', 'mcc']
cv_results = cross_validate(model, X, y, cv=StratifiedKFold(5), scoring=scoring)
```

### 4. **Code**: "Implement K-Fold CV with proper leakage prevention."

```python
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.feature_selection import SelectKBest, f_classif
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import cross_validate, StratifiedKFold

# Full pipeline prevents leakage
pipeline = Pipeline([
    ('imputer', SimpleImputer(strategy='median')),
    ('scaler', StandardScaler()),
    ('selector', SelectKBest(f_classif, k=20)),
    ('model', LogisticRegression(class_weight='balanced', max_iter=1000, random_state=42))
])

# Cross-validation with multiple metrics
scoring = ['roc_auc', 'average_precision', 'f1', 'precision', 'recall', 'accuracy']
cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)

cv_results = cross_validate(
    pipeline, X, y,
    cv=cv,
    scoring=scoring,
    return_train_score=True,
    return_estimator=True
)

for metric in scoring:
    train_scores = cv_results[f'train_{metric}']
    test_scores = cv_results[f'test_{metric}']
    print(f"{metric}: train={train_scores.mean():.4f}±{train_scores.std():.4f}, "
          f"test={test_scores.mean():.4f}±{test_scores.std():.4f}")

# Check overfitting
overfit = cv_results['train_roc_auc'].mean() - cv_results['test_roc_auc'].mean()
print(f"Overfitting gap: {overfit:.4f}")
```

### 5. **Conceptual**: "What is data leakage and how do you prevent it?"

```
DATA LEAKAGE = Using information during training that won't be available at inference.

TYPES:
1. TARGET LEAKAGE:
   - Feature derived from target (e.g., "days_since_last_purchase" for churn)
   - Future values used to predict past
   - Prevention: Feature audit, temporal splits

2. PREPROCESSING LEAKAGE:
   - Fit scaler/imputer/encoder on full dataset
   - Feature selection on full data
   - Prevention: Pipeline (fit on train fold only)

3. TIME LEAKAGE:
   - Random split for time series
   - Future data in training set
   - Prevention: TimeSeriesSplit, expanding window

4. GROUP LEAKAGE:
   - Same user/group in train and test
   - Prevention: GroupKFold, LeaveOneGroupOut

5. TARGET ENCODING LEAKAGE:
   - Mean target encoding using full data
   - Prevention: CV-safe encoding (leave-one-out, smoothing)

PREVENTION CHECKLIST:
☐ All preprocessing in Pipeline/ColumnTransformer
☐ Temporal splits for time series
☐ Group splits for grouped data
☐ Feature engineering without target
☐ No peeking at test set
☐ Nested CV for hyperparameter tuning
```

### 6. **Code**: "Implement time series cross-validation."

```python
from sklearn.model_selection import TimeSeriesSplit
import numpy as np

def expanding_window_cv(model, X, y, n_splits=5, initial_train_size=None):
    """Expanding window (cumulative training set)"""
    tscv = TimeSeriesSplit(n_splits=n_splits)
    scores = []
    
    for fold, (train_idx, val_idx) in enumerate(tscv.split(X)):
        if initial_train_size:
            # Expanding: always start from 0
            train_idx = np.arange(val_idx[0])
        else:
            # Sliding: use tscv default
            pass
        
        X_train, X_val = X[train_idx], X[val_idx]
        y_train, y_val = y[train_idx], y[val_idx]
        
        model.fit(X_train, y_train)
        pred = model.predict_proba(X_val)[:, 1]
        score = roc_auc_score(y_val, pred)
        scores.append(score)
        print(f"Fold {fold}: Train={len(train_idx)}, Val={len(val_idx)}, AUC={score:.4f}")
    
    return np.mean(scores), np.std(scores)

def walk_forward_validation(model, X, y, step=1, min_train_size=100):
    """Walk-forward (rolling forecast origin)"""
    n = len(X)
    scores = []
    
    for i in range(min_train_size, n, step):
        train_end = i
        test_end = min(i + step, n)
        
        X_train, y_train = X[:train_end], y[:train_end]
        X_test, y_test = X[train_end:test_end], y[train_end:test_end]
        
        model.fit(X_train, y_train)
        pred = model.predict_proba(X_test)[:, 1]
        score = roc_auc_score(y_test, pred)
        scores.append(score)
    
    return scores

# Purged K-Fold (for financial time series - prevents leakage from overlap)
# from mlfinlab.cross_validation import PurgedKFold
# purged_cv = PurgedKFold(n_splits=5, pct_embargo=0.01)
```

### 7. **Conceptual**: "Explain the difference between GridSearch, RandomSearch, and Bayesian Optimization."

```
GRID SEARCH:
- Exhaustive search over parameter grid
- Guarantees finding global optimum in grid
- Exponential complexity: O(k^d)
- Good for few params, small grids

RANDOM SEARCH:
- Sample random parameter combinations
- O(n_iter) independent of dimension
- "More efficient than grid" (Bergstra & Bengio 2012)
- Good baseline, parallelizes trivially

BAYESIAN OPTIMIZATION (Optuna, Hyperopt, GPyOpt):
- Build surrogate model p(score|params)
- Acquisition function (EI, UCB, PI) guides search
- Sequential: each trial informs next
- Best for expensive models, many params
- Handles conditional params, constraints

SUCCESSIVE HALVING / HYPERBAND:
- Start many configs with few resources
- Iteratively halve configs, double resources
- 5-10x faster than random for same quality
- Good for iterative models (trees, NN)

RECOMMENDATION:
1. Start: Random Search (baseline, fast)
2. Scale: Bayesian (Optuna) for expensive models
3. Iterative models: Hyperband / Successive Halving
4. Few params (<5): Grid Search
```

### 8. **Code**: "Implement Optuna optimization with pruning."

```python
import optuna
from sklearn.model_selection import cross_val_score, StratifiedKFold

def optimize_xgboost(X, y, n_trials=200, timeout=3600):
    def objective(trial):
        params = {
            'n_estimators': trial.suggest_int('n_estimators', 100, 2000),
            'max_depth': trial.suggest_int('max_depth', 3, 12),
            'learning_rate': trial.suggest_float('learning_rate', 1e-3, 0.3, log=True),
            'subsample': trial.suggest_float('subsample', 0.5, 1.0),
            'colsample_bytree': trial.suggest_float('colsample_bytree', 0.5, 1.0),
            'reg_alpha': trial.suggest_float('reg_alpha', 1e-8, 10.0, log=True),
            'reg_lambda': trial.suggest_float('reg_lambda', 1e-8, 10.0, log=True),
            'min_child_weight': trial.suggest_int('min_child_weight', 1, 20),
            'gamma': trial.suggest_float('gamma', 0, 5.0),
            'random_state': 42,
            'n_jobs': -1,
            'eval_metric': 'auc',
        }
        
        model = xgb.XGBClassifier(**params)
        
        # Pruning callback for Optuna
        pruning_callback = optuna.integration.XGBoostPruningCallback(trial, 'validation_0-auc')
        
        scores = cross_val_score(
            model, X, y,
            cv=StratifiedKFold(5, shuffle=True, random_state=42),
            scoring='roc_auc',
            fit_params={'eval_set': [(X, y)], 'callbacks': [pruning_callback]},
            n_jobs=-1
        )
        return scores.mean()
    
    study = optuna.create_study(
        direction='maximize',
        sampler=optuna.samplers.TPESampler(seed=42),
        pruner=optuna.pruners.MedianPruner(n_warmup_steps=10)
    )
    
    study.optimize(objective, n_trials=n_trials, timeout=timeout)
    
    print(f"Best AUC: {study.best_value:.4f}")
    print(f"Best params: {study.best_params}")
    
    return study.best_params, study.best_value
```

### 9. **Conceptual**: "What's the difference between nested CV and regular CV?"

```
REGULAR CV:
- Single CV loop for hyperparameter tuning
- Best model selected on same CV scores
- Problem: Optimistic bias (selection bias on CV estimates)
- CV error underestimates true test error

NESTED CV:
- TWO CV loops: Inner + Outer
- Inner CV: Hyperparameter tuning (model selection)
- Outer CV: Performance estimation (unbiased)
- Outer test folds NEVER used for model selection

WHEN NEEDED:
- Comparing multiple models/algorithms
- Reporting final performance estimate
- Publishing results
- High-stakes decisions

COST:
- 5x outer × 3x inner = 15x more training vs single CV
- Worth it for unbiased estimate

ALTERNATIVE:
- Holdout validation set (single split)
- Faster but higher variance
- Use nested CV when possible
```

### 10. **Code**: "Implement statistical comparison of multiple models."

```python
from scipy import stats
from scipy.stats import friedmanchisquare
from sklearn.model_selection import cross_val_score, StratifiedKFold
import numpy as np

def compare_models_statistically(models, X, y, cv=5):
    """Compare multiple models with statistical tests"""
    cv_strategy = StratifiedKFold(n_splits=cv, shuffle=True, random_state=42)
    model_names = list(models.keys())
    n_models = len(model_names)
    
    # Collect scores for each model
    all_scores = {}
    for name, model in models.items():
        scores = cross_val_score(model, X, y, cv=cv_strategy, scoring='roc_auc')
        all_scores[name] = scores
        print(f"{name}: {np.mean(scores):.4f} ± {np.std(scores):.4f}")
    
    # 1. Friedman test (non-parametric ANOVA)
    ranks = np.zeros((cv, n_models))
    for fold_idx in range(cv):
        fold_scores = [all_scores[name][fold_idx] for name in model_names]
        ranks[fold_idx] = n_models - np.argsort(np.argsort(fold_scores))
    
    friedman_stat, p_friedman = friedmanchisquare(*[ranks[:, i] for i in range(n_models)])
    print(f"\nFriedman test: χ²={friedman_stat:.3f}, p={p_friedman:.4f}")
    
    if p_friedman < 0.05:
        # 2. Nemenyi post-hoc test
        avg_ranks = np.mean(ranks, axis=0)
        cd = 1.96 * np.sqrt(n_models * (n_models + 1) / (6 * cv))
        
        print(f"Average ranks: {dict(zip(model_names, avg_ranks))}")
        print(f"Critical difference (CD): {cd:.3f}")
        
        print("\nPairwise comparisons (rank diff > CD = significant):")
        for i, name1 in enumerate(model_names):
            for name2 in model_names[i+1:]:
                j = model_names.index(name2)
                diff = abs(avg_ranks[i] - avg_ranks[j])
                sig = diff > cd
                print(f"  {name1} vs {name2}: diff={diff:.3f} {'***' if sig else ''}")
    
    # 3. Pairwise paired t-tests (with Bonferroni correction)
    n_comparisons = n_models * (n_models - 1) // 2
    alpha_adj = 0.05 / n_comparisons
    
    print(f"\nPaired t-tests (Bonferroni α={alpha_adj:.4f}):")
    for i, name1 in enumerate(model_names):
        for name2 in model_names[i+1:]:
            t_stat, p_val = stats.ttest_rel(all_scores[name1], all_scores[name2])
            sig = p_val < alpha_adj
            better = name1 if np.mean(all_scores[name1]) > np.mean(all_scores[name2]) else name2
            print(f"  {name1} vs {name2}: p={p_val:.4f} {'***' if sig else ''} (better: {better})")

# McNemar's test for two classifiers on same test set
def mcnemar_test(y_true, pred1, pred2):
    from statsmodels.stats.contingency_tables import mcnemar
    correct1 = (pred1 == y_true).astype(int)
    correct2 = (pred2 == y_true).astype(int)
    
    table = [[
        np.sum((correct1==1)&(correct2==1)),  # both correct
        np.sum((correct1==1)&(correct2==0))   # 1 correct, 2 wrong
    ], [
        np.sum((correct1==0)&(correct2==1)),  # 1 wrong, 2 correct
        np.sum((correct1==0)&(correct2==0))   # both wrong
    ]]
    
    result = mcnemar(table, exact=True)
    return result.statistic, result.pvalue
```

### 11. **Conceptual**: "How do you detect overfitting in cross-validation?"

```
OVERFITTING DETECTION IN CV:

1. TRAIN-TEST GAP:
   cv_results = cross_validate(..., return_train_score=True)
   gap = train_scores.mean() - test_scores.mean()
   - Large gap → overfitting
   - Rule of thumb: gap > 0.05-0.10 concerning

2. LEARNING CURVES:
   - Plot train/val score vs training set size
   - High variance: gap persists at large n
   - High bias: both low, converge

3. VALIDATION CURVES:
   - Plot train/val score vs hyperparameter (e.g., max_depth)
   - Overfitting: train increases, val decreases

4. STABILITY ACROSS FOLDS:
   - High std of CV scores = unstable
   - Check individual fold scores

5. NESTED CV vs REGULAR CV:
   - If nested CV << regular CV → selection bias/overfitting

VISUALIZATION:
from sklearn.model_selection import learning_curve, validation_curve

train_sizes, train_scores, val_scores = learning_curve(
    model, X, y, cv=5, train_sizes=np.linspace(0.1, 1.0, 10)
)

plt.plot(train_sizes, train_scores.mean(axis=1), label='Train')
plt.fill_between(train_sizes, 
                 train_scores.mean(axis=1) - train_scores.std(axis=1),
                 train_scores.mean(axis=1) + train_scores.std(axis=1), alpha=0.2)
plt.plot(train_sizes, val_scores.mean(axis=1), label='Val')
plt.fill_between(train_sizes,
                 val_scores.mean(axis=1) - val_scores.std(axis=1),
                 val_scores.mean(axis=1) + val_scores.std(axis=1), alpha=0.2)
```

### 12. **Code**: "Calibration curve and expected calibration error."

```python
from sklearn.calibration import calibration_curve, CalibratedClassifierCV
import matplotlib.pyplot as plt

def plot_calibration_curve(y_true, y_prob, n_bins=10):
    """Reliability diagram"""
    prob_true, prob_pred = calibration_curve(y_true, y_prob, n_bins=n_bins)
    
    plt.figure(figsize=(6, 6))
    plt.plot([0, 1], [0, 1], 'k--', label='Perfectly calibrated')
    plt.plot(prob_pred, prob_true, 's-', label='Model')
    plt.xlabel('Mean Predicted Probability')
    plt.ylabel('Fraction of Positives')
    plt.title('Calibration Curve (Reliability Diagram)')
    plt.legend()
    plt.grid(True, alpha=0.3)
    plt.show()

def expected_calibration_error(y_true, y_prob, n_bins=10):
    """ECE = Σ (|acc_bin - conf_bin| * prop_bin)"""
    bin_boundaries = np.linspace(0, 1, n_bins + 1)
    bin_lowers = bin_boundaries[:-1]
    bin_uppers = bin_boundaries[1:]
    
    ece = 0
    for bin_lower, bin_upper in zip(bin_lowers, bin_uppers):
        in_bin = (y_prob > bin_lower) & (y_prob <= bin_upper)
        prop_in_bin = in_bin.mean()
        
        if prop_in_bin > 0:
            accuracy_in_bin = y_true[in_bin].mean()
            avg_confidence_in_bin = y_prob[in_bin].mean()
            ece += np.abs(avg_confidence_in_bin - accuracy_in_bin) * prop_in_bin
    
    return ece

def calibrate_model(model, X_train, y_train, X_val, y_val, method='isotonic'):
    """Calibrate probabilities"""
    calibrator = CalibratedClassifierCV(model, method=method, cv='prefit')
    calibrator.fit(X_val, y_val)
    return calibrator

# Usage
ece = expected_calibration_error(y_test, y_prob)
print(f"Expected Calibration Error: {ece:.4f}")

# Calibrate
calibrated = calibrate_model(model, X_train, y_train, X_val, y_val)
y_prob_cal = calibrated.predict_proba(X_test)[:, 1]
ece_cal = expected_calibration_error(y_test, y_prob_cal)
print(f"ECE after calibration: {ece_cal:.4f}")
```

---

## QUICK REFERENCE: MODEL EVALUATION CHEAT SHEET

### Metric Selection by Problem
| Problem | Primary Metric | Secondary |
|---------|---------------|-----------|
| Balanced classification | AUC-ROC, Accuracy | F1, Precision, Recall |
| Imbalanced classification | AUC-PR, F1 | Precision@K, Recall@K, MCC |
| Cost-sensitive | Business cost / Fβ | Precision/Recall at operating point |
| Probability estimation | Log Loss, Brier Score | Calibration curve, ECE |
| Ranking | NDCG, MAP, MRR | AUC-ROC |
| Regression | RMSE, MAE | R², MAPE |
| Time series forecasting | MAE, SMAPE | MASE, RMSE |

### CV Strategy Selection
| Data Type | CV Strategy |
|-----------|-------------|
| IID classification | StratifiedKFold |
| IID regression | KFold |
| Time series | TimeSeriesSplit / Expanding Window |
| Grouped (user, session) | GroupKFold / LeaveOneGroupOut |
| Small dataset | LeaveOneOut / RepeatedKFold |
| Model selection | Nested CV |

### Leakage Prevention Checklist
- [ ] All preprocessing in Pipeline
- [ ] Temporal splits for time series
- [ ] Group splits for grouped data
- [ ] Feature selection inside CV
- [ ] Target encoding CV-safe
- [ ] No test set peeking
- [ ] Nested CV for model selection

### Statistical Test Selection
| Comparison | Test |
|------------|------|
| 2 models, same CV folds | Paired t-test / Wilcoxon |
| 2 models, same test set | McNemar's test |
| 2 models, 5x2 CV | Dietterich 5x2 CV test |
| >2 models | Friedman + Nemenyi |
| Hyperparameter search | Nested CV (unbiased) |

### Hyperparameter Optimization
| Method | When |
|--------|------|
| Grid Search | <5 params, small grid |
| Random Search | Baseline, many params |
| Bayesian (Optuna) | Expensive models, many params |
| Successive Halving | Iterative models (trees, NN) |
| Hyperband | Unknown budget, many configs |