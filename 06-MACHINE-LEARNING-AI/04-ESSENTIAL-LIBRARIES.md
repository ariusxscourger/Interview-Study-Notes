# ESSENTIAL LIBRARIES FOR MACHINE LEARNING
## Exam-Style Study Notes

**Roadmap Section:** Essential Libraries (NumPy, Pandas, Matplotlib, Seaborn, Scikit-learn)
**Prerequisites:** Python fundamentals, basic statistics, linear algebra

---

## 1. NUMPY (Numerical Python)

### WHY? (Problem Statement)
**Foundation of all ML computing.** NumPy provides:
- N-dimensional arrays (ndarray) - efficient, contiguous memory
- Vectorized operations - 10-100x faster than Python loops
- Broadcasting - automatic shape alignment
- Random number generation - reproducible experiments

### HOW? (Core Concepts)

#### Array Creation & Manipulation
```python
import numpy as np

# Creation
np.array([1,2,3])                    # From list
np.zeros((3,4)), np.ones((3,4))      # Zeros/ones
np.eye(3)                            # Identity matrix
np.arange(0, 10, 2)                  # Range with step
np.linspace(0, 1, 10)                # Linear spaced
np.random.normal(0, 1, (3,4))        # Random normal
np.random.rand(3,4)                  # Uniform [0,1)

# Shape manipulation
a = np.arange(12).reshape(3,4)       # Reshape
a.T                                  # Transpose
a.ravel()                            # Flatten
np.concatenate([a, b], axis=0)       # Join arrays
np.hstack([a, b]), np.vstack([a, b]) # Horizontal/vertical stack
np.split(a, 3, axis=1)               # Split
```

#### Indexing & Broadcasting
```python
# Basic indexing
a = np.arange(12).reshape(3,4)
a[0, 2]              # Single element
a[1:3, 1:3]          # Slice
a[:, 1]              # Column 1
a[0]                 # Row 0

# Fancy indexing
a[[0, 2], [1, 3]]    # Elements at (0,1) and (2,3)
a[a > 5]             # Boolean mask
a[(a > 2) & (a < 8)] # Compound condition

# Broadcasting rules (align from right)
a = np.arange(6).reshape(3,2)  # (3,2)
b = np.array([10, 20])         # (2,)
a + b                          # (3,2) + (2,) → (3,2)

# Explicit broadcasting
np.broadcast_to(b, (3,2))
```

#### Vectorized Operations (Performance)
```python
# SLOW (Python loop)
result = []
for i in range(len(a)):
    result.append(a[i] * 2 + 1)

# FAST (vectorized)
result = a * 2 + 1

# Reduction operations
a.sum(), a.mean(), a.std(), a.var()
a.min(), a.max(), a.argmin(), a.argmax()
a.sum(axis=0)          # Column sums
a.sum(axis=1, keepdims=True)  # Row sums, keep dims

# Cumulative operations
a.cumsum(), a.cumprod()
```

#### Linear Algebra
```python
# Matrix multiplication
A @ B              # Matrix multiply
A.dot(B)           # Same
np.matmul(A, B)    # Same

# Decompositions
U, s, Vt = np.linalg.svd(A)           # SVD
eigvals, eigvecs = np.linalg.eig(A)   # Eigendecomposition
Q, R = np.linalg.qr(A)                # QR decomposition
L, U = np.linalg.cholesky(A)          # Cholesky (positive definite)

# Solve linear system
x = np.linalg.solve(A, b)  # Ax = b
x = np.linalg.lstsq(A, b, rcond=None)[0]  # Least squares

# Norms
np.linalg.norm(x, 2)   # L2 norm
np.linalg.norm(x, 1)   # L1 norm
np.linalg.norm(A, 'fro')  # Frobenius norm
```

#### Random Number Generation
```python
# Reproducibility
np.random.seed(42)
np.random.default_rng(42)  # New API (preferred)

rng = np.random.default_rng(42)
rng.normal(0, 1, (3,4))      # Normal
rng.uniform(0, 1, (3,4))     # Uniform
rng.integers(0, 10, (3,4))   # Integers
rng.choice([1,2,3], 10)      # Sample with replacement
rng.choice([1,2,3], 10, replace=False)  # Without replacement
rng.permutation(10)          # Shuffle indices
rng.shuffle(arr)             # In-place shuffle

# Distributions
rng.normal(loc=0, scale=1, size=1000)
rng.exponential(scale=1.0, size=1000)
rng.poisson(lam=2, size=1000)
rng.binomial(n=10, p=0.5, size=1000)
rng.multivariate_normal(mean, cov, size=1000)
rng.dirichlet(alpha=[1,2,3], size=1000)
```

