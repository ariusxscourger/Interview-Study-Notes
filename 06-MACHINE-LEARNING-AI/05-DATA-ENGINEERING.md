# DATA ENGINEERING FOR MACHINE LEARNING
## Exam-Style Study Notes

**Roadmap Section:** Data Sources, Collection, Formats, Cleaning, Preprocessing, Feature Engineering
**Prerequisites:** Python, SQL, basic statistics, Pandas/NumPy

---

## 1. DATA SOURCES & COLLECTION

### WHY? (Problem Statement)
**Garbage in, garbage out.** ML models are only as good as their training data. Understanding data sources, collection methods, and biases is critical for building reliable models.

### HOW? (Data Source Types & Collection Methods)

#### Data Source Categories
| Source Type | Examples | Characteristics | ML Challenges |
|-------------|----------|-----------------|---------------|
| **Structured** | Databases, CSVs, Parquet | Schema-defined, queryable | Schema drift, missing values |
| **Semi-structured** | JSON, XML, logs, API responses | Hierarchical, flexible schema | Parsing, normalization |
| **Unstructured** | Text, images, audio, video | No predefined schema | Feature extraction, storage |
| **Streaming** | Kafka, Kinesis, IoT sensors | Real-time, unbounded | Latency, exactly-once, ordering |
| **External/Third-party** | APIs, web scraping, data vendors | Limited control, rate limits | Reliability, schema changes, cost |

#### Collection Methods
| Method | Use Case | Tools | Challenges |
|--------|----------|-------|------------|
| **Batch ETL** | Daily/weekly batch loads | Airflow, dbt, Spark | Latency, schema evolution |
| **Streaming ETL** | Real-time features | Kafka Connect, Flink, Spark Streaming | Exactly-once, late data |
| **CDC (Change Data Capture)** | Database replication | Debezium, Maxwell | Schema changes, latency |
| **API Polling** | Third-party data | Airbyte, Fivetran, custom | Rate limits, pagination |
| **Web Scraping** | Public web data | Scrapy, Playwright, Selenium | Anti-bot, legal, structure changes |
| **Manual/Annotated** | Labels, ground truth | Label Studio, Prodigy, CVAT | Cost, quality, consistency |

#### Data Contracts (Schema Registry)
```python
# Schema definition (Pydantic/Avro/Protobuf)
from pydantic import BaseModel, Field
from typing import Optional
from datetime import datetime

class UserEvent(BaseModel):
    event_id: str = Field(..., description="Unique event ID")
    user_id: str = Field(..., min_length=1)
    event_type: str = Field(..., pattern="^(click|purchase|view)$")
    timestamp: datetime
    value: Optional[float] = Field(None, ge=0)
    metadata: dict = Field(default_factory=dict)

# Avro schema (for Kafka)
{
  "type": "record",
  "name": "UserEvent",
  "fields": [
    {"name": "event_id", "type": "string"},
    {"name": "user_id", "type": "string"},
    {"name": "event_type", "type": {"type": "enum", "name": "EventType", "symbols": ["click", "purchase", "view"]}},
    {"name": "timestamp", "type": {"type": "long", "logicalType": "timestamp-millis"}},
    {"name": "value", "type": ["null", "double"], "default": null}
  ]
}
```

---

## 2. DATA FORMATS

### WHY? (Problem Statement)
**Format choice affects:** storage cost, query performance, schema evolution, interoperability, tool compatibility.

### HOW? (Format Comparison)

| Format | Type | Compression | Schema | Query | Best For |
|--------|------|-------------|--------|-------|----------|
| **CSV** | Row-based | None/gzip | None | Full scan | Exchange, small data |
| **JSON** | Row-based | None/gzip | Implicit | Full scan | APIs, nested data |
| **Parquet** | Columnar | Snappy/Zstd | Embedded | Predicate pushdown | Analytics, ML features |
| **ORC** | Columnar | Zlib/Snappy | Embedded | Predicate pushdown | Hive, Spark |
| **Avro** | Row-based | Snappy/Deflate | Embedded | Full scan | Kafka, serialization |
| **Protobuf** | Row-based | None | .proto | Full scan | gRPC, low latency |
| **Feather** | Columnar | LZ4/Zstd | Embedded | Fast read | Python/R interchange |
| **Delta Lake** | Columnar | - | Embedded | ACID, time travel | Lakehouse |
| **Iceberg** | Columnar | - | Embedded | ACID, time travel | Lakehouse |

#### Parquet Deep Dive (ML Standard)
```python
import pandas as pd
import pyarrow.parquet as pq
import pyarrow as pa

# Write with compression
df.to_parquet('data.parquet', 
              compression='snappy',  # snappy, gzip, brotli, zstd
              partition_cols=['date', 'category'],  # Partitioning
              index=False)

# Read with predicate pushdown (only reads needed partitions/columns)
df = pd.read_parquet('data.parquet', 
                     columns=['user_id', 'feature_1', 'target'],
                     filters=[('date', '>=', '2023-01-01'), ('category', 'in', ['A', 'B'])])

# PyArrow for advanced operations
table = pq.read_table('data.parquet')
# Schema evolution
new_schema = table.schema.append(pa.field('new_feature', pa.float64()))
```

#### Schema Evolution (Parquet/Avro/Iceberg)
| Change | Parquet | Avro | Iceberg/Delta |
|--------|---------|------|---------------|
| Add column (end) | ✓ | ✓ | ✓ |
| Add column (middle) | ✗ | ✓ | ✓ |
| Remove column | ✗ | ✗ | ✓ |
| Rename column | ✗ | ✗ | ✓ |
| Change type (compatible) | ✓ | ✓ | ✓ |
| Change type (incompatible) | ✗ | ✗ | ✗ |
| Reorder columns | ✗ | ✓ | ✓ |

