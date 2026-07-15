# UNSUPERVISED LEARNING
## Exam-Style Study Notes

**Roadmap Section:** Unsupervised Learning
**Prerequisites:** Linear algebra, probability, Python, Scikit-learn

---

## 1. CLUSTERING

### WHY? (Problem Statement)
**Find hidden structure in unlabeled data.** Group similar data points together without predefined labels.

### HOW? (Algorithms & Mechanics)

#### K-Means
```math
\min_{C} \sum_{i=1}^{k} \sum_{x \in C_i} \|x - \mu_i\|^2
```
1. Initialize k centroids (K-Means++)
2. Assign each point to nearest centroid
3. Recompute centroids as mean of assigned points
4. Repeat until convergence

```python
from sklearn.cluster import KMeans
from sklearn.metrics import silhouette_score

# Elbow method for k selection
inertias = []
for k in range(2, 11):
    kmeans = KMeans(n_clusters=k, random_state=42, n_init=10)
    kmeans.fit(X)
    inertias.append(kmeans.inertia_)

# Silhouette analysis
for k in range(2, 11):
    kmeans = KMeans(n_clusters=k, random_state=42)
    labels = kmeans.fit_predict(X)
    print(f"k={k}, silhouette={silhouette_score(X, labels):.3f}")
```

#### Hierarchical Clustering
```python
from scipy.cluster.hierarchy import dendrogram, linkage, fcluster
from sklearn.cluster import AgglomerativeClustering

# Linkage methods
# 'ward' - minimizes variance (default, Euclidean only)
# 'complete' - maximum distance
# 'average' - average distance
# 'single' - minimum distance

# Dendrogram
linked = linkage(X, method='ward')
dendrogram(linked, truncate_mode='level', p=5)
plt.show()

# Clustering
agg = AgglomerativeClustering(n_clusters=5, linkage='ward')
labels = agg.fit_predict(X)
```

#### DBSCAN (Density-Based)
```python
from sklearn.cluster import DBSCAN

# eps: max distance between neighbors
# min_samples: minimum points to form dense region
# Finds arbitrary shapes, handles noise (-1 label)

dbscan = DBSCAN(eps=0.5, min_samples=5)
labels = dbscan.fit_predict(X)
n_clusters = len(set(labels)) - (1 if -1 in labels else 0)
n_noise = list(labels).count(-1)
```

#### HDBSCAN (Hierarchical DBSCAN)
```python
import hdbscan

# More robust, automatic cluster selection
clusterer = hdbscan.HDBSCAN(
    min_cluster_size=5,
    min_samples=5,
    cluster_selection_method='eom'  # Excess of Mass
)
labels = clusterer.fit_predict(X)
probabilities = clusterer.probabilities_  # Membership strength
```

---

## 2. DIMENSIONALITY REDUCTION

### WHY? (Problem Statement)
**Curse of dimensionality.** Reduce features while preserving information:
- Visualization (2D/3D)
- Noise reduction
- Computational efficiency
- Avoid overfitting

### HOW? (Techniques)

#### PCA (Principal Component Analysis)
```python
from sklearn.decomposition import PCA
import numpy as np

# PCA finds orthogonal directions of maximum variance
pca = PCA(n_components=2)
X_pca = pca.fit_transform(X)

# Explained variance
print(f"Explained variance ratio: {pca.explained_variance_ratio_}")
print(f"Cumulative: {np.cumsum(pca.explained_variance_ratio_)}")

# Choose n_components for 95% variance
pca = PCA(n_components=0.95)
X_pca = pca.fit_transform(X)
```

#### t-SNE (Visualization)
```python
from sklearn.manifold import TSNE

# t-SNE preserves local structure (good for visualization)
tsne = TSNE(n_components=2, perplexity=30, random_state=42)
X_tsne = tsne.fit_transform(X)

# For large datasets: use PCA first, then t-SNE
pca = PCA(n_components=50)
X_pca = pca.fit_transform(X)
X_tsne = TSNE(n_components=2).fit_transform(X_pca)
```