---

## 2. PANDAS (Data Manipulation)

### WHY? (Problem Statement)
**DataFrames = SQL + Excel in Python.** Pandas handles:
- Heterogeneous data (mixed types)
- Missing data handling
- Time series operations
- GroupBy aggregations
- Joins/merges

### HOW? (Core Operations)

#### DataFrame Basics
```python
import pandas as pd

# Creation
df = pd.DataFrame({'A': [1,2,3], 'B': ['x','y','z']})
df = pd.read_csv('file.csv', parse_dates=['date'])
df = pd.read_parquet('file.parquet')
df = pd.read_sql('SELECT * FROM table', engine)

# Inspection
df.head(), df.tail(), df.info(), df.describe()
df.shape, df.columns, df.dtypes
df.memory_usage(deep=True)
```

#### Indexing & Selection
```python
# Label-based (loc)
df.loc[0]                    # Row 0
df.loc[0:2, 'A':'C']         # Rows 0-2, columns A-C
df.loc[df['A'] > 5, 'B']     # Boolean mask

# Position-based (iloc)
df.iloc[0]                   # Row 0
df.iloc[0:3, 1:3]            # Rows 0-2, cols 1-2

# Boolean indexing
df[df['A'] > 5]
df.query('A > 5 and B == "x"')
df[df['B'].isin(['x', 'y'])]

# Setting values
df.loc[df['A'] > 5, 'B'] = 'large'
df['C'] = df['A'] * 2
```

#### Missing Data
```python
# Detection
df.isnull().sum()           # Count per column
df.notnull().all()          # Any missing?

# Removal
df.dropna()                 # Drop any row with NaN
df.dropna(axis=1)           # Drop columns with NaN
df.dropna(thresh=3)         # Keep rows with ≥3 non-null
df.dropna(subset=['A', 'B'])# Only check specific columns

# Imputation
df.fillna(0)                # Fill with constant
df.fillna(df.mean())        # Fill with column mean
df.fillna(method='ffill')   # Forward fill
df.fillna(method='bfill')   # Backward fill
df.interpolate()            # Linear interpolation

# Advanced: model-based imputation
from sklearn.impute import KNNImputer, IterativeImputer
imp = IterativeImputer(random_state=42)
df_imputed = pd.DataFrame(imp.fit_transform(df), columns=df.columns)
```

#### GroupBy & Aggregation
```python
# Single aggregation
df.groupby('category')['value'].sum()
df.groupby('category')['value'].agg(['sum', 'mean', 'count'])

# Multiple columns, multiple functions
df.groupby('category').agg({
    'value': ['sum', 'mean', 'std'],
    'count': 'count'
})

# Multiple grouping keys
df.groupby(['category', 'subcategory']).agg('mean')

# Named aggregation (pandas 0.25+)
df.groupby('category').agg(
    total_value=('value', 'sum'),
    avg_value=('value', 'mean'),
    n=('value', 'count')
)

# Transform (broadcast back to original shape)
df['group_mean'] = df.groupby('category')['value'].transform('mean')
df['zscore'] = df.groupby('category')['value'].transform(
    lambda x: (x - x.mean()) / x.std()
)

# Filter
df.groupby('category').filter(lambda x: len(x) > 10)

# Apply custom function
def top_n(group, n=3):
    return group.nlargest(n, 'value')
df.groupby('category').apply(top_n, n=2)
```

#### Time Series
```python
# Datetime parsing
df['date'] = pd.to_datetime(df['date'])
df = df.set_index('date')

# Resampling
df.resample('D').sum()       # Daily
df.resample('W').mean()      # Weekly
df.resample('M').last()      # Month end

# Rolling windows
df['rolling_7'] = df['value'].rolling(7).mean()
df['expanding'] = df['value'].expanding().mean()
df['ewm'] = df['value'].ewm(span=7).mean()

# Shifting
df['lag_1'] = df['value'].shift(1)
df['diff_1'] = df['value'].diff(1)
df['pct_change'] = df['value'].pct_change()

# Time-based grouping
df.groupby(df.index.month).mean()
df.groupby(pd.Grouper(freq='W')).sum()
```