---

## 3. DATA CLEANING

### WHY? (Problem Statement)
**Real-world data is messy.** Cleaning transforms raw data into analysis-ready format while preserving signal and removing noise.

### HOW? (Cleaning Pipeline)

#### Missing Value Patterns
```python
import pandas as pd
import numpy as np

# Missingness types
# MCAR: Missing Completely at Random (no pattern)
# MAR: Missing at Random (depends on observed data)
# MNAR: Missing Not at Random (depends on missing value itself)

# Detection
df.isnull().sum()           # Count
df.isnull().mean()          # Percentage
df.isnull().any(axis=1)     # Rows with any missing

# Visualization
import missingno as msno
msno.matrix(df)             # Missingness matrix
msno.heatmap(df)            # Correlation of missingness
msno.dendrogram(df)         # Clustering of missingness patterns
```

#### Imputation Strategies
```python
from sklearn.impute import SimpleImputer, KNNImputer, IterativeImputer
from sklearn.experimental import enable_iterative_imputer

# 1. Simple Imputer
imp = SimpleImputer(strategy='median')  # mean, median, most_frequent, constant
X_filled = imp.fit_transform(X_train)
X_test_filled = imp.transform(X_test)  # Only transform test!

# 2. KNN Imputer (uses similar rows)
imp = KNNImputer(n_neighbors=5, weights='uniform')
X_filled = imp.fit_transform(X_train)

# 3. Iterative Imputer (MICE - Multiple Imputation by Chained Equations)
imp = IterativeImputer(max_iter=10, random_state=42)
X_filled = imp.fit_transform(X_train)

# 4. Domain-specific
# Time series: forward/backward fill, interpolation
df['value'] = df['value'].interpolate(method='time')
df['value'] = df['value'].ffill().bfill()

# Categorical: mode, 'missing' category
df['category'] = df['category'].fillna('unknown')
df['category'] = df['category'].fillna(df['category'].mode()[0])
```

#### Outlier Detection & Handling
```python
import numpy as np
from scipy import stats
from sklearn.ensemble import IsolationForest
from sklearn.neighbors import LocalOutlierFactor

# 1. Z-score (assumes normality)
z_scores = np.abs(stats.zscore(df['value']))
outliers_z = df[z_scores > 3]

# 2. Modified Z-score (robust)
median = df['value'].median()
mad = stats.median_abs_deviation(df['value'])
modified_z = 0.6745 * (df['value'] - median) / mad
outliers_mz = df[modified_z > 3.5]

# 3. IQR method (non-parametric)
Q1, Q3 = df['value'].quantile([0.25, 0.75])
IQR = Q3 - Q1
outliers_iqr = df[(df['value'] < Q1 - 1.5*IQR) | (df['value'] > Q3 + 1.5*IQR)]

# 4. Isolation Forest (multivariate, unsupervised)
iso = IsolationForest(contamination=0.01, random_state=42)
outliers_iso = iso.fit_predict(df[['feat1', 'feat2', 'feat3']]) == -1

# 5. Local Outlier Factor (density-based)
lof = LocalOutlierFactor(n_neighbors=20, contamination=0.01)
outliers_lof = lof.fit_predict(df[['feat1', 'feat2']]) == -1

# Handling
# Option 1: Remove
df_clean = df[~outliers]

# Option 2: Cap (winsorize)
from scipy.stats.mstats import winsorize
df['value_winsorized'] = winsorize(df['value'], limits=[0.01, 0.01])

# Option 3: Flag (create feature)
df['is_outlier'] = outliers_z.astype(int)

# Option 4: Transform (log, sqrt)
df['value_log'] = np.log1p(df['value'])
```

#### Data Type Issues
```python
# Inconsistent formats
df['date'] = pd.to_datetime(df['date'], errors='coerce', infer_datetime_format=True)
df['price'] = pd.to_numeric(df['price'].str.replace('[$,]', '', regex=True), errors='coerce')

# Categorical standardization
df['category'] = df['category'].str.lower().str.strip()
df['category'] = df['category'].replace({'nyc': 'new_york', 'la': 'los_angeles'})

# Text cleaning
df['text'] = df['text'].str.lower()
df['text'] = df['text'].str.replace(r'[^\w\s]', '', regex=True)
df['text'] = df['text'].str.strip()
```

---

## 4. PREPROCESSING TECHNIQUES

### WHY? (Problem Statement)
**Raw features ≠ model-ready features.** Preprocessing transforms raw data into format suitable for ML algorithms.

### HOW? (Preprocessing Pipeline)

#### Scaling & Normalization
```python
from sklearn.preprocessing import (
    StandardScaler, MinMaxScaler, RobustScaler, 
    MaxAbsScaler, PowerTransformer, QuantileTransformer
)

# StandardScaler: (x - μ) / σ
# Best for: Linear models, PCA, KNN, Neural nets, Gaussian-like
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X_train)

# MinMaxScaler: (x - min) / (max - min) → [0, 1]
# Best for: Neural nets, images, bounded features
scaler = MinMaxScaler()

# RobustScaler: (x - median) / IQR
# Best for: Outliers present, skewed data
scaler = RobustScaler()

# PowerTransformer (Yeo-Johnson / Box-Cox)
# Makes distribution more Gaussian
pt = PowerTransformer(method='yeo-johnson')  # handles negative
X_pt = pt.fit_transform(X_train)

# QuantileTransformer (non-parametric)
# Maps to uniform/normal distribution
qt = QuantileTransformer(n_quantiles=1000, output_distribution='normal')
X_qt = qt.fit_transform(X_train)
```