#### UMAP (Fast, Preserves Global + Local)
```python
import umap

# UMAP: faster, preserves global + local structure
reducer = umap.UMAP(n_components=2, n_neighbors=15, min_dist=0.1)
X_umap = reducer.fit_transform(X)

# Can transform new data
X_new_umap = reducer.transform(X_new)
```

#### Autoencoders (Non-linear, Deep)
```python
import torch
import torch.nn as nn

class Autoencoder(nn.Module):
    def __init__(self, input_dim, encoding_dim=32):
        super().__init__()
        self.encoder = nn.Sequential(
            nn.Linear(input_dim, 128),
            nn.ReLU(),
            nn.Linear(128, 64),
            nn.ReLU(),
            nn.Linear(64, encoding_dim)
        )
        self.decoder = nn.Sequential(
            nn.Linear(encoding_dim, 64),
            nn.ReLU(),
            nn.Linear(64, 128),
            nn.ReLU(),
            nn.Linear(128, input_dim)
        )
    
    def forward(self, x):
        encoded = self.encoder(x)
        decoded = self.decoder(encoded)
        return decoded, encoded

# Variational Autoencoder (VAE) - generative
class VAE(nn.Module):
    def __init__(self, input_dim, latent_dim=20):
        super().__init__()
        # Encoder
        self.fc1 = nn.Linear(input_dim, 256)
        self.fc_mu = nn.Linear(256, latent_dim)
        self.fc_logvar = nn.Linear(256, latent_dim)
        # Decoder
        self.fc3 = nn.Linear(latent_dim, 256)
        self.fc4 = nn.Linear(256, input_dim)
    
    def encode(self, x):
        h = F.relu(self.fc1(x))
        return self.fc_mu(h), self.fc_logvar(h)
    
    def reparameterize(self, mu, logvar):
        std = torch.exp(0.5 * logvar)
        eps = torch.randn_like(std)
        return mu + eps * std
    
    def forward(self, x):
        mu, logvar = self.encode(x)
        z = self.reparameterize(mu, logvar)
        return self.decode(z), mu, logvar
```

---

## 3. ANOMALY DETECTION

### WHY? (Problem Statement)
**Find rare events / outliers.** Fraud detection, system monitoring, quality control.

### HOW? (Methods)

| Method | Type | Best For |
|--------|------|----------|
| **Isolation Forest** | Tree-based | High-dim, fast |
| **Local Outlier Factor** | Density-based | Local anomalies |
| **One-Class SVM** | Boundary-based | Small datasets |
| **Elliptic Envelope** | Gaussian assumption | Gaussian-like data |
| **Autoencoder** | Reconstruction error | Complex patterns, deep learning |
| **Statistical (Z-score, IQR)** | Statistical | Simple, univariate |

```python
from sklearn.ensemble import IsolationForest
from sklearn.neighbors import LocalOutlierFactor
from sklearn.svm import OneClassSVM
from sklearn.covariance import EllipticEnvelope

# Isolation Forest (most popular)
iso = IsolationForest(contamination=0.01, random_state=42)
preds = iso.fit_predict(X)
scores = iso.decision_function(X)  # Lower = more anomalous

# Local Outlier Factor
lof = LocalOutlierFactor(n_neighbors=20, contamination=0.01)
preds = lof.fit_predict(X)

# Autoencoder anomaly detection
# Train on normal data only
# Anomaly score = reconstruction error
```

---

## PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Scale before clustering** | Raw features | Features with large scale dominate |
| **Elbow + Silhouette for k** | Arbitrary k | Wrong number of clusters |
| **PCA before clustering** | High-dim raw | Noise, curse of dimensionality |
| **Multiple algorithms** | Single algorithm | Different algorithms find different structures |
| **Visualize (UMAP/t-SNE)** | Only metrics | Miss structural patterns |
| **Domain validation** | Pure metrics | Clusters may be meaningless |

---

## INTERVIEW QUESTIONS (Top 15)

### 1. **Conceptual**: "K-Means vs DBSCAN vs Hierarchical - when to use which?"