#### Merging & Joining
```python
# Merge (like SQL JOIN)
pd.merge(left, right, on='key', how='inner')   # inner, left, right, outer
pd.merge(left, right, left_on='key1', right_on='key2')

# Merge on index
left.join(right, on='key')      # left index, right column
left.join(right, lsuffix='_l', rsuffix='_r')

# Concat
pd.concat([df1, df2], axis=0)   # Stack vertically
pd.concat([df1, df2], axis=1)   # Side by side

# Update/combine
df1.combine_first(df2)          # Fill NaN in df1 with df2
df1.update(df2)                 # In-place update
```

---

## 3. MATPLOTLIB & SEABORN (Visualization)

### WHY? (Problem Statement)
**Exploratory Data Analysis (EDA) = 80% of ML.** Visualization reveals:
- Distributions, outliers, correlations
- Model diagnostics (residuals, learning curves)
- Feature importance, decision boundaries

### HOW? (Matplotlib - Low-level, Seaborn - High-level)

#### Matplotlib Basics
```python
import matplotlib.pyplot as plt
import numpy as np

# Object-oriented API (recommended)
fig, ax = plt.subplots(figsize=(10, 6))
ax.plot(x, y, 'o-', label='data')
ax.set_xlabel('X')
ax.set_ylabel('Y')
ax.set_title('Title')
ax.legend()
ax.grid(True, alpha=0.3)
plt.tight_layout()
plt.savefig('plot.png', dpi=300, bbox_inches='tight')
plt.show()

# Subplots
fig, axes = plt.subplots(2, 2, figsize=(12, 10))
axes[0, 0].plot(x, y1)
axes[0, 1].scatter(x, y2)
axes[1, 0].hist(data, bins=30)
axes[1, 1].boxplot([group1, group2])

# Twin axes (dual y-axis)
ax1 = ax.twinx()
ax1.plot(x, y2, 'r-', label='y2')
ax1.set_ylabel('Y2', color='r')
```

#### Seaborn (Statistical Visualization)
```python
import seaborn as sns

# Style
sns.set_style('whitegrid')
sns.set_context('talk')  # paper, notebook, talk, poster
sns.color_palette('husl', 8)  # Set palette

# Distributions
sns.histplot(data, kde=True, bins=30)
sns.kdeplot(data, fill=True)
sns.boxplot(x='category', y='value', data=df)
sns.violinplot(x='category', y='value', data=df)
sns.stripplot(x='category', y='value', data=df, jitter=0.2)

# Relationships
sns.scatterplot(x='x', y='y', hue='category', data=df)
sns.regplot(x='x', y='y', data=df)  # Regression line
sns.lmplot(x='x', y='y', hue='category', data=df)

# Categorical
sns.barplot(x='category', y='value', data=df, ci=95)
sns.pointplot(x='time', y='value', hue='group', data=df)
sns.catplot(x='category', y='value', kind='box', data=df)

# Correlation & Heatmaps
corr = df.corr()
sns.heatmap(corr, annot=True, cmap='RdBu_r', center=0, square=True)

# Pairplot (pairwise relationships)
sns.pairplot(df, hue='target', diag_kind='kde')

# FacetGrid (multiple subplots by variable)
g = sns.FacetGrid(df, col='category', row='group')
g.map(sns.histplot, 'value')
```