#### Categorical Encoding
```python
from sklearn.preprocessing import (
    OneHotEncoder, OrdinalEncoder, LabelEncoder, TargetEncoder
)
from category_encoders import TargetEncoder, CatBoostEncoder, LeaveOneOutEncoder

# 1. OneHotEncoder (low cardinality)
ohe = OneHotEncoder(handle_unknown='ignore', sparse_output=False, drop='first')
X_ohe = ohe.fit_transform(X_train[['category']])

# 2. OrdinalEncoder (tree models, ordinal relationship)
oe = OrdinalEncoder(handle_unknown='use_encoded_value', unknown_value=-1)
X_ord = oe.fit_transform(X_train[['category']])

# 3. TargetEncoder (high cardinality, linear models)
# Risk: Target leakage! Must fit within CV loop
te = TargetEncoder(smoothing=10, min_samples_leaf=20)
X_te = te.fit_transform(X_train['category'], y_train)

# 3b. LeaveOneOutEncoder (reduces leakage)
loe = LeaveOneOutEncoder()
X_loe = loe.fit_transform(X_train['category'], y_train)

# 4. Feature Hashing (fixed dimension, high cardinality)
from sklearn.feature_extraction import FeatureHasher
hasher = FeatureHasher(n_features=256, input_type='string')
X_hashed = hasher.transform(df['category'].apply(lambda x: [x]))

# 5. Embeddings (learned, for NNs)
# nn.Embedding(num_categories, embedding_dim)
```

#### Text Feature Extraction
```python
from sklearn.feature_extraction.text import (
    CountVectorizer, TfidfVectorizer, HashingVectorizer
)

# CountVectorizer (Bag of Words)
cv = CountVectorizer(
    max_features=10000,
    ngram_range=(1, 2),      # unigrams + bigrams
    stop_words='english',
    min_df=2,                # ignore rare terms
    max_df=0.95              # ignore too common
)
X_counts = cv.fit_transform(texts)

# TF-IDF (weighted by inverse document frequency)
tfidf = TfidfVectorizer(
    max_features=10000,
    ngram_range=(1, 2),
    sublinear_tf=True        # 1 + log(tf)
)
X_tfidf = tfidf.fit_transform(texts)

# HashingVectorizer (fixed size, no vocabulary)
hv = HashingVectorizer(n_features=2**18, alternate_sign=False)
X_hash = hv.transform(texts)

# For Deep Learning: Tokenizers
from transformers import AutoTokenizer
tokenizer = AutoTokenizer.from_pretrained('bert-base-uncased')
encodings = tokenizer(texts, truncation=True, padding=True, max_length=512, return_tensors='pt')
```

#### Date/Time Features
```python
def extract_datetime_features(df, col='timestamp'):
    df = df.copy()
    dt = pd.to_datetime(df[col])
    df[f'{col}_year'] = dt.dt.year
    df[f'{col}_month'] = dt.dt.month
    df[f'{col}_day'] = dt.dt.day
    df[f'{col}_dayofweek'] = df[col].dt.dayofweek
    df[f'{col}_hour'] = dt.dt.hour
    df[f'{col}_is_weekend'] = (dt.dt.dayofweek >= 5).astype(int)
    df[f'{col}_is_month_start'] = dt.dt.is_month_start.astype(int)
    df[f'{col}_is_month_end'] = dt.dt.is_month_end.astype(int)
    df[f'{col}_quarter'] = dt.dt.quarter
    # Cyclical encoding (preserve periodicity)
    df[f'{col}_sin_hour'] = np.sin(2 * np.pi * dt.dt.hour / 24)
    df[f'{col}_cos_hour'] = np.cos(2 * np.pi * dt.dt.hour / 24)
    df[f'{col}_sin_month'] = np.sin(2 * np.pi * dt.dt.month / 12)
    df[f'{col}_cos_month'] = np.cos(2 * np.pi * dt.dt.month / 12)
    return df
```

---

## 5. FEATURE ENGINEERING

### WHY? (Problem Statement)
**Features = Model Performance.** Good features can make simple models outperform complex ones with raw features.

### HOW? (Feature Engineering Patterns)

#### Numerical Features
```python
import numpy as np
import pandas as pd

# 1. Mathematical transformations
df['log_value'] = np.log1p(df['value'])           # Log transform (reduce skew)
df['sqrt_value'] = np.sqrt(df['value'])           # Square root
df['value_squared'] = df['value'] ** 2            # Polynomial
df['value_cubed'] = df['value'] ** 3

# 2. Binning/Discretization
df['value_bin'] = pd.cut(df['value'], bins=10, labels=False)      # Equal width
df['value_qcut'] = pd.qcut(df['value'], q=10, labels=False, duplicates='drop')  # Quantile
from sklearn.preprocessing import KBinsDiscretizer
kb = KBinsDiscretizer(n_bins=10, encode='ordinal', strategy='quantile')

# 3. Interaction features
df['feat_ratio'] = df['feat_a'] / (df['feat_b'] + 1e-8)
df['feat_sum'] = df['feat_a'] + df['feat_b']
df['feat_diff'] = df['feat_a'] - df['feat_b']
df['feat_prod'] = df['feat_a'] * df['feat_b']

# 3b. Polynomial features
from sklearn.preprocessing import PolynomialFeatures
poly = PolynomialFeatures(degree=2, include_bias=False, interaction_only=True)
X_poly = poly.fit_transform(X[['feat_a', 'feat_b']])

# 3c. Domain-specific ratios
df['ctr'] = df['clicks'] / (df['impressions'] + 1)
df['conversion_rate'] = df['purchases'] / (df['clicks'] + 1)
df['avg_order_value'] = df['revenue'] / (df['orders'] + 1)
```