```
K-MEANS:
✓ Spherical clusters, known k, large datasets
✗ Non-spherical, varying density, outliers, unknown k

DBSCAN:
✓ Arbitrary shapes, noise handling, no k needed
✗ Varying densities, high-dimensional, parameter sensitive

HIERARCHICAL:
✓ Dendrogram interpretation, no k needed (dendrogram cut)
✗ O(n²) or O(n³), slow for large data, sensitive to noise

DECISION TREE:
1. Know k? → K-Means (or GMM)
2. Unknown k, arbitrary shapes, noise → DBSCAN/HDBSCAN
3. Need hierarchy/interpretability → Hierarchical
4. Large n, unknown k, arbitrary shapes → HDBSCAN
```

### 2. **Code**: "Implement K-Means from scratch."

```python
import numpy as np

class KMeans:
    def __init__(self, n_clusters=3, max_iter=300, tol=1e-4, random_state=42):
        self.n_clusters = n_clusters
        self.max_iter = max_iter
        self.tol = tol
        self.random_state = random_state
    
    def _kmeans_plus_plus(self, X):
        """K-Means++ initialization"""
        np.random.seed(self.random_state)
        n_samples = X.shape[0]
        centroids = [X[np.random.randint(0, n_samples)]]
        
        for _ in range(1, self.n_clusters):
            distances = np.min(np.sum((X[:, np.newaxis] - centroids)**2, axis=2), axis=1)
            probs = distances / distances.sum()
            next_idx = np.random.choice(n_samples, p=probs)
            centroids.append(X[next_idx])
        
        return np.array(centroids)
    
    def fit(self, X):
        np.random.seed(self.random_state)
        centroids = self._kmeans_plus_plus(X)
        
        for i in range(self.max_iter):
            # Assign
            distances = np.sum((X[:, np.newaxis] - centroids)**2, axis=2)
            labels = np.argmin(distances, axis=1)
            
            # Update
            new_centroids = np.array([X[labels == k].mean(axis=0) for k in range(self.n_clusters)])
            
            # Check convergence
            if np.max(np.abs(new_centroids - centroids)) < self.tol:
                break
            centroids = new_centroids
        
        self.cluster_centers_ = centroids
        self.labels_ = labels
        self.inertia_ = np.sum((X - centroids[labels])**2)
        return self
    
    def predict(self, X):
        distances = np.sum((X[:, np.newaxis] - self.cluster_centers_)**2, axis=2)
        return np.argmin(distances, axis=1)
```

### 3. **Design**: "How would you cluster 10M user embeddings?"

```
SCALABILITY CHALLENGES:
- K-Means: O(n * k * d * iterations) - too slow
- DBSCAN: O(n²) - impossible
- HDBSCAN: O(n log n) - better but still heavy

SOLUTIONS:

1. MINI-BATCH K-MEANS (Scikit-learn)
   - Updates centroids on small batches
   - O(b * k * d) per iteration
   - Scikit-learn: MiniBatchKMeans

2. FAISS (Facebook AI Similarity Search)
   - GPU-accelerated K-Means
   - IVF (Inverted File Index) for ANN
   - 100x faster than sklearn
   
   import faiss
   kmeans = faiss.Kmeans(d, k, niter=20, gpu=True)
   kmeans.train(X.astype(np.float32))

3. MINI-BATCH K-MEANS + PCA
   - Reduce dimensionality first (PCA to 50-100 dims)
   - Then MiniBatchKMeans
   - 100x speedup

4. DISTRIBUTED (Spark MLlib)
   - Distributed K-Means
   - For truly massive scale

5. APPROXIMATE NEAREST NEIGHBORS
   - HNSW, Annoy, ScaNN
   - Build graph, cluster on graph
```

### 4. **Code**: "Implement PCA from scratch using SVD."