#### ML-Specific Visualizations
```python
# Learning curves
from sklearn.model_selection import learning_curve
train_sizes, train_scores, val_scores = learning_curve(
    estimator, X, y, cv=5, train_sizes=np.linspace(0.1, 1.0, 10)
)
plt.plot(train_sizes, train_scores.mean(axis=1), label='Train')
plt.plot(train_sizes, val_scores.mean(axis=1), label='Val')
plt.fill_between(train_sizes, 
                 val_scores.mean(axis=1) - val_scores.std(axis=1),
                 val_scores.mean(axis=1) + val_scores.std(axis=1), alpha=0.2)

# ROC Curve
from sklearn.metrics import roc_curve, auc
fpr, tpr, _ = roc_curve(y_true, y_prob)
plt.plot(fpr, tpr, label=f'AUC = {auc(fpr, tpr):.3f}')
plt.plot([0, 1], [0, 1], 'k--')
plt.xlabel('False Positive Rate')
plt.ylabel('True Positive Rate')

# Precision-Recall
from sklearn.metrics import precision_recall_curve
prec, rec, _ = precision_recall_curve(y_true, y_prob)
plt.plot(rec, prec)

# Confusion Matrix
from sklearn.metrics import confusion_matrix
cm = confusion_matrix(y_true, y_pred)
sns.heatmap(cm, annot=True, fmt='d', cmap='Blues')

# Feature Importance
importances = model.feature_importances_
idx = np.argsort(importances)[::-1]
plt.barh(range(len(importances)), importances[idx])
plt.yticks(range(len(importances)), [features[i] for i in idx])

# Residual plots
residuals = y_true - y_pred
plt.scatter(y_pred, residuals)
plt.axhline(0, color='red', linestyle='--')
plt.xlabel('Predicted')
plt.ylabel('Residuals')

# Q-Q plot (normality check)
from scipy.stats import probplot
probplot(residuals, dist="norm", plot=plt)
```

---

## 4. SCIKIT-LEARN (Machine Learning Pipeline)

### WHY? (Problem Statement)
**Production-ready ML toolkit.** Consistent API across:
- Preprocessing
- Feature selection
- Model training
- Evaluation
- Pipelines

### HOW? (Core API Patterns)

#### Estimator API (fit/predict/transform)
```python
from sklearn.base import BaseEstimator, TransformerMixin

# All estimators follow:
# estimator.fit(X, y)          # Learn from data
# estimator.predict(X)         # Predict (supervised)
# estimator.transform(X)       # Transform (unsupervised/preprocessing)
# estimator.fit_transform(X, y) # Fit + transform

# Pipeline: chain transformers + estimator
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.decomposition import PCA
from sklearn.linear_model import LogisticRegression

pipe = Pipeline([
    ('scaler', StandardScaler()),
    ('pca', PCA(n_components=10)),
    ('clf', LogisticRegression())
])

pipe.fit(X_train, y_train)
pipe.predict(X_test)
pipe.predict_proba(X_test)
```

#### Cross-Validation
```python
from sklearn.model_selection import (
    cross_val_score, cross_validate, StratifiedKFold,
    cross_val_predict, KFold, GroupKFold, TimeSeriesSplit
)

# Simple CV
scores = cross_val_score(model, X, y, cv=5, scoring='roc_auc')

# Multiple metrics
scoring = ['accuracy', 'precision', 'recall', 'f1', 'roc_auc']
cv_results = cross_validate(model, X, y, cv=5, scoring=scoring,
                            return_train_score=True)

# Custom CV strategies
cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
cv = GroupKFold(n_splits=5)  # Grouped data
cv = TimeSeriesSplit(n_splits=5)  # Time series

# Predictions from CV
y_pred_cv = cross_val_predict(model, X, y, cv=5, method='predict_proba')[:, 1]
```

#### Hyperparameter Tuning
```python
from sklearn.model_selection import GridSearchCV, RandomizedSearchCV
from sklearn.model_selection import HalvingGridSearchCV, HalvingRandomSearchCV

# Grid Search
param_grid = {
    'n_estimators': [100, 200, 500],
    'max_depth': [3, 5, 7, None],
    'learning_rate': [0.01, 0.1, 0.3],
    'subsample': [0.8, 1.0],
}
grid = GridSearchCV(
    XGBClassifier(), param_grid, cv=5, scoring='roc_auc',
    n_jobs=-1, verbose=1
)
grid.fit(X, y)

# Random Search (more efficient)
param_dist = {
    'n_estimators': randint(100, 1000),
    'max_depth': randint(3, 15),
    'learning_rate': loguniform(0.01, 0.3),
}
rand = RandomizedSearchCV(model, param_dist, n_iter=100, cv=5, 
                          scoring='roc_auc', n_jobs=-1, random_state=42)

# Successive Halving (fast)
halving = HalvingGridSearchCV(model, param_grid, cv=5, 
                              factor=3, min_resources=50, max_resources='auto')

# Access results
print(grid.best_params_)
print(grid.best_score_)
print(grid.cv_results_['mean_test_score'])
```