#### Categorical Features
```python
# 1. Frequency/Count encoding
freq_map = df['category'].value_counts(normalize=True)
df['cat_freq'] = df['category'].map(freq_map)

# 2. Target encoding (with smoothing, CV-safe)
from category_encoders import TargetEncoder
te = TargetEncoder(smoothing=10, min_samples_leaf=20)
df['cat_target'] = te.fit_transform(df['category'], y)

# 3. Embeddings (pre-trained or learned)
# Entity embeddings for high cardinality
# entity_emb = nn.Embedding(num_categories, embedding_dim)

# 4. Feature hashing (fixed size)
from sklearn.feature_extraction import FeatureHasher
hasher = FeatureHasher(n_features=64, input_type='string')
X_hashed = hasher.transform(df['category'].apply(lambda x: [x]))

# 5. Binary encoding (for tree models)
df = pd.get_dummies(df, columns=['category'], prefix='cat', drop_first=True)
```

#### Aggregation Features (Time Series / User Behavior)
```python
def create_aggregation_features(df, group_col, target_col, windows=['7D', '30D']):
    """Create rolling window features"""
    df = df.sort_values('timestamp')
    df = df.set_index('timestamp')
    
    for window in windows:
        # Rolling statistics
        rolled = df.groupby('user_id')[target_col].rolling(window)
        df[f'{target_col}_mean_{window}'] = rolled.mean().reset_index(level=0, drop=True)
        df[f'{target_col}_std_{window}'] = rolled.std().reset_index(level=0, drop=True)
        df[f'{target_col}_min_{window}'] = rolled.min().reset_index(level=0, drop=True)
        df[f'{target_col}_max_{window}'] = rolled.max().reset_index(level=0, drop=True)
        df[f'{target_col}_count_{window}'] = rolled.count().reset_index(level=0, drop=True)
    
    return df.reset_index()

# Expanding window (cumulative)
df['cumsum'] = df.groupby('user_id')['value'].expanding().sum().reset_index(level=0, drop=True)
df['cumcount'] = df.groupby('user_id').cumcount() + 1
df['cummean'] = df.groupby('user_id')['value'].expanding().mean().reset_index(level=0, drop=True)

# Lag features
for lag in [1, 7, 30]:
    df[f'lag_{lag}'] = df.groupby('user_id')['value'].shift(lag)
```

#### Text Features (NLP)
```python
# Basic stats
df['text_length'] = df['text'].str.len()
df['word_count'] = df['text'].str.split().str.len()
df['char_count'] = df['text'].str.len()
df['avg_word_length'] = df['text'].str.len() / df['text'].str.split().str.len()
df['caps_ratio'] = df['text'].str.count(r'[A-Z]') / df['text'].str.len()
df['punct_count'] = df['text'].str.count(r'[^\w\s]')

# Readability
import textstat
df['flesch_reading_ease'] = df['text'].apply(textstat.flesch_reading_ease)
df['gunning_fog'] = df['text'].apply(textstat.gunning_fog)

# Sentiment
from textblob import TextBlob
df['polarity'] = df['text'].apply(lambda x: TextBlob(x).sentiment.polarity)
df['subjectivity'] = df['text'].apply(lambda x: TextBlob(x).sentiment.subjectivity)

# TF-IDF features (as dense features)
tfidf = TfidfVectorizer(max_features=100, ngram_range=(1,2))
tfidf_features = tfidf.fit_transform(df['text'])
tfidf_df = pd.DataFrame(tfidf_features.toarray(), 
                         columns=[f'tfidf_{i}' for i in range(100)])
df = pd.concat([df, tfidf_df], axis=1)
```

#### Image Features (Computer Vision)
```python
# Traditional features
import cv2
from skimage.feature import hog, local_binary_pattern

# HOG features
hog_features = hog(image, orientations=9, pixels_per_cell=(8, 8), 
                   cells_per_block=(2, 2), visualize=False)

# Color histograms
hist = cv2.calcHist([image], [0,1,2], None, [8,8,8], [0,256,0,256,0,256])
hist = cv2.normalize(hist, hist).flatten()

# Deep features (pre-trained CNN)
import torchvision.models as models
import torchvision.transforms as transforms

model = models.resnet50(pretrained=True)
model = torch.nn.Sequential(*list(model.children())[:-1])  # Remove final layer
model.eval()

transform = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
)

def extract_features(image_path):
    img = Image.open(image_path).convert('RGB')
    img = transform(img).unsqueeze(0)
    with torch.no_grad():
        features = model(img)
    return features.squeeze().numpy()
```

---

## 6. FEATURE SELECTION

### WHY? (Problem Statement)
**More features ≠ better models.** Irrelevant features add noise, increase overfitting, slow training, reduce interpretability.

### HOW? (Selection Methods)

#### Filter Methods (Model-agnostic)
```python
from sklearn.feature_selection import (
    SelectKBest, SelectPercentile, f_classif, f_regression,
    mutual_info_classif, mutual_info_regression, VarianceThreshold
)
from sklearn.feature_selection import SelectFromModel

# 1. Variance Threshold (remove constant/near-constant)
vt = VarianceThreshold(threshold=0.01)
X_reduced = vt.fit_transform(X)

# 2. Univariate Statistical Tests
# Classification: ANOVA F-value, Mutual Information
selector = SelectKBest(f_classif, k=50)
selector = SelectKBest(mutual_info_classif, k=50)
X_selected = selector.fit_transform(X, y)

# Regression: F-regression, Mutual Information
selector = SelectKBest(f_regression, k=50)
selector = SelectKBest(mutual_info_regression, k=50)

# 3. Correlation-based (remove redundant)
corr_matrix = X.corr().abs()
upper = corr_matrix.where(np.triu(np.ones(corr_matrix.shape), k=1).astype(bool))
to_drop = [col for col in upper.columns if any(upper[col] > 0.95)]
X_reduced = X.drop(columns=to_drop)
```