```python
import numpy as np

class PCA:
    def __init__(self, n_components=None, var_threshold=None):
        self.n_components = n_components
        self.var_threshold = var_threshold
        self.components_ = None
        self.mean_ = None
        self.explained_variance_ = None
        self.explained_variance_ratio_ = None
    
    def fit(self, X):
        # Center data
        self.mean_ = X.mean(axis=0)
        X_centered = X - self.mean_
        
        # SVD
        U, S, Vt = np.linalg.svd(X_centered, full_matrices=False)
        
        # Components = top-k right singular vectors
        # S contains singular values
        n_samples = X.shape[0]
        explained_var = (S ** 2) / (n_samples - 1)
        total_var = explained_var.sum()
        explained_ratio = explained_var / total_var
        
        # Determine n_components
        if self.n_components is not None:
            n_comp = self.n_components
        elif self.var_threshold is not None:
            cumsum = np.cumsum(explained_ratio)
            n_comp = np.argmax(cumsum >= self.var_threshold) + 1
        else:
            n_comp = min(X.shape)
        
        self.components_ = Vt[:n_comp]
        self.explained_variance_ = explained_var[:n_comp]
        self.explained_variance_ratio_ = explained_ratio[:n_comp]
        
        return self
    
    def transform(self, X):
        X_centered = X - self.mean_
        return X_centered @ self.components_.T
    
    def inverse_transform(self, X_transformed):
        return X_transformed @ self.components_ + self.mean_

# Usage
pca = PCA(n_components=2)
X_pca = pca.fit_transform(X)
```

### 5. **Conceptual**: "t-SNE vs UMAP - when to use which?"

```
t-SNE:
✓ Excellent local structure preservation
✓ Standard for visualization (papers, presentations)
✗ Slow O(n²) or O(n log n) with Barnes-Hut
✗ No transform() for new data (must refit)
✗ Hyperparameter sensitive (perplexity)
✗ No global structure preservation

UMAP:
✓ Fast O(n log n) - 10-100x faster
✓ Preserves BOTH local AND global structure
✓ Has transform() for new data
✓ More robust hyperparameters
✓ Scales to millions of points
✓ Can be used for feature reduction (not just viz)

USE t-SNE WHEN:
- Publication-quality visualizations
- Small datasets (<50k)
- Only need 2D/3D visualization

USE UMAP WHEN:
- Large datasets (>50k)
- Need to transform new data
- Feature reduction for downstream ML
- Need global structure preservation
- Interactive exploration
```

### 5. **Code**: "Implement DBSCAN from scratch."

```python
import numpy as np
from collections import deque

class DBSCAN:
    def __init__(self, eps=0.5, min_samples=5):
        self.eps = eps
        self.min_samples = min_samples
    
    def _region_query(self, X, point_idx):
        distances = np.linalg.norm(X - X[point_idx], axis=1)
        return np.where(distances <= self.eps)[0]
    
    def fit(self, X):
        n_samples = X.shape[0]
        labels = np.full(n_samples, -1)  # -1 = unvisited
        cluster_id = 0
        
        for i in range(n_samples):
            if labels[i] != -1:
                continue  # Already visited
            
            neighbors = self._region_query(X, i)
            
            if len(neighbors) < self.min_samples:
                labels[i] = -2  # Noise (temporarily -2)
                continue
            
            # Expand cluster
            labels[i] = cluster_id
            queue = deque(neighbors)
            
            while queue:
                q_idx = queue.popleft()
                if labels[q_idx] == -2:  # Was noise
                    labels[q_idx] = cluster_id
                
                if labels[q_idx] != -1:
                    continue  # Already assigned
                
                labels[q_idx] = cluster_id
                q_neighbors = self._region_query(X, q_idx)
                
                if len(q_neighbors) >= self.min_samples:
                    queue.extend(q_neighbors)
            
            cluster_id += 1
        
        # Convert noise labels
        labels[labels == -2] = -1
        self.labels_ = labels
        return self
```

### 5. **Conceptual**: "How do you evaluate clustering quality without labels?"