#### Preprocessing
```python
from sklearn.preprocessing import (
    StandardScaler, MinMaxScaler, RobustScaler,
    OneHotEncoder, OrdinalEncoder, LabelEncoder,
    PolynomialFeatures, KBinsDiscretizer
)
from sklearn.compose import ColumnTransformer
from sklearn.impute import SimpleImputer, KNNImputer, IterativeImputer

# ColumnTransformer (different preprocessing per column)
numeric_features = ['age', 'income']
categorical_features = ['city', 'gender']

preprocessor = ColumnTransformer([
    ('num', Pipeline([
        ('imputer', SimpleImputer(strategy='median')),
        ('scaler', StandardScaler())
    ]), numeric_features),
    ('cat', Pipeline([
        ('imputer', SimpleImputer(strategy='constant', fill_value='missing')),
        ('encoder', OneHotEncoder(handle_unknown='ignore', sparse_output=False))
    ]), categorical_features)
])

# Full pipeline
pipe = Pipeline([
    ('preprocessor', preprocessor),
    ('model', LogisticRegression())
])
```

#### Model Evaluation
```python
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score, f1_score,
    roc_auc_score, average_precision_score,
    precision_recall_curve, roc_curve,
    confusion_matrix, classification_report,
    mean_squared_error, mean_absolute_error, r2_score
)

# Classification
y_pred = model.predict(X_test)
y_prob = model.predict_proba(X_test)[:, 1]

print(classification_report(y_test, y_pred))
print(f"AUC: {roc_auc_score(y_test, y_prob):.4f}")
print(f"AP: {average_precision_score(y_test, y_prob):.4f}")

# Confusion matrix
cm = confusion_matrix(y_test, y_pred)
sns.heatmap(cm, annot=True, fmt='d')

# Regression
y_pred = model.predict(X_test)
print(f"MSE: {mean_squared_error(y_test, y_pred):.4f}")
print(f"MAE: {mean_absolute_error(y_test, y_pred):.4f}")
print(f"R²: {r2_score(y_test, y_pred):.4f}")

# Custom scorer
from sklearn.metrics import make_scorer
def custom_metric(y_true, y_pred):
    return f1_score(y_true, y_pred, average='macro')
custom_scorer = make_scorer(custom_metric, greater_is_better=True)
```

---

## PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Vectorized NumPy** | Python loops on arrays | 100x slower |
| **Pandas vectorized ops** | `apply` with lambda row-wise | Slow, not vectorized |
| **Method chaining** | Intermediate variables | Hard to debug, memory |
| **Pipeline + ColumnTransformer** | Manual preprocessing | Leakage, inconsistency |
| **Type hints + dataclasses** | Dict/args everywhere | Bugs, poor IDE support |
| **Structured logging** | Print statements | Unsearchable, no structure |
| **Virtual environments** | Global pip install | Dependency conflicts |
| **Requirements.txt / pyproject.toml** | No dependency management | Unreproducible |
| **Type hints + mypy** | No type checking | Runtime errors in production |

---

## INTERVIEW QUESTIONS (Top 15)

### 1. **NumPy**: "Explain broadcasting rules with an example."

```
Broadcasting aligns shapes from RIGHT to LEFT:
(3, 4) + (4,)     → (3, 4) + (1, 4) → (3, 4)  ✓
(3, 1) + (4,)     → (3, 1) + (1, 4) → (3, 4)  ✓
(3, 4) + (3,)     → (3, 4) + (3, 1) → Error!  ✗ (aligned right: 4 vs 3)

RULES:
1. Pad shorter shape with 1s on LEFT
2. Dimensions must match or be 1
3. Result shape = max of each dimension
```

### 2. **NumPy**: "Vectorize this loop: `for i in range(len(x)): y[i] = x[i] * 2 if x[i] > 0 else 0`"

```python
# Vectorized
y = np.where(x > 0, x * 2, 0)
# Or
y = np.maximum(x, 0) * 2
```

### 3. **Pandas**: "How do you handle missing data in a DataFrame?"

```
1. DETECT: df.isnull().sum(), df.isnull().mean()
2. STRATEGY:
   - Drop: df.dropna(thresh=3, subset=['col1', 'col2'])
   - Fill constant: df.fillna(0)
   - Fill statistic: df.fillna(df.mean()/median()/mode())
   - Forward/backward: df.fillna(method='ffill'/'bfill')
   - Interpolate: df.interpolate(method='linear'/'time')
   - Model-based: IterativeImputer, KNNImputer
3. ALWAYS: Fit imputer on TRAIN only, transform TEST
```