#### Wrapper Methods (Model-based)
```python
from sklearn.feature_selection import RFE, RFECV, SelectFromModel

# 1. Recursive Feature Elimination (RFE)
from sklearn.linear_model import LogisticRegression
from sklearn.feature_selection import RFE

estimator = LogisticRegression(max_iter=1000)
rfe = RFE(estimator, n_features_to_select=20, step=1)
X_rfe = rfe.fit_transform(X, y)
print(f"Selected features: {X.columns[rfe.support_]}")

# 1b. RFECV (CV-based optimal number)
rfecv = RFECV(estimator, step=1, cv=5, scoring='roc_auc')
X_rfecv = rfecv.fit_transform(X, y)
print(f"Optimal features: {rfecv.n_features_}")

# 2. SelectFromModel (feature importance threshold)
from sklearn.ensemble import RandomForestClassifier
rf = RandomForestClassifier(n_estimators=100, random_state=42)
sfm = SelectFromModel(rf, threshold='median', max_features=50)
X_sfm = sfm.fit_transform(X, y)

# 3. Sequential Feature Selection (forward/backward)
from sklearn.feature_selection import SequentialFeatureSelector
sfs = SequentialFeatureSelector(estimator, n_features_to_select=20, 
                                 direction='forward', cv=5)
X_sfs = sfs.fit_transform(X, y)
```

#### Embedded Methods (Model-internal)
```python
# L1 Regularization (Lasso) - sparse coefficients
from sklearn.linear_model import Lasso, LassoCV, LogisticRegression
lasso = LassoCV(cv=5, random_state=42).fit(X, y)
selected = np.where(lasso.coef_ != 0)[0]

# Tree-based feature importance
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
rf = RandomForestClassifier(n_estimators=100, random_state=42)
rf.fit(X, y)
importances = pd.Series(rf.feature_importances_, index=X.columns).sort_values(ascending=False)
selected = importances[importances > importances.mean()].index

# Gradient Boosting importance
import xgboost as xgb
xgb_model = xgb.XGBClassifier().fit(X, y)
importances = pd.Series(xgb_model.feature_importances_, index=X.columns)
```

---

## PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Fit on train, transform test** | Fit on full dataset | Data leakage, optimistic bias |
| **Pipeline for preprocessing** | Manual preprocessing | Inconsistent, hard to reproduce |
| **ColumnTransformer** | Manual column selection | Errors when columns change |
| **Type-specific imputation** | Mean imputation for categorical | Nonsensical values |
| **Target encoding in CV** | Target encoding on full data | Target leakage |
| **Schema validation** | Assume schema static | Silent failures, corrupt data |
| **Schema registry** | No schema management | Breaking changes undetected |
| **Partitioning (Parquet)** | Full scans | Slow queries, high cost |
| **Data contracts** | Verbal agreements | Miscommunication, breakage |
| **Data versioning** | Overwrite files | No reproducibility, no rollback |

---

## INTERVIEW QUESTIONS (Top 15)

### 1. **Data Cleaning**: "How do you handle missing data in a production ML pipeline?"

```
STRATEGY BY MISSINGNESS TYPE:

1. MCAR (Missing Completely at Random):
   - Any imputation works
   - SimpleImputer(strategy='median') fine

2. MAR (Missing at Random - depends on observed):
   - IterativeImputer (MICE) - uses other features
   - KNNImputer - similar samples
   - Model-based imputation

3. MNAR (Missing Not at Random - depends on missing value):
   - Domain knowledge required
   - Add missing indicator feature
   - Domain-specific logic (e.g., "not answered" = negative signal)

PRODUCTION PIPELINE:
1. Fit imputer on TRAIN only
2. Save imputer with model (joblib.dump)
3. Transform test/production data with SAME imputer
4. Monitor missing rates in production (alert if > threshold)

CODE:
from sklearn.impute import SimpleImputer
from sklearn.pipeline import Pipeline

imputer = SimpleImputer(strategy='median')
pipeline = Pipeline([
    ('imputer', imputer),
    ('scaler', StandardScaler()),
    ('model', LogisticRegression())
])
# In production: load fitted pipeline, call predict()
```

---

### 2. **Data Formats**: "Parquet vs CSV - when to use which?"

```
PARQUET (Default for ML/Analytics):
✓ Columnar storage → predicate pushdown (read only needed columns)
✓ Compression (Snappy/Zstd) → 5-10x smaller than CSV
✓ Schema embedded → self-describing, type safety
✓ Partitioning → partition pruning (read only needed partitions)
✓ Schema evolution support
✓ Supported by all major engines (Spark, Pandas, Spark, Presto, Athena, Snowflake)

CSV:
✓ Human-readable, universal
✓ Streaming friendly (no need to read full file)
✓ Simple tools (Excel, text editors)
✗ No schema, no types, no compression (by default)
✗ Full scan always
✗ No partitioning

USE PARQUET FOR:
- ML feature stores
- Data lakes / warehouses
- Large datasets (>1GB)
- Analytics queries
- ML feature stores

USE CSV FOR:
- Human-readable exchange
- Small files (<100MB)
- Interop with legacy systems
- Quick prototyping
```

---

### 3. **Data Cleaning**: "How do you detect and handle data drift in production?"