```
INTERNAL METRICS (no ground truth):

1. SILHOUETTE SCORE (-1 to 1):
   s = (b - a) / max(a, b)
   a = mean intra-cluster distance
   b = mean nearest-cluster distance
   Higher = better separation

2. CALINSKI-HARABAZ INDEX:
   Ratio of between-cluster dispersion to within-cluster
   Higher = better

3. DAVIES-BOULDIN INDEX:
   Average similarity between clusters
   Lower = better

EXTERNAL METRICS (with ground truth):
- Adjusted Rand Index (ARI)
- Normalized Mutual Information (NMI)
- Adjusted Mutual Information (AMI)
- Fowlkes-Mallows Score

VISUAL VALIDATION:
- UMAP/t-SNE colored by clusters
- Pairwise feature plots colored by cluster
- Cluster size distribution

STABILITY:
- Bootstrap resampling
- Compare clusterings (ARI between runs)
```

### 6. **Code**: "Implement Isolation Forest anomaly detection."

```python
import numpy as np
import random

class IsolationTree:
    def __init__(self, max_depth=10):
        self.max_depth = max_depth
        self.split_feature = None
        self.split_value = None
        self.left = None
        self.right = None
        self.size = 0
        self.is_leaf = False
    
    def fit(self, X, depth=0):
        self.size = len(X)
        
        if depth >= self.max_depth or len(X) <= 1:
            self.is_leaf = True
            return
        
        # Random feature and split value
        n_features = X.shape[1]
        self.split_feature = random.randint(0, n_features - 1)
        col_min, col_max = X[:, self.split_feature].min(), X[:, self.split_feature].max()
        self.split_value = random.uniform(col_min, col_max)
        
        left_mask = X[:, self.split_feature] < self.split_value
        right_mask = ~left_mask
        
        if np.sum(left_mask) > 0:
            self.left = IsolationTree(self.max_depth)
            self.left.fit(X[left_mask], depth + 1)
        
        if np.sum(~left_mask) > 0:
            self.right = IsolationTree(self.max_depth)
            self.right.fit(X[~left_mask], depth + 1)
    
    def path_length(self, x, depth=0):
        if self.is_leaf:
            return depth + self._c_factor(self.size)
        
        if x[self.split_feature] < self.split_value:
            return self.left.path_length(x, depth + 1) if self.left else depth
        else:
            return self.right.path_length(x, depth + 1) if self.right else depth
    
    def _c_factor(self, n):
        if n <= 1:
            return 0
        return 2 * (np.log(n - 1) + 0.5772156649) - 2 * (n - 1) / n


class IsolationForest:
    def __init__(self, n_estimators=100, max_samples=256, max_depth=10, contamination=0.1):
        self.n_estimators = n_estimators
        self.max_samples = max_samples
        self.max_depth = max_depth
        self.contamination = contamination
        self.trees = []
    
    def fit(self, X):
        self.trees = []
        n_samples = X.shape[0]
        
        for _ in range(self.n_estimators):
            # Subsample
            sample_size = min(self.max_samples, n_samples)
            idx = np.random.choice(n_samples, sample_size, replace=False)
            X_sample = X[idx]
            
            tree = IsolationTree(max_depth=self.max_depth)
            tree.fit(X_sample)
            self.trees.append(tree)
        
        # Compute anomaly scores
        scores = self.decision_function(X)
        self.threshold_ = np.percentile(scores, 100 * (1 - self.contamination))
        return self
    
    def decision_function(self, X):
        # Average path length (lower = more anomalous)
        avg_path_length = np.mean([[tree.path_length(x) for tree in self.trees] for x in X], axis=1)
        # Normalize
        c = self._c_factor(self.max_samples)
        scores = 2 ** (-avg_path_length / c)
        return scores
    
    def predict(self, X):
        scores = self.decision_function(X)
        return np.where(scores > self.threshold_, 1, -1)
    
    def _c_factor(self, n):
        if n <= 1: return 0
        return 2 * (np.log(n - 1) + 0.5772156649) - 2 * (n - 1) / n
```

### 7. **Code**: "HDBSCAN clustering with visualization."