### 4. **Pandas**: "Explain `groupby` with `transform` vs `agg`."

```
AGG: Reduces groups to single value per group
    df.groupby('cat')['val'].agg('mean')  # Returns Series (n_groups,)

TRANSFORM: Broadcasts result back to original shape
    df['group_mean'] = df.groupby('cat')['val'].transform('mean')
    # Returns Series (n_rows,) - same length as df
    # Useful for: z-scores, centering, flagging outliers
```

### 5. **Matplotlib/Seaborn**: "How do you visualize a correlation matrix?"

```python
corr = df.corr()
mask = np.triu(np.ones_like(corr, dtype=bool))  # Upper triangle mask
sns.heatmap(corr, mask=mask, annot=True, cmap='RdBu_r', 
            center=0, square=True, fmt='.2f',
            cbar_kws={'shrink': 0.8})
plt.title('Correlation Matrix')
```

### 6. **Scikit-learn**: "Explain the Pipeline API and why it prevents data leakage."

```
Pipeline chains transformers + estimator:
Pipe = Pipeline([('scaler', StandardScaler()), ('pca', PCA()), ('clf', LogisticRegression())])

LEAKAGE PREVENTION:
- In CV: scaler.fit() on TRAIN fold only, transform both train/val
- Without pipeline: scaler.fit(X_all) → transforms both → leakage
- Pipeline ensures: fit_transform on train, transform on val/test
- ColumnTransformer applies different preprocessing per column
```

### 7. **Scikit-learn**: "How do you do hyperparameter tuning efficiently?"

```
1. RANDOM SEARCH > GRID SEARCH (Bergstra & Bengio 2012)
   - Sample from distributions, not grid
   - More efficient for high-dimensional spaces

2. SUCCESSIVE HALVING (HalvingGridSearchCV):
   - Start with many configs, few resources
   - Iteratively halve configs, double resources
   - 5-10x faster than GridSearch

3. BAYESIAN OPTIMIZATION (Optuna):
   - Models p(score|params) with GP/TPE
   - Sequential, sample promising regions
   - Best for expensive models

4. KEY PRACTICES:
   - Nested CV for unbiased estimate
   - StratifiedKFold for classification
   - TimeSeriesSplit for time series
   - GroupKFold for grouped data
   - Nested CV for unbiased performance estimate
```

### 8. **Scikit-learn**: "How do you handle categorical features?"

```
OPTIONS:
1. OneHotEncoder(handle_unknown='ignore', sparse_output=False)
   - Good for low cardinality (<10)
   - Creates sparse matrix → dense if needed

2. OrdinalEncoder (for tree models)
   - Assigns integers, preserves order
   - handle_unknown='use_encoded_value', unknown_value=-1

3. TargetEncoder (category_encoders)
   - Replace category with target mean
   - Smooth with global mean (regularization)
   - Risk: target leakage if not in CV loop!

4. Embeddings (Neural nets)
   - Learn dense vectors
   - Good for high cardinality

COLUMNTRANSFORMER PATTERN:
preprocessor = ColumnTransformer([
    ('num', StandardScaler(), numeric_cols),
    ('cat', OneHotEncoder(handle_unknown='ignore'), cat_cols)
])
```

### 8. **Visualization**: "How do you detect outliers visually?"

```python
# Boxplot
sns.boxplot(x='category', y='value', data=df)

# Violin plot (distribution + boxplot)
sns.violinplot(x='category', y='value', data=df)

# Scatter with outlier highlight
sns.scatterplot(x='x', y='y', hue='outlier', data=df)

# Pairplot with outliers
sns.pairplot(df, hue='outlier', diag_kind='kde')

# Statistical
from scipy import stats
z_scores = np.abs(stats.zscore(df['value']))
outliers = df[z_scores > 3]

# IQR method
Q1, Q3 = df['value'].quantile([0.25, 0.75])
IQR = Q3 - Q1
outliers = df[(df['value'] < Q1 - 1.5*IQR) | (df['value'] > Q3 + 1.5*IQR)]
```

### 9. **Scikit-learn**: "How do you evaluate a classifier properly?"