```
DATA DRIFT TYPES:
1. COVARIATE SHIFT: P(X) changes, P(Y|X) same
   - Feature distributions shift
   - Detection: KS test, PSI, KL divergence on features
   
2. CONCEPT DRIFT: P(Y|X) changes
   - Same features, different relationship to target
   - Detection: Model performance decay, label distribution shift
   
3. PRIOR PROBABILITY SHIFT: P(Y) changes
   - Class balance changes
   - Detection: Class distribution monitoring

DETECTION METHODS:
1. STATISTICAL TESTS:
   - KS test (continuous): scipy.stats.ks_2samp(train_feat, prod_feat)
   - Chi-square (categorical): scipy.stats.chi2_contingency
   - PSI (Population Stability Index): Σ (p_prod - p_train) * ln(p_prod/p_train)
   - PSI < 0.1: stable, 0.1-0.2: moderate, >0.2: significant drift

2. ML-BASED:
   - Train classifier to distinguish train vs prod data (AUC > 0.7 = drift)
   - Domain classifier

3. MONITORING:
   - Feature distributions (daily/weekly)
   - Prediction distributions
   - Performance metrics (if labels available)
   - Data quality metrics (null rates, ranges)

RESPONSE:
1. Retrain with recent data
2. Feature engineering updates
3. Model recalibration (Platt scaling)
4. Feature store updates
3. Alert on-call
```

---

### 4. **Preprocessing**: "When to use StandardScaler vs RobustScaler vs MinMaxScaler?"

```
STANDARDSCALER (z-score):
(x - mean) / std
✓ Gaussian-like data, no outliers
✓ Linear models, PCA, KNN, Neural nets
✗ Outliers distort mean/std → bad scaling

ROBUSTSCALER:
(x - median) / IQR
✓ Outliers present, skewed data
✓ Tree models don't need scaling, but for linear/NN
✗ Less effective for Gaussian data

MINMAXSCALER:
(x - min) / (max - min) → [0, 1]
✓ Neural networks, images, bounded features
✓ Preserves zero for sparse data
✗ Very sensitive to outliers

MAXABSSCALER:
x / max(|x|) → [-1, 1]
✓ Preserves sparsity
✓ Sparse data (text, one-hot)

POWERTRANSFORMER (Yeo-Johnson):
✓ Makes distribution more Gaussian
✓ Reduces skew, helps linear models
✓ Works with negative values (unlike Box-Cox)

QUANTILETRANSFORMER:
Maps to uniform/normal distribution
✓ Non-parametric, any distribution
✓ Good for outliers, heavy tails

CHOICE GUIDE:
- Gaussian, no outliers → StandardScaler
- Outliers/skew → RobustScaler / PowerTransformer
- Neural nets / bounded → MinMaxScaler
- Sparse data → MaxAbsScaler
- Unknown distribution → QuantileTransformer / PowerTransformer
```

---

### 5. **Feature Engineering**: "How do you create features from timestamps?"

```python
def extract_datetime_features(df, col='timestamp'):
    dt = pd.to_datetime(df[col])
    df = df.copy()
    
    # Basic components
    df[f'{col}_year'] = dt.dt.year
    df[f'{col}_month'] = dt.dt.month
    df[f'{col}_day'] = dt.dt.day
    df[f'{col}_dayofweek'] = dt.dt.dayofweek
    df[f'{col}_hour'] = dt.dt.hour
    df[f'{col}_minute'] = dt.dt.minute
    
    # Binary flags
    df[f'{col}_is_weekend'] = (dt.dt.dayofweek >= 5).astype(int)
    df[f'{col}_is_month_start'] = dt.dt.is_month_start.astype(int)
    df[f'{col}_is_month_end'] = dt.dt.is_month_end.astype(int)
    df[f'{col}_is_quarter_start'] = dt.dt.is_quarter_start.astype(int)
    
    # Cyclical encoding (preserves continuity)
    # Hour: 23 → 0 is continuous
    df[f'{col}_sin_hour'] = np.sin(2 * np.pi * dt.dt.hour / 24)
    df[f'{col}_cos_hour'] = np.cos(2 * np.pi * dt.dt.hour / 24)
    df[f'{col}_sin_month'] = np.sin(2 * np.pi * dt.dt.month / 12)
    df[f'{col}_cos_month'] = np.cos(2 * np.pi * dt.dt.month / 12)
    df[f'{col}_sin_dayofweek'] = np.sin(2 * np.pi * dt.dt.dayofweek / 7)
    df[f'{col}_cos_dayofweek'] = np.cos(2 * np.pi * dt.dt.dayofweek / 7)
    
    # Time since epoch (for trend)
    df[f'{col}_days_since_epoch'] = (dt - pd.Timestamp('1970-01-01')).dt.days
    
    # Time since last event (per user)
    df = df.sort_values('user_id', 'timestamp')
    df['time_since_last'] = df.groupby('user_id')['timestamp'].diff().dt.total_seconds()
    
    return df

# Cyclical encoding WHY:
# Hour 23 and 0 are close, but 23-0 = 23 in linear space
# sin/cos: sin(23*2π/24) ≈ sin(0) → close in feature space
```

---

### 6. **Feature Engineering**: "How do you create features from categorical variables with high cardinality?"

```
HIGH CARDINALITY STRATEGIES:

1. TARGET ENCODING (with regularization)
   - Replace category with P(target=1|category)
   - Smooth: (count * cat_mean + α * global_mean) / (count + α)
   - CV-safe: fit on train fold, transform val fold
   
   from category_encoders import TargetEncoder
   te = TargetEncoder(smoothing=10, min_samples_leaf=20)

2. FEATURE HASHING (fixed dimension)
   - Hash category → fixed-size vector
   - Collisions possible but bounded
   from sklearn.feature_extraction import FeatureHasher
   hasher = FeatureHasher(n_features=64, input_type='string')

3. ENTITY EMBEDDINGS (Neural Nets)
   - Learn dense vectors: nn.Embedding(n_categories, dim)
   - Learns similarity from data
   - Handle OOV with special index

4. FREQUENCY/COUNT ENCODING
   - Replace with frequency/count
   - Simple, no leakage risk
   df['cat_freq'] = df['cat'].map(df['cat'].value_counts(normalize=True))

5. LEAVE-ONE-OUT ENCODING (CV-safe target encoding)
   - For each row: encode with target mean of OTHER rows in same category
   from category_encoders import LeaveOneOutEncoder

DECISION GUIDE:
- <20 categories: One-hot
- 20-1000: Target encoding / Embedding
- >1000: Feature hashing / Embedding
- Tree models: Ordinal / Target encoding
- Linear models: Target encoding / One-hot
- Neural nets: Embeddings
```