```python
import hdbscan
import umap
import matplotlib.pyplot as plt
import seaborn as sns

# HDBSCAN clustering
clusterer = hdbscan.HDBSCAN(
    min_cluster_size=15,
    min_samples=5,
    cluster_selection_method='eom',  # Excess of Mass
    metric='euclidean'
)
clusterer.fit(X)

# Results
labels = clusterer.labels_
probabilities = clusterer.probabilities_  # Membership strength
n_clusters = len(set(labels)) - (1 if -1 in labels else 0)

# Visualize with UMAP
reducer = umap.UMAP(n_components=2, n_neighbors=15, min_dist=0.1)
X_umap = reducer.fit_transform(X)

plt.figure(figsize=(12, 10))
# Color by cluster
scatter = plt.scatter(X_umap[:, 0], X_umap[:, 1], c=labels, 
                      cmap='tab20', s=10, alpha=0.7)
plt.colorbar(scatter, label='Cluster')
plt.title(f'HDBSCAN Clustering ({n_clusters} clusters)')
plt.show()

# Cluster probabilities
plt.figure(figsize=(10, 6))
plt.hist(probabilities[labels != -1], bins=50, alpha=0.7, label='Clustered')
plt.hist(probabilities[labels == -1], bins=50, alpha=0.7, label='Noise')
plt.xlabel('Cluster Membership Probability')
plt.ylabel('Frequency')
plt.legend()
plt.show()

# Cluster persistence (stability across epsilon)
# clusterer.condensed_tree_.plot(select_clusters=True, selection_palette='tab20')
```

### 8. **Design**: "How would you build a customer segmentation system?"

```
CUSTOMER SEGMENTATION PIPELINE:

1. FEATURE ENGINEERING:
   - RFM features (Recency, Frequency, Monetary)
   - Behavioral: categories browsed, devices, channels
   - Temporal: seasonality, tenure, lifecycle stage
   - Text: review embeddings, search queries
   - Image: product image embeddings

2. PREPROCESSING:
   - Handle missing (median for numerical, mode for categorical)
   - Outlier capping (winsorize 1-99 percentile)
   - Scaling: RobustScaler (outlier-robust)
   - Categorical: Target encoding / Embeddings
   - Text/Image: Pre-trained embeddings

3. DIMENSIONALITY REDUCTION:
   - PCA (50-100 components) for noise reduction
   - UMAP (2-10 dims) for visualization

3. CLUSTERING:
   - HDBSCAN (robust, auto-k, handles noise)
   - MiniBatchKMeans (scalable baseline)
   - Compare: silhouette, stability, business interpretability

4. VALIDATION:
   - Internal: Silhouette, Davies-Bouldin, stability (bootstrap)
   - Business: Segment size, revenue per segment, actionability
   - Profile: Feature importance per cluster (SHAP)

4. PRODUCTION:
   - Assign new users: nearest centroid / HDBSCAN approximate_predict
   - Recluster monthly (or when drift detected)
   - Monitor: segment sizes, migration rates, revenue per segment

CODE STRUCTURE:
class CustomerSegmentation:
    def __init__(self, n_clusters=None, min_cluster_size=50):
        self.reducer = umap.UMAP(n_components=50)
        self.clusterer = hdbscan.HDBSCAN(min_cluster_size=min_cluster_size)
    
    def fit(self, X):
        X_reduced = self.reducer.fit_transform(X)
        self.clusterer.fit(X_reduced)
        return self
    
    def predict(self, X):
        X_reduced = self.reducer.transform(X)
        return self.clusterer.approximate_predict(X_reduced)[0]
```

### 7. **Code**: "Gaussian Mixture Model with model selection."