```
METRICS BY PROBLEM:
- Balanced: Accuracy, F1
- Imbalanced: Precision, Recall, F1, AUC-ROC, AUC-PR
- Ranking: AUC-ROC, AUC-PR, NDCG
- Calibration: Brier score, reliability diagram

VALIDATION:
- StratifiedKFold (classification)
- GroupKFold (grouped data)
- TimeSeriesSplit (time series)
- Nested CV for hyperparameter tuning

METRICS:
- Classification report (precision, recall, f1 per class)
- Confusion matrix
- ROC curve + AUC
- Precision-Recall curve + AUC
- Confusion matrix
- Calibration curve (reliability diagram)
```

### 9. **Pandas**: "How do you merge DataFrames efficiently?"

```python
# MERGE TYPES
pd.merge(left, right, on='key', how='inner')  # inner, left, right, outer

# KEY OPTIONS
# how='left'   → keep all left rows
# how='right'  → keep all right rows
# how='outer'  → keep all rows
# how='inner'  → only matching rows

# DIFFERENT COLUMN NAMES
pd.merge(left, right, left_on='id_left', right_on='id_right')

# MULTIPLE KEYS
pd.merge(left, right, on=['user_id', 'date'])

# JOIN ON INDEX
df1.join(df2, on='key')       # df1 index, df2 column
df1.join(df2, lsuffix='_l', rsuffix='_r')

# CONCAT
pd.concat([df1, df2], axis=0)   # Stack rows
pd.concat([df1, df2], axis=1)   # Side by side

# PERFORMANCE
# - Use 'how='left'' with smaller right df
# - Ensure key columns have same dtype
# - Sort keys before merge for large DataFrames
```

### 10. **NumPy**: "How do you compute pairwise distances efficiently?"

```python
# Scipy (optimized)
from scipy.spatial.distance import pdist, squareform, cdist
# Pairwise within X
dists = pdist(X, metric='euclidean')
dist_matrix = squareform(dists)

# Between X and Y
dists = cdist(X, Y, metric='cosine')

# NumPy broadcasting (small-medium arrays)
diff = X[:, np.newaxis, :] - Y[np.newaxis, :, :]  # (n, m, d)
dists = np.sqrt(np.sum(diff**2, axis=2))  # (n, m)

# For large arrays: use FAISS, Annoy, HNSW (approximate)
import faiss
index = faiss.IndexFlatL2(d)
index.add(X.astype(np.float32))
distances, indices = index.search(Y.astype(np.float32), k=10)
```

### 11. **Pandas**: "How do you optimize memory usage?"

```python
# 1. Downcast numerics
df['int_col'] = df['int_col'].astype('int32')  # or int16, int8
df['float_col'] = df['float_col'].astype('float32')

# 2. Categorical for low-cardinality strings
df['category'] = df['category'].astype('category')

# 3. Sparse arrays for sparse data
from pandas.arrays import SparseArray
df['sparse_col'] = pd.arrays.SparseArray(df['sparse_col'], fill_value=0)

# 4. Chunked reading
chunks = pd.read_csv('large.csv', chunksize=100000)
for chunk in chunks:
    process(chunk)

# 5. Parquet (columnar, compressed)
df.to_parquet('file.parquet', compression='snappy')
df = pd.read_parquet('file.parquet')

# Memory check
df.memory_usage(deep=True).sum() / 1024**2  # MB
```

### 11. **Scikit-learn**: "How do you handle class imbalance?"

```
TECHNIQUES (in order of preference):

1. CLASS WEIGHTS (easiest, no data change)
   LogisticRegression(class_weight='balanced')
   XGBClassifier(scale_pos_weight=neg/pos)
   RandomForestClassifier(class_weight='balanced')

2. THRESHOLD TUNING
   - Find optimal threshold on validation (max F1, or business cost)
   - predict_proba > threshold

3. ENSEMBLE METHODS
   BalancedRandomForestClassifier (imbalanced-learn)
   EasyEnsembleClassifier

4. RESAMPLING (in pipeline, inside CV!)
   SMOTE, ADASYN (oversample minority)
   RandomUnderSampler, TomekLinks (undersample majority)
   SMOTE + Tomek (hybrid)

4. ANOMALY DETECTION FRAMING
   OneClassSVM, IsolationForest (minority as anomaly)

5. ENSEMBLE OF BALANCED BAGS
   Each tree on balanced bootstrap sample

EVALUATION METRICS FOR IMBALANCE:
- NEVER use Accuracy
- AUC-ROC (threshold independent)
- AUC-PR (better for extreme imbalance)
- F1, Precision@K, Recall@K
- Matthews Correlation Coefficient (MCC)
```