---

### 7. **Feature Selection**: "How do you select features for a model?"

```
FILTER METHODS (Model-agnostic, fast):
1. VarianceThreshold - Remove constant/quasi-constant
2. Correlation filter - Remove highly correlated (|r| > 0.95)
3. Univariate tests: 
   - Classification: f_classif, mutual_info_classif
   - Regression: f_regression, mutual_info_regression
4. Mutual Information (captures non-linear)

WRAPPER METHODS (Model-based):
1. RFE (Recursive Feature Elimination) - backward elimination
2. RFECV - CV to find optimal number
3. Forward/Backward selection

EMBEDDED (Model-internal):
1. L1 Regularization (Lasso) - sparse coefficients
2. Tree importance - RandomForest, XGBoost feature_importances_
3. LGBM/GBM feature importance

BEST PRACTICE:
1. Start with filter methods (fast, remove obvious noise)
2. Use embedded (Lasso/Tree importance) for initial selection
3. RFECV for final optimization if needed
4. Always do feature selection INSIDE CV loop!
```

---

### 8. **Data Formats**: "Explain Parquet partitioning and predicate pushdown."

```
PARTITIONING:
df.to_parquet('s3://bucket/data/', 
              partition_cols=['date', 'country'],
              compression='snappy')

Creates directory structure:
s3://bucket/data/date=2023-01-01/country=US/part-0.parquet
s3://bucket/data/date=2023-01-01/country=UK/part-0.parquet

PREDICATE PUSHDOWN:
When reading:
pd.read_parquet('s3://bucket/data/',
                filters=[('date', '>=', '2023-01-01'), 
                         ('country', 'in', ['US', 'CA'])],
                columns=['user_id', 'purchase_amount'])

ENGINE:
1. Reads partition metadata (file footers)
2. Skips entire partitions not matching filters
3. Reads only needed columns (column pruning)
4. Reads only needed row groups (row group pruning)

RESULT:
- Reads 1% of data instead of 100%
- 10-100x faster queries
- Lower cost (less data scanned)

BEST PRACTICES:
- Partition by high-cardinality, frequently filtered columns
- Partition size: 100MB-1GB per file
- Avoid over-partitioning (too many small files)
- Partition by time + one categorical dimension
```

---

### 9. **Data Quality**: "How do you implement data quality checks in a pipeline?"

```python
import great_expectations as ge
from great_expectations.dataset import PandasDataset

# Define expectations
def validate_data(df: pd.DataFrame) -> dict:
    """Return validation results"""
    df_ge = ge.from_pandas(df)
    
    results = {
        'row_count': df_ge.expect_table_row_count_to_be_between(min_value=1000),
        'no_nulls_id': df_ge.expect_column_values_to_not_be_null('user_id'),
        'valid_category': df_ge.expect_column_values_to_be_in_set('category', ['A', 'B', 'C']),
        'positive_price': df_ge.expect_column_values_to_be_between('price', min_value=0),
        'unique_ids': df_ge.expect_column_values_to_be_unique('transaction_id'),
        'date_format': df_ge.expect_column_values_to_match_regex('date', r'^\d{4}-\d{2}-\d{2}$'),
        'value_range': df_ge.expect_column_values_to_be_between('score', 0, 100),
    }
    
    # Convert to dict
    return {k: v.success for k, v in results.items()}

# In pipeline (e.g., Airflow, Dagster, Prefect)
def validate_and_alert(df):
    results = validate_data(df)
    failed = [k for k, v in results.items() if not v]
    if failed:
        alert_oncall(f"Data quality failed: {failed}")
        raise DataQualityError(f"Failed checks: {failed}")
    return df

# Pandera (schema validation)
import pandera as pa
from pandera import Column, Check, DataFrameSchema

schema = DataFrameSchema({
    'user_id': Column(int, Check.greater_than(0)),
    'category': Column(str, Check.isin(['A', 'B', 'C'])),
    'price': Column(float, Check.greater_than(0)),
    'timestamp': Column(pd.Timestamp),
})

@pa.check_types
def process_data(df: DataFrameSchema) -> pd.DataFrame:
    return df  # Validates on entry
```

---

### 13. **Data Pipeline**: "How do you handle late-arriving data in streaming?"

```
LATE DATA HANDLING STRATEGIES:

1. WATERMARKS (Event-time processing):
   - Watermark = max_event_time - allowed_lateness
   - Events before watermark → processed
   - Events after watermark → dropped or side output
   
   Flink/Spark:
   .assignTimestampsAndWatermarks(
     WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofMinutes(10)))

2. ALLOWED LATENESS:
   - Process on time, allow late events for window
   - Window fires at watermark, but accepts late data for allowed_lateness
   
   .allowedLateness(Duration.ofHours(1))

3. SIDE OUTPUTS (Late data capture):
   - Late events diverted to side output
   - Process separately (reprocess, alert, store)
   
   late_data_tag = OutputTag("late-data")
   .sideOutputLateData(late_data_tag)

3. STATE MANAGEMENT:
   - Keyed state for windowed aggregations
   - TTL for state cleanup
   - RocksDB backend for large state

4. EXACTLY-ONCE:
   - Kafka transactions (exactly-once source)
   - Checkpointing (periodic state snapshots)
   - Idempotent sinks (idempotent writes)

4. REPROCESSING:
   - Backfill jobs for historical corrections
   - Idempotent writes enable safe reprocessing
   - Separate backfill pipeline from real-time
```