```python
from sklearn.mixture import GaussianMixture
from sklearn.model_selection import GridSearchCV
import numpy as np

# GMM with full covariance
gmm = GaussianMixture(
    n_components=3,
    covariance_type='full',  # 'full', 'tied', 'diag', 'spherical'
    max_iter=200,
    n_init=10,
    random_state=42
)
gmm.fit(X)
labels = gmm.predict(X)
probs = gmm.predict_proba(X)  # Soft assignments

# Model selection: BIC / AIC
def select_gmm_n_components(X, max_k=10):
    bics, aics = [], []
    models = []
    
    for k in range(1, max_k + 1):
        gmm = GaussianMixture(n_components=k, covariance_type='full',
                              random_state=42, n_init=5)
        gmm.fit(X)
        bics.append(gmm.bic(X))
        aics.append(gmm.aic(X))
        models.append(gmm)
    
    best_k_bic = np.argmin(bics) + 1
    best_k_aic = np.argmin(aics) + 1
    
    plt.plot(range(1, max_k+1), bics, 'o-', label='BIC')
    plt.plot(range(1, max_k+1), aics, 's-', label='AIC')
    plt.xlabel('n_components')
    plt.legend()
    plt.show()
    
    return models[best_k_bic - 1], best_k_bic

# Soft clustering advantages:
# - Probabilistic assignments
# - Handles overlapping clusters
# - Provides uncertainty (probabilities)
# - Model selection via BIC/AIC
```

### 8. **Conceptual**: "What's the difference between PCA and Autoencoder?"

```
PCA:
✓ Linear dimensionality reduction
✓ Convex optimization (global optimum)
✓ Interpretable components (linear combinations)
✓ Fast, deterministic, no hyperparameters
✓ Optimal for linear reconstruction error
✗ Cannot capture non-linear manifolds
✗ Limited to linear subspaces

AUTOENCODER:
✓ Non-linear dimensionality reduction
✓ Learns complex manifolds
✓ Can be regularized (sparse, denoising, variational)
✓ Learns hierarchical features
✗ Non-convex (local minima)
✗ Needs hyperparameter tuning (architecture, lr, regularization)
✗ More data required
✗ Harder to interpret

WHEN TO USE:
PCA: d < 1000, linear relationships, baseline, interpretability needed
AUTOENCODER: Complex non-linear structure, images, sequences, sufficient data

VARIANTS:
- Sparse AE: L1 on activations (feature selection)
- Denoising AE: Corrupt input, reconstruct clean (robustness)
- Contractive AE: Jacobian penalty (local invariance)
- VAE: Probabilistic, generative, latent space sampling
- Contractive AE: Jacobian norm penalty
```

### 9. **Code**: "Variational Autoencoder (VAE) implementation."

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import torch.optim as optim

class VAE(nn.Module):
    def __init__(self, input_dim, hidden_dim=256, latent_dim=20):
        super().__init__()
        self.latent_dim = latent_dim
        
        # Encoder
        self.fc1 = nn.Linear(input_dim, hidden_dim)
        self.fc_mu = nn.Linear(hidden_dim, latent_dim)
        self.fc_logvar = nn.Linear(hidden_dim, latent_dim)
        
        # Decoder
        self.fc3 = nn.Linear(latent_dim, hidden_dim)
        self.fc4 = nn.Linear(hidden_dim, input_dim)
    
    def encode(self, x):
        h = F.relu(self.fc1(x))
        return self.fc_mu(h), self.fc_logvar(h)
    
    def reparameterize(self, mu, logvar):
        std = torch.exp(0.5 * logvar)
        eps = torch.randn_like(std)
        return mu + eps * std
    
    def decode(self, z):
        h = F.relu(self.fc3(z))
        return torch.sigmoid(self.fc4(h))  # Sigmoid for [0,1] output
    
    def forward(self, x):
        mu, logvar = self.encode(x)
        z = self.reparameterize(mu, logvar)
        recon = self.decode(z)
        return recon, mu, logvar

def vae_loss(recon_x, x, mu, logvar):
    # Reconstruction loss (BCE for [0,1] data)
    recon_loss = F.binary_cross_entropy(recon_x, x, reduction='sum')
    # KL divergence
    kl_loss = -0.5 * torch.sum(1 + logvar - mu.pow(2) - logvar.exp())
    return recon_loss + kl_loss

# Training
vae = VAE(input_dim=784, latent_dim=20)
optimizer = optim.Adam(vae.parameters(), lr=1e-3)