### 11. **Statistics**: "What's the difference between StandardScaler and RobustScaler?"

```
STANDARDSCALER:
    (x - mean) / std
    - Sensitive to outliers (mean/std affected)
    - Assumes Gaussian-like distribution
    - Good for: Linear models, PCA, KNN, Neural nets

ROBUSTSCALER:
    (x - median) / IQR
    - Robust to outliers (median/IQR robust)
    - Doesn't assume normality
    - Good for: Skewed data, outliers present

MINMAXSCALER:
    (x - min) / (max - min) → [0, 1]
    - Sensitive to outliers
    - Good for: Neural nets, images, bounded features

MAXABSSCALER:
    x / max(|x|)
    - Preserves sparsity
    - Good for: Sparse data, features already centered

POWERTRANSFORMER (Yeo-Johnson/Box-Cox):
    - Makes distribution more Gaussian
    - Good for: Linear models, reducing skew
```

### 12. **General**: "How do you ensure reproducibility in ML experiments?"

```python
# 1. Set all seeds
import random, os
import numpy as np
import torch

def set_seed(seed=42):
    random.seed(seed)
    os.environ['PYTHONHASHSEED'] = str(seed)
    np.random.seed(seed)
    torch.manual_seed(seed)
    torch.cuda.manual_seed_all(seed)
    torch.backends.cudnn.deterministic = True
    torch.backends.cudnn.benchmark = False

# 2. Environment management
# requirements.txt or pyproject.toml
# conda env export > environment.yml

# 3. Version control
# Git for code, DVC for data/models

# 4. Experiment tracking
import mlflow
mlflow.start_run()
mlflow.log_params(params)
mlflow.log_metrics(metrics)
mlflow.sklearn.log_model(model, "model")

# 5. Configuration management
# Hydra, OmegaConf, or simple YAML configs
# Separate config from code

# 5. Containerize
# Dockerfile with pinned versions
```

---

## QUICK REFERENCE: LIBRARY CHEAT SHEET

### NumPy Array Creation
| Function | Use Case |
|----------|----------|
| `np.array()` | From list |
| `np.zeros/ones/empty` | Preallocate |
| `np.arange/ linspace` | Sequences |
| `np.random.*` | Random data |
| `np.eye/identity` | Identity matrix |
| `np.diag` | Diagonal matrix |

### Pandas Essential Operations
| Operation | Code |
|-----------|------|
| Read CSV | `pd.read_csv('file.csv', parse_dates=['date'])` |
| Filter rows | `df[df['col'] > 5]` or `df.query('col > 5')` |
| Select columns | `df[['a', 'b']]` or `df.loc[:, 'a':'c']` |
| GroupBy agg | `df.groupby('cat').agg({'val': ['sum', 'mean']})` |
| Pivot | `df.pivot_table(index='cat', columns='sub', values='val', aggfunc='mean')` |
| Merge | `pd.merge(left, right, on='key', how='left')` |
| Datetime | `pd.to_datetime(df['col'])` + `.dt.year/.month/.day` |
| Rolling | `df.rolling(7).mean()` |
| Shift | `df['lag1'] = df['val'].shift(1)` |

### Scikit-learn Pipeline Template
```python
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.impute import SimpleImputer
from sklearn.model_selection import cross_val_score

numeric_pipe = Pipeline([
    ('imputer', SimpleImputer(strategy='median')),
    ('scaler', StandardScaler())
])
categorical_pipe = Pipeline([
    ('imputer', SimpleImputer(strategy='constant', fill_value='missing')),
    ('encoder', OneHotEncoder(handle_unknown='ignore', sparse_output=False))
])

preprocessor = ColumnTransformer([
    ('num', numeric_pipe, numeric_cols),
    ('cat', categorical_pipe, categorical_cols)
])

pipeline = Pipeline([
    ('preprocessor', preprocessor),
    ('model', LogisticRegression())
])

scores = cross_val_score(pipeline, X, y, cv=5, scoring='roc_auc')
```

---

*This document covers all topics from the Essential Libraries section of the roadmap.sh Machine Learning roadmap with exam-style depth.*