---

### 14. **Feature Store**: "What is a feature store and why do you need one?"

```
FEATURE STORE = Centralized feature management

COMPONENTS:
1. OFFLINE STORE (Historical)
   - Data warehouse / Data lake (Parquet/Delta/Iceberg)
   - Point-in-time correctness (time travel)
   - Training data generation

2. ONLINE STORE (Real-time)
   - Low-latency KV store (Redis, DynamoDB, Cassandra)
   - Feature vector lookup by entity key
   - Sub-millisecond latency

3. FEATURE REGISTRY
   - Feature definitions, metadata, lineage
   - Versioning, documentation, ownership
   - Discovery, reuse

WORKFLOW:
1. Define feature (SQL/Python)
2. Register in registry (metadata, owner, SLA)
3. Materialize to offline store (batch job)
4. Push to online store (streaming/CDC)
5. Training: point-in-time join from offline store
6. Serving: Read from online store (sub-ms)

BENEFITS:
✓ Feature reuse (no duplicate engineering)
✓ Consistency (train/serve same features)
✓ Point-in-time correctness (no leakage)
✓ Governance (lineage, ownership, monitoring)
✓ Scale (separate compute for offline/online)

POPULAR TOOLS:
- Feast (open source)
- Tecton (managed)
- Hopsworks
- Databricks Feature Store
- AWS SageMaker Feature Store
- GCP Vertex AI Feature Store
```

---

### 15. **MLOps**: "How do you implement CI/CD for ML pipelines?"

```
CI/CD FOR ML:

1. CONTINUOUS INTEGRATION (Code + Data):
   - Lint/format (black, flake8, mypy)
   - Unit tests (pytest)
   - Data validation tests (Great Expectations)
   - Model tests (behavioral, invariance)
   - Security scan (bandit, safety)

2. CONTINUOUS DELIVERY (Model Artifacts):
   - Build Docker image with model + dependencies
   - Push to registry (ECR, GCR, Harbor)
   - Scan for vulnerabilities (Trivy, Snyk)

3. CONTINUOUS DEPLOYMENT (Progressive):
   - Canary deployment (5% → 100%)
   - Shadow mode (shadow traffic, compare predictions)
   - A/B testing framework
   - Automated rollback on SLO breach

PIPELINE (GitHub Actions / GitLab CI / Argo):
```yaml
# .github/workflows/ml-pipeline.yml
name: ML Pipeline
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
      - run: pip install -r requirements.txt
      - run: pytest tests/
      - run: python -m pytest tests/data_validation.py
      
  train:
    needs: test
    runs-on: [self-hosted, gpu]
    steps:
      - uses: actions/checkout@v3
      - run: python train.py --config configs/prod.yaml
      - uses: actions/upload-artifact@v3
        with:
          name: model
          path: models/
          
  deploy:
    needs: train
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v3
        with: {name: model}
      - run: docker build -t model:v${{github.sha}} .
      - run: docker push registry/model:v${{github.sha}}
      - run: kubectl set image deployment/ml-model model=registry/model:v${{github.sha}}
```

KEY PRINCIPLES:
1. Immutable artifacts (model + code + config versioned together)
2. Reproducible builds (pinned dependencies, Docker)
3. Automated testing at every level
4. Progressive delivery with observability
5. Rollback capability (git revert = model revert)
4. Audit trail (who, what, when, why)
```

---

## QUICK REFERENCE: DATA ENGINEERING CHEAT SHEET

### Data Cleaning Checklist
- [ ] Missing value analysis (pattern, percentage per column
- [ ] Outlier detection (IQR, Z-score, Isolation Forest)
- [ ] Duplicate detection (exact + fuzzy)
- [ ] Data type consistency
- [ ] Range/constraint validation
- [ ] Categorical standardization
- [ ] Date/time parsing validation
- [ ] Text cleaning (encoding, whitespace, special chars)

### Imputation Strategy Decision
| Missing Type | Method |
|--------------|--------|
| Numerical, MCAR | Mean/Median |
| Numerical, MAR | IterativeImputer/KNN |
| Categorical | Mode / 'missing' category |
| Time series | Forward fill / Interpolation |
| MNAR | Domain logic + missing indicator |

### Feature Engineering Checklist
- [ ] Numerical: log, sqrt, polynomial, ratios
- [ ] Categorical: target encoding, frequency, embedding
- [ ] Datetime: cyclical, parts, lag, rolling
- [ ] Text: TF-IDF, embeddings, stats
- [ ] Image: pretrained embeddings
- [ ] Aggregations: groupby + rolling/expanding
- [ ] Interactions: ratios, products, differences

### Data Format Decision
| Use Case | Format |
|----------|--------|
| ML Feature Store | Parquet (partitioned) |
| Data Lake | Delta Lake / Iceberg |
| Streaming | Avro / Protobuf |
| API Exchange | JSON / Protobuf |
| Human Review | CSV / Excel |
| Low-latency Serving | Protobuf / FlatBuffers |
| Interop (R/Python) | Feather / Parquet |

### Data Quality Metrics to Monitor
| Metric | Alert Threshold |
|--------|-----------------|
| Missing rate | > 5% per column |
| Duplicate rate | > 1% |
| Schema violations | Any |
| Distribution drift (PSI) | > 0.2 |
| Freshness lag | > 2x expected |
| Volume anomaly | > 3σ from baseline |
| Null rate in critical cols | > 0% |

### Feature Store Selection
| Need | Tool |
|------|------|
| Open source, self-hosted | Feast |
| Managed, enterprise | Tecton |
| Databricks ecosystem | Databricks Feature Store |
| AWS native | SageMaker Feature Store |
| GCP native | Vertex AI Feature Store |
| Kubernetes-native | Feast + Redis/Cassandra |