for epoch in range(epochs):
    for batch in dataloader:
        x = batch[0].view(-1, 784)
        optimizer.zero_grad()
        recon, mu, logvar = vae(x)
        loss = vae_loss(recon, x, mu, logvar)
        loss.backward()
        optimizer.step()
```

### 10. **Conceptual**: "How do you choose the number of clusters?"

```
METHODS:

1. ELBOW METHOD (K-Means):
   - Plot inertia vs k
   - Look for "elbow" (diminishing returns)
   - Subjective, not always clear

2. SILHOUETTE ANALYSIS:
   - Compute silhouette for k=2...10
   - Choose k with max average silhouette
   - Works for any clustering algorithm

3. GAP STATISTIC:
   - Compare log(inertia) to null reference distribution
   - Gap(k) = E[log(W_k)] - log(W_k)
   - Choose smallest k where Gap(k) >= Gap(k+1) - s_{k+1}

4. BIC/AIC (GMM):
   - BIC = -2 log L + k ln(n)
   - AIC = -2 log L + 2k
   - Lower = better

4. STABILITY (Bootstrap):
   - Resample data, cluster, compare (ARI)
   - Stable k = consistent clusters

5. DOMAIN / BUSINESS:
   - Actionability of segments
   - Minimum segment size
   - Interpretability

PRACTICAL APPROACH:
1. Run multiple methods
2. Look for consensus
3. Validate with domain experts
4. Prefer simpler (fewer clusters) if similar quality
```

---

## QUICK REFERENCE: UNSUPERVISED LEARNING CHEAT SHEET

### Algorithm Selection
| Need | Algorithm |
|------|-----------|
| Spherical clusters, known k | K-Means / MiniBatchKMeans |
| Arbitrary shapes, noise | DBSCAN / HDBSCAN |
| Hierarchy needed | Agglomerative |
| Probabilistic / soft clusters | Gaussian Mixture |
| Large scale (>100k) | MiniBatchKMeans / HDBSCAN / FAISS |
| Non-linear manifolds | UMAP / t-SNE / Autoencoder |
| Outlier detection | Isolation Forest / LOF |
| Categorical data | K-Modes / K-Prototypes |

### Dimensionality Reduction Selection
| Need | Method |
|------|--------|
| Linear, interpretable, fast | PCA |
| Visualization (small) | t-SNE |
| Visualization + features (large) | UMAP |
| Non-linear features | Autoencoder / VAE |
| Preserve global + local | UMAP |
| Sparse features | TruncatedSVD |

### Clustering Evaluation
| Type | Metric |
|------|--------|
| Internal (no labels) | Silhouette, Calinski-Harabasz, Davies-Bouldin |
| External (labels) | ARI, NMI, AMI, Fowlkes-Mallows |
| Stability | Bootstrap ARI |

### Anomaly Detection Selection
| Data Type | Method |
|-----------|--------|
| Tabular, medium | Isolation Forest |
| High-dim, complex | Autoencoder |
| Density-based | Local Outlier Factor |
| Time series | LSTM-AE, Prophet residuals |
| Simple/univariate | IQR, Z-score, Isolation Forest |

### Scaling Guide
| Scale | Approach |
|-------|----------|
| <10k | Full K-Means, DBSCAN, Hierarchical |
| 10k-100k | MiniBatchKMeans, HDBSCAN, UMAP |
| 100k-1M | MiniBatchKMeans, FAISS K-Means, HDBSCAN |
| >1M | FAISS, Spark MLlib, Distributed K-Means |

### Key Hyperparameters
| Algorithm | Key Params |
|-----------|------------|
| K-Means | n_clusters, n_init, max_iter |
| DBSCAN | eps, min_samples |
| HDBSCAN | min_cluster_size, min_samples, cluster_selection_method |
| GMM | n_components, covariance_type |
| PCA | n_components / variance_ratio |
| UMAP | n_neighbors, min_dist, n_components |
| t-SNE | perplexity, learning_rate, n_iter |
| Isolation Forest | n_estimators, contamination, max_samples |
| Autoencoder | encoding_dim, hidden_dims, activation |