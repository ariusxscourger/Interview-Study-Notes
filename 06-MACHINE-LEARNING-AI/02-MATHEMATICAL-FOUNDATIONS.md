# MATHEMATICAL FOUNDATIONS FOR MACHINE LEARNING
## Exam-Style Study Notes

**Roadmap Section:** Mathematical Foundations (Calculus, Linear Algebra, Probability & Statistics)
**Prerequisites:** High school mathematics (algebra, basic calculus, basic probability)

---

## 1. CALCULUS

### WHY? (Problem Statement)
**ML is optimization.** Every training loop minimizes a loss function. Calculus gives us:
- **Derivatives** → Gradient (direction of steepest descent)
- **Partial derivatives** → Gradient w.r.t each parameter (backpropagation)
- **Chain rule** → Backpropagation through computational graphs
- **Hessian** → Second-order optimization (Newton's method, curvature)

### HOW? (Core Concepts)

#### Derivatives & Partial Derivatives
```math
f'(x) = lim_{h→0} [f(x+h) - f(x)] / h          # Single variable
∂f/∂xᵢ                                         # Partial derivative (multi-variable)
∇f = [∂f/∂x₁, ∂f/∂x₂, ..., ∂f/∂xₙ]ᵀ           # Gradient vector
```

#### Chain Rule (Backpropagation Foundation)
```math
# Single variable: df/dx = df/du · du/dx
# Multi-variable (computational graph):
df/dx = Σ (∂f/∂uᵢ) · (duᵢ/dx)

# Neural network example:
∂L/∂W = ∂L/∂ŷ · ∂ŷ/∂z · ∂z/∂W
       ↑       ↑       ↑
     loss    activation  linear    weights
```

#### Gradient, Jacobian, Hessian
| Matrix | Shape | Meaning |
|--------|-------|---------|
| **Gradient ∇f** | (n,) | First derivatives, direction of steepest ascent |
| **Jacobian J** | (m, n) | First derivatives of vector function f: ℝⁿ → ℝᵐ |
| **Hessian H** | (n, n) | Second derivatives, curvature matrix |

```python
import torch
x = torch.tensor([2.0, 3.0], requires_grad=True)
y = x[0]**2 + 3*x[1]**3
y.backward()
print(x.grad)  # Gradient: [4, 81]

# Jacobian (for vector outputs)
from torch.autograd.functional import jacobian
def f(x): return torch.stack([x[0]**2, x[0]*x[1]])
print(jacobian(f, torch.tensor([2., 3.])))

# Hessian (second derivatives)
from torch.autograd.functional import hessian
def f(x): return x[0]**2 + 3*x[1]**3
print(hessian(f, torch.tensor([2., 3.])))
```

### EXAMPLES

**Example 1: Gradient Descent on f(x) = x²**
```python
# f(x) = x², f'(x) = 2x
# GD: x_{t+1} = x_t - η · 2x_t
x = 10.0
lr = 0.1
for i in range(20):
    grad = 2 * x
    x -= lr * grad
    print(f"Step {i}: x={x:.4f}, f(x)={x**2:.4f}")
# Converges to 0
```

**Example 2: Chain Rule in 2-layer Network**
```python
# y = σ(W₂ σ(W₁x + b₁) + b₂)
# ∂L/∂W₁ = ∂L/∂y · ∂y/∂h₂ · ∂h₂/∂h₁ · ∂h₁/∂W₁
#          ↑        ↑        ↑        ↑
#       loss    output   hidden1  input
```

---

## 2. LINEAR ALGEBRA

### WHY? (Problem Statement)
**Data = Matrices.** Every ML operation is linear algebra:
- **Data matrix** X ∈ ℝⁿˣᵈ (n samples, d features)
- **Weights** W ∈ ℝᵈˣᵏ
- **Forward pass** = Matrix multiplication
- **PCA/SVD** = Eigendecomposition
- **Recommendation** = Matrix factorization

### HOW? (Core Concepts)

#### Vectors, Matrices, Tensors
| Object | Notation | Shape | ML Use |
|--------|----------|-------|--------|
| Scalar | a | () | Learning rate, loss |
| Vector | x | (d,) | Feature vector, bias |
| Matrix | X | (n, d) | Dataset, weights |
| 3D Tensor | T | (b, s, d) | Batch, sequence, features |

#### Matrix Operations
| Operation | Notation | Shape Rule | ML Use |
|-----------|----------|------------|--------|
| Dot product | xᵀy | (d,) · (d,) → () | Similarity, attention |
| Matrix multiply | AB | (m, n) × (n, p) → (m, p) | Forward pass, projection |
| Transpose | Aᵀ | (m, n) → (n, m) | Gradient computation |
| Inverse | A⁻¹ | (n, n) → (n, n) | Normal equation, whitening |
| Determinant | det(A) | (n, n) → ℝ | Volume scaling, invertibility |

#### Eigendecomposition & SVD
```math
# Eigendecomposition (square matrices)
A = QΛQ⁻¹
Λ = diag(λ₁, ..., λₙ)    # Eigenvalues
Q = [v₁, ..., vₙ]        # Eigenvectors (orthogonal if A symmetric)

# SVD (ANY matrix)
A = UΣVᵀ
U ∈ ℝᵐˣᵐ (left singular vectors)
Σ ∈ ℝᵐˣⁿ (singular values σ₁ ≥ σ₂ ≥ ...)
V ∈ ℝⁿˣⁿ (right singular vectors)

# Applications:
# - PCA: Keep top-k singular vectors
# - Recommender systems: A ≈ UₖΣₖVₖᵀ
# - Dimensionality reduction
# - Noise reduction (truncate small σ)
```

#### Key Decompositions for ML
| Decomposition | Formula | Use Case |
|---------------|---------|----------|
| **Eigendecomposition** | A = QΛQ⁻¹ | PCA (covariance matrix), spectral clustering |
| **SVD** | A = UΣVᵀ | Recommender systems, NLP (LSA), compression |
| **QR** | A = QR | Linear regression (stable), least squares |
| **Cholesky** | A = LLᵀ | Gaussian processes, sampling (A positive definite) |

### EXAMPLES

**Example 1: PCA from Scratch**
```python
import numpy as np

def pca(X, k):
    # X: (n, d) centered data
    # 1. Covariance matrix
    Σ = X.T @ X / (len(X) - 1)  # (d, d)
    # 2. Eigendecomposition
    eigvals, eigvecs = np.linalg.eigh(Σ)  # ascending order
    # 3. Top-k eigenvectors
    idx = np.argsort(eigvals)[::-1][:k]
    components = eigvecs[:, idx]  # (d, k)
    # 4. Project
    return X @ components  # (n, k)

# Verify with sklearn
from sklearn.decomposition import PCA
pca = PCA(n_components=2)
X_transformed = pca.fit_transform(X)
```

**Example 2: Matrix Factorization for Recommendations**
```python
# R = U Vᵀ  where R ∈ ℝᵐˣⁿ (users × items)
# U ∈ ℝᵐˣᵏ, V ∈ ℝⁿˣᵏ
# Loss: ||R - UVᵀ||²_F + λ(||U||² + ||V||²)

import numpy as np

def matrix_factorization(R, k=10, lr=0.01, reg=0.01, epochs=100):
    m, n = R.shape
    U = np.random.normal(0, 0.1, (m, k))
    V = np.random.normal(0, 0.1, (n, k))
    
    mask = R > 0  # observed entries
    
    for epoch in range(epochs):
        pred = U @ V.T
        error = (pred - R) * mask
        
        # Gradients
        dU = (error @ V) + reg * U
        dV = (error.T @ U) + reg * V
        
        U -= lr * dU
        V -= lr * dV
    
    return U, V
```

---

## 3. PROBABILITY & STATISTICS

### WHY? (Problem Statement)
**ML = Statistical Inference.** Every model makes probabilistic assumptions:
- **Loss functions** = Negative log-likelihood
- **Regularization** = Prior distributions
- **Uncertainty quantification** = Prediction intervals
- **Hypothesis testing** = A/B tests, feature significance

### HOW? (Core Concepts)

#### Probability Basics
| Concept | Formula | ML Use |
|---------|---------|--------|
| **Conditional** | P(A\|B) = P(A∩B)/P(B) | Bayes classifiers |
| **Bayes Theorem** | P(A\|B) = P(B\|A)P(A)/P(B) | Naive Bayes, Bayesian inference |
| **Independence** | P(A∩B) = P(A)P(B) | Naive Bayes assumption |
| **Chain Rule** | P(A,B,C) = P(A)P(B\|A)P(C\|A,B) | Autoregressive models |

#### Random Variables & Distributions
| Type | Distribution | Parameters | ML Use |
|--------|-------------|------------|--------|
| **Discrete** | Bernoulli(p) | p | Binary classification |
| **Discrete** | Binomial(n, p) | n, p | Count data |
| **Discrete** | Categorical(π) | π (vector) | Multi-class classification |
| **Continuous** | Gaussian(μ, σ²) | μ, σ² | Regression, GMM, VAE |
| **Continuous** | Exponential(λ) | λ | Survival analysis |
| **Continuous** | Beta(α, β) | α, β | Priors for probabilities |
| **Continuous** | Dirichlet(α) | α (vector) | Topic models, priors |

#### Expectation, Variance, Covariance
| Statistic | Formula | ML Use |
|-----------|---------|--------|
| **Expectation** | E[X] = Σ x P(x) | Expected loss, baseline |
| **Variance** | Var[X] = E[(X-μ)²] | Uncertainty, regularization |
| **Covariance** | Cov[X,Y] = E[(X-μₓ)(Y-μᵧ)] | Feature correlation |
| **Correlation** | ρ = Cov/(σₓσᵧ) | Feature selection |

#### Bayesian Inference
```math
Posterior ∝ Likelihood × Prior
P(θ|D) = P(D|θ) P(θ) / P(D)

# Conjugate Priors (analytical posteriors):
# Beta-Binomial, Gamma-Poisson, Normal-Normal

# Example: Coin flip with Beta prior
# Prior: Beta(α, β)
# Likelihood: Binomial(n, θ)
# Posterior: Beta(α + heads, β + tails)
```

#### Information Theory
| Measure | Formula | ML Use |
|---------|---------|--------|
| **Entropy** | H(X) = -Σ P(x) log P(x) | Uncertainty, decision trees |
| **Cross-Entropy** | H(P, Q) = -Σ P(x) log Q(x) | Classification loss |
| **KL Divergence** | D_KL(P||Q) = Σ P log(P/Q) | Variational inference, distillation |
| **Mutual Info** | I(X;Y) = H(X) - H(X\|Y) | Feature selection, IB |

---

## PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Vectorize operations** | Python loops over arrays | 100-1000x slower |
| **Use broadcasting** | Explicit tiling/repeating | Memory waste, bugs |
| **Check matrix shapes** | Assume shapes match | Silent shape bugs |
| **Use log-sum-exp** | Direct softmax | Numerical overflow |
| **Add ε to variance** | Divide by zero | NaN in normalization |
| **Use log-probabilities** | Multiply probabilities | Underflow |

---

## INTERVIEW QUESTIONS (Top 15)

### 1. **Conceptual**: "Explain the chain rule and how it enables backpropagation."

**Answer:**
```
CHAIN RULE: df/dx = df/du · du/dx (single var)
           ∂f/∂x = Σ (∂f/∂uᵢ) · (∂uᵢ/∂x) (multi-var)

BACKPROPAGATION:
Neural network = composition of functions:
f(x) = f_L ∘ f_{L-1} ∘ ... ∘ f_1 (x)

∂L/∂W₁ = ∂L/∂y · ∂y/∂h_L · ... · ∂h₂/∂h₁ · ∂h₁/∂W₁

COMPUTATIONAL GRAPH:
- Forward pass: Compute values, store intermediates
- Backward pass: Apply chain rule in reverse topological order
- AutoDiff frameworks (PyTorch, JAX) automate this

KEY INSIGHT: Chain rule lets us compute gradient of 
scalar loss w.r.t millions of parameters in O(N) time,
where N = number of operations in graph.
```

### 2. **Code**: "Implement Gradient Descent with Momentum from Scratch"

```python
import numpy as np

class SGDMomentum:
    def __init__(self, lr=0.01, momentum=0.9):
        self.lr = lr
        self.momentum = momentum
        self.velocity = None
    
    def update(self, params, grads):
        if self.velocity is None:
            self.velocity = [np.zeros_like(p) for p in params]
        
        for i, (param, grad) in enumerate(zip(params, grads)):
            self.velocity[i] = self.momentum * self.velocity[i] - self.lr * grad
            param += self.velocity[i]

# Nesterov Accelerated Gradient (NAG)
class NAG:
    def __init__(self, lr=0.01, momentum=0.9):
        self.lr = lr
        self.momentum = momentum
        self.velocity = None
    
    def update(self, params, grads):
        if self.velocity is None:
            self.velocity = [np.zeros_like(p) for p in params]
        
        for i, (param, grad) in enumerate(zip(params, grads)):
            # Lookahead position
            lookahead = param + self.momentum * self.velocity[i]
            # Compute gradient at lookahead (need forward pass here)
            # For true NAG, gradient must be computed at lookahead
            # Simplified version (gradient at current):
            self.velocity[i] = self.momentum * self.velocity[i] - self.lr * grad
            param += self.velocity[i]
```

### 3. **Design**: "Explain SVD and its use in Recommender Systems."

```
SVD: A = U Σ Vᵀ
- U (m×m): Left singular vectors (user embeddings)
- Σ (m×n): Singular values (importance of each dimension)
- Vᵀ (n×n): Right singular vectors (item embeddings)

RECOMMENDATION SYSTEM:
R (users × items) ≈ Uₖ Σₖ Vₖᵀ  (rank-k approximation)

TRAINING:
1. R observed entries only (sparse matrix)
2. Minimize: ||P_Ω(R - UVᵀ)||² + λ(||U||² + ||V||²)
3. Alternating Least Squares (ALS) or SGD

COLD START:
- New user: Map demographics → U embedding (side info)
- New item: Content features → V embedding (content-based)

SERVING:
- Precompute U, V
- User u: scores = U[u] @ V.T  (dot product with all items)
- Top-k via ANN (FAISS, Annoy) or full scan if n small
```

### 4. **Code**: "Implement PCA from Scratch using SVD"

```python
import numpy as np

class PCA:
    def __init__(self, n_components):
        self.n_components = n_components
        self.components_ = None
        self.mean_ = None
        self.explained_variance_ratio_ = None
    
    def fit(self, X):
        # Center data
        self.mean_ = X.mean(axis=0)
        X_centered = X - self.mean_
        
        # SVD
        U, S, Vt = np.linalg.svd(X_centered, full_matrices=False)
        
        # Components = top-k right singular vectors
        self.components_ = Vt[:self.n_components]
        
        # Explained variance
        total_var = np.sum(S**2)
        self.explained_variance_ratio_ = (S[:self.n_components]**2) / total_var
        
        return self
    
    def transform(self, X):
        X_centered = X - self.mean_
        return X_centered @ self.components_.T
    
    def fit_transform(self, X):
        self.fit(X)
        return self.transform(X)
    
    def inverse_transform(self, X_transformed):
        return X_transformed @ self.components_ + self.mean_

# Verify
from sklearn.decomposition import PCA as SklearnPCA
X = np.random.randn(100, 10)
pca1 = PCA(2).fit_transform(X)
pca2 = SklearnPCA(2).fit_transform(X)
print(np.allclose(np.abs(pca1), np.abs(pca2)))  # True (sign may flip)
```

### 5. **Code**: "Bayesian Linear Regression with Conjugate Prior"

```python
import numpy as np
from scipy.stats import norm, multivariate_normal

class BayesianLinearRegression:
    """
    Prior: w ~ N(0, α⁻¹I)
    Likelihood: y|X,w ~ N(Xw, β⁻¹I)
    Posterior: w|D ~ N(μₙ, Σₙ)
    """
    def __init__(self, alpha=1.0, beta=1.0):
        self.alpha = alpha  # Prior precision
        self.beta = beta    # Likelihood precision
        self.mu = None
        self.Sigma = None
    
    def fit(self, X, y):
        # Add bias
        X = np.column_stack([np.ones(len(X)), X])
        
        # Posterior covariance: Σₙ = (αI + βXᵀX)⁻¹
        self.Sigma = np.linalg.inv(
            self.alpha * np.eye(X.shape[1]) + self.beta * X.T @ X
        )
        # Posterior mean: μₙ = β Σₙ Xᵀ y
        self.mu = self.beta * self.Sigma @ X.T @ y
        return self
    
    def predict(self, X, return_std=False):
        X = np.column_stack([np.ones(len(X)), X])
        y_mean = X @ self.mu
        
        if return_std:
            # Predictive variance: 1/β + xᵀ Σₙ x
            y_var = 1/self.beta + np.sum(X @ self.Sigma * X, axis=1)
            return y_mean, np.sqrt(y_var)
        return y_mean
    
    def sample_weights(self, n_samples=1):
        return np.random.multivariate_normal(self.mu, self.Sigma, n_samples)

# Usage
blr = BayesianLinearRegression(alpha=1.0, beta=10.0)
blr.fit(X_train, y_train)
y_pred, y_std = blr.predict(X_test, return_std=True)
```

### 6. **Conceptual**: "Bias-Variance Tradeoff in Regularized Linear Regression"

```
RIDGE (L2):    min ||y - Xw||² + λ||w||²
LASSO (L1):    min ||y - Xw||² + λ||w||₁

EFFECTIVE DEGREES OF FREEDOM:
df(λ) = trace(X(XᵀX + λI)⁻¹Xᵀ)  for Ridge

λ → 0:  df → d (OLS, high variance, low bias)
λ → ∞:  df → 0 (w → 0, high bias, low variance)

OPTIMAL λ: Cross-validation minimizes test error
```

### 7. **Code**: "Implement Softmax with Numerical Stability"

```python
def softmax(x, axis=-1):
    """Numerically stable softmax"""
    # Subtract max for numerical stability
    x_max = np.max(x, axis=axis, keepdims=True)
    exp_x = np.exp(x - x_max)
    return exp_x / np.sum(exp_x, axis=axis, keepdims=True)

def cross_entropy_loss(y_true, y_pred, eps=1e-15):
    """Cross-entropy with clipping for numerical stability"""
    y_pred = np.clip(y_pred, eps, 1 - eps)
    return -np.mean(np.sum(y_true * np.log(y_pred), axis=-1))

def log_sum_exp(x, axis=-1):
    """Stable log-sum-exp"""
    x_max = np.max(x, axis=axis, keepdims=True)
    return x_max + np.log(np.sum(np.exp(x - x_max), axis=axis, keepdims=True))

# In PyTorch (built-in stable):
# F.log_softmax(x, dim=-1)
# F.cross_entropy(logits, targets)  # combines log_softmax + nll_loss
```

### 8. **Conceptual**: "When to use PCA vs Autoencoder?"

```
PCA:
✓ Linear dimensionality reduction
✓ Interpretable components
✓ Fast, deterministic, convex
✓ Works well when data lies near linear subspace
✗ Cannot capture non-linear manifolds

AUTOENCODER:
✓ Non-linear dimensionality reduction
✓ Can learn complex manifolds
✓ Can be regularized (sparse, denoising, variational)
✗ Non-convex, requires tuning
✗ Harder to interpret
✗ Needs more data

USE PCA WHEN:
- d < 1000, linear relationships dominate
- Need interpretable features
- Baseline before trying deep methods

USE AUTOENCODER WHEN:
- Complex non-linear structure (images, sequences)
- Need compression for downstream task
- Have sufficient data and compute
```

### 9. **Code**: "Matrix Factorization with Implicit Feedback (BPR)"

```python
import numpy as np

class BPR:
    """Bayesian Personalized Ranking for Implicit Feedback"""
    def __init__(self, n_users, n_items, k=64, lr=0.01, reg=0.01):
        self.U = np.random.normal(0, 0.1, (n_users, k))
        self.V = np.random.normal(0, 0.1, (n_items, k))
        self.lr = lr
        self.reg = reg
    
    def _sigmoid(self, x):
        return 1 / (1 + np.exp(-np.clip(x, -500, 500)))
    
    def train_step(self, user, pos_item, neg_item):
        # BPR loss: -ln σ(ûᵢvⱼ - ûᵢvₖ)
        u = self.U[user]
        v_pos = self.V[pos_item]
        v_neg = self.V[neg_item]
        
        x_uij = np.dot(u, v_pos - v_neg)
        sigmoid = self._sigmoid(x_uij)
        
        # Gradients
        grad = (1 - sigmoid) * (v_pos - v_neg)
        grad_U = -grad + self.reg * u
        grad_V_pos = grad + self.reg * v_pos
        grad_V_neg = -grad + self.reg * v_neg
        
        # Update
        self.U[user] -= self.lr * grad_U
        self.V[pos_item] -= self.lr * grad_V_pos
        self.V[neg_item] -= self.lr * grad_V_neg
    
    def predict(self, user, items):
        return self.U[user] @ self.V[items].T
```

### 9. **Code**: "Custom Loss Functions in PyTorch (Focal Loss)"

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class FocalLoss(nn.Module):
    """Focal Loss for dense object detection / imbalanced classification
    FL(p_t) = -α_t (1 - p_t)^γ log(p_t)
    """
    def __init__(self, alpha=0.25, gamma=2.0, reduction='mean'):
        super().__init__()
        self.alpha = alpha
        self.gamma = gamma
        self.reduction = reduction
    
    def forward(self, logits, targets):
        # logits: (N, C) or (N,), targets: (N,) with class indices
        ce_loss = F.cross_entropy(logits, targets, reduction='none')
        pt = torch.exp(-ce_loss)  # p_t
        focal_loss = self.alpha * (1 - pt) ** self.gamma * ce_loss
        
        if self.reduction == 'mean':
            return focal_loss.mean()
        elif self.reduction == 'sum':
            return focal_loss.sum()
        return focal_loss

class LabelSmoothingCrossEntropy(nn.Module):
    """Label Smoothing Regularization"""
    def __init__(self, smoothing=0.1):
        super().__init__()
        self.smoothing = smoothing
        self.confidence = 1.0 - smoothing
    
    def forward(self, logits, targets):
        log_probs = F.log_softmax(logits, dim=-1)
        nll_loss = -log_probs.gather(dim=-1, index=targets.unsqueeze(1)).squeeze(1)
        smooth_loss = -log_probs.mean(dim=-1)
        loss = self.confidence * nll_loss + self.smoothing * smooth_loss
        return loss.mean()

# Weighted Loss for Imbalanced Data
class WeightedCrossEntropy(nn.Module):
    def __init__(self, class_weights):
        super().__init__()
        self.register_buffer('class_weights', class_weights)
    
    def forward(self, logits, targets):
        return F.cross_entropy(logits, targets, weight=self.class_weights)
```

### 10. **Code**: "K-Fold Cross Validation with Stratification"

```python
from sklearn.model_selection import (
    KFold, StratifiedKFold, TimeSeriesSplit, 
    GroupKFold, LeaveOneGroupOut
)
from sklearn.base import clone
import numpy as np

def cross_validate_model(model, X, y, cv_strategy='stratified', n_splits=5):
    """Flexible cross-validation with different strategies"""
    
    if cv_strategy == 'stratified':
        cv = StratifiedKFold(n_splits=n_splits, shuffle=True, random_state=42)
    elif cv_strategy == 'kfold':
        cv = KFold(n_splits=n_splits, shuffle=True, random_state=42)
    elif cv_strategy == 'timeseries':
        cv = TimeSeriesSplit(n_splits=n_splits)
    elif cv_strategy == 'group':
        # Requires groups parameter
        cv = GroupKFold(n_splits=n_splits)
    else:
        raise ValueError(f"Unknown cv_strategy: {cv_strategy}")
    
    scores = []
    models = []
    
    for fold, (train_idx, val_idx) in enumerate(cv.split(X, y)):
        X_train, X_val = X[train_idx], X[val_idx]
        y_train, y_val = y[train_idx], y[val_idx]
        
        model = clone(model)
        model.fit(X_train, y_train)
        
        val_pred = model.predict(X_val)
        score = roc_auc_score(y_val, model.predict_proba(X_val)[:, 1])
        scores.append(score)
        models.append(model)
        
        print(f"Fold {fold+1}: AUC = {score:.4f}")
    
    print(f"Mean AUC: {np.mean(scores):.4f} ± {np.std(scores):.4f}")
    return np.mean(scores), models

# Nested CV for hyperparameter tuning
def nested_cv(model, param_grid, X, y, outer_cv=5, inner_cv=3):
    from sklearn.model_selection import GridSearchCV, cross_val_score
    
    outer = StratifiedKFold(n_splits=outer_cv, shuffle=True, random_state=42)
    inner = StratifiedKFold(n_splits=inner_cv, shuffle=True, random_state=42)
    
    clf = GridSearchCV(model, param_grid, cv=inner, scoring='roc_auc', n_jobs=-1)
    nested_scores = cross_val_score(clf, X, y, cv=outer, scoring='roc_auc')
    
    return nested_scores
```

### 11. **Code**: "Hyperparameter Optimization with Optuna"

```python
import optuna
from sklearn.model_selection import cross_val_score
import xgboost as xgb

def optimize_xgboost(X, y, n_trials=100, timeout=3600):
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
        scores = cross_val_score(
            model, X, y, 
            cv=StratifiedKFold(5, shuffle=True, random_state=42),
            scoring='roc_auc', n_jobs=-1
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

# Usage
best_params, best_score = optimize_xgboost(X_train, y_train, n_trials=200)
```

### 10. **Code**: "Data Leakage Prevention in Cross-Validation"

```python
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.feature_selection import SelectKBest, f_classif
from sklearn.model_selection import cross_val_score

# WRONG: Leakage!
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)  # Fits on ALL data!
selector = SelectKBest(f_classif, k=10)
X_selected = selector.fit_transform(X_scaled, y)
scores = cross_val_score(model, X_selected, y, cv=5)

# CORRECT: Pipeline ensures fitting only on training folds
pipeline = Pipeline([
    ('scaler', StandardScaler()),
    ('selector', SelectKBest(f_classif, k=10)),
    ('model', LogisticRegression())
)

scores = cross_val_score(pipeline, X, y, cv=StratifiedKFold(5), scoring='roc_auc')

# FOR TIME SERIES - NO RANDOM SHUFFLE
from sklearn.model_selection import TimeSeriesSplit
tscv = TimeSeriesSplit(n_splits=5)
scores = cross_val_score(pipeline, X, y, cv=tscv)

# GROUP K-FOLD (e.g., user-level split to prevent leakage)
from sklearn.model_selection import GroupKFold
groups = user_ids  # Each user's data stays together
gkf = GroupKFold(n_splits=5)
scores = cross_val_score(pipeline, X, y, cv=gkf.split(X, y, groups))
```

### 12. **Code**: "Evaluation Metrics Implementation"

```python
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score, f1_score,
    roc_auc_score, average_precision_score, confusion_matrix,
    precision_recall_curve, roc_curve, classification_report
)
import numpy as np

def evaluate_classifier(y_true, y_pred, y_prob=None):
    """Comprehensive classification evaluation"""
    results = {
        'accuracy': accuracy_score(y_true, y_pred),
        'precision': precision_score(y_true, y_pred, average='binary'),
        'recall': recall_score(y_true, y_pred, average='binary'),
        'f1': f1_score(y_true, y_pred, average='binary'),
    }
    
    if y_prob is not None:
        results['roc_auc'] = roc_auc_score(y_true, y_prob)
        results['pr_auc'] = average_precision_score(y_true, y_prob)
        
        # Optimal threshold by F1
        precisions, recalls, thresholds = precision_recall_curve(y_true, y_prob)
        f1_scores = 2 * precisions * recalls / (precisions + recalls + 1e-10)
        best_idx = np.argmax(f1_scores)
        results['optimal_threshold'] = thresholds[best_idx] if best_idx < len(thresholds) else 1.0
        results['best_f1'] = f1_scores[best_idx]
    
    cm = confusion_matrix(y_true, y_pred)
    results['confusion_matrix'] = cm
    results['tn'], results['fp'], results['fn'], results['tp'] = cm.ravel()
    
    return results

def regression_metrics(y_true, y_pred):
    from sklearn.metrics import mean_squared_error, mean_absolute_error, r2_score
    return {
        'mse': mean_squared_error(y_true, y_pred),
        'rmse': np.sqrt(mean_squared_error(y_true, y_pred)),
        'mae': mean_absolute_error(y_true, y_pred),
        'r2': r2_score(y_true, y_pred),
        'mape': np.mean(np.abs((y_true - y_pred) / (y_true + 1e-8))) * 100
    }

def ranking_metrics(y_true, y_pred, k=10):
    """NDCG, MAP, MRR for ranking"""
    from sklearn.metrics import ndcg_score
    # y_true: relevance scores, y_pred: predicted scores
    return {
        f'ndcg@{k}': ndcg_score([y_true], [y_pred], k=k),
    }
```

### 13. **Code**: "Probability Calibration"

```python
from sklearn.calibration import CalibratedClassifierCV, calibration_curve
from sklearn.isotonic import IsotonicRegression
from sklearn.linear_model import LogisticRegression
import matplotlib.pyplot as plt

def calibrate_classifier(model, X_train, y_train, X_val, y_val, method='isotonic'):
    """Calibrate predicted probabilities"""
    if method == 'isotonic':
        calibrator = CalibratedClassifierCV(model, method='isotonic', cv='prefit')
    elif method == 'platt':
        calibrator = CalibratedClassifierCV(model, method='sigmoid', cv='prefit')
    
    calibrator.fit(X_val, y_val)
    return calibrator

def plot_calibration_curve(y_true, y_prob, n_bins=10):
    """Plot reliability diagram"""
    prob_true, prob_pred = calibration_curve(y_true, y_prob, n_bins=n_bins)
    
    plt.figure(figsize=(6, 6))
    plt.plot([0, 1], [0, 1], 'k--', label='Perfectly calibrated')
    plt.plot(prob_pred, prob_true, 's-', label='Model')
    plt.xlabel('Mean Predicted Probability')
    plt.ylabel('Fraction of Positives')
    plt.title('Calibration Curve (Reliability Diagram)')
    plt.legend()
    plt.show()

# Expected Calibration Error (ECE)
def expected_calibration_error(y_true, y_prob, n_bins=10):
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
```

### 13. **Code**: "Time Series Cross-Validation"

```python
from sklearn.model_selection import TimeSeriesSplit
import numpy as np

def expanding_window_cv(model, X, y, n_splits=5, initial_train_size=None):
    """Expanding window CV for time series"""
    tscv = TimeSeriesSplit(n_splits=n_splits)
    scores = []
    
    for fold, (train_idx, val_idx) in enumerate(tscv.split(X)):
        # Optional: expanding window (not sliding)
        if initial_train_size:
            train_idx = train_idx[train_idx >= len(train_idx) - initial_train_size]
        
        X_train, X_val = X[train_idx], X[val_idx]
        y_train, y_val = y[train_idx], y[val_idx]
        
        model.fit(X_train, y_train)
        pred = model.predict(X_val)
        score = roc_auc_score(y_val, pred)
        scores.append(score)
        print(f"Fold {fold}: Train {len(train_idx)}, Val {len(val_idx)}, AUC={score:.4f}")
    
    return np.mean(scores), np.std(scores)

def walk_forward_validation(model, X, y, step=1, min_train_size=100):
    """Walk-forward validation (rolling forecast origin)"""
    n = len(X)
    scores = []
    
    for i in range(min_train_size, n, step):
        train_end = i
        test_end = min(i + step, n)
        
        X_train, y_train = X[:train_end], y[:train_end]
        X_test, y_test = X[train_end:test_end], y[train_end:test_end]
        
        model.fit(X_train, y_train)
        pred = model.predict(X_test)
        score = roc_auc_score(y_test, pred)
        scores.append(score)
    
    return scores

# Purged K-Fold for Financial Time Series (prevents leakage)
from mlfinlab.cross_validation import PurgedKFold
purged_cv = PurgedKFold(n_splits=5, pct_embargo=0.01)
```

### 14. **Code**: "Hyperparameter Optimization with Optuna + Pruning"

```python
import optuna
from sklearn.model_selection import cross_val_score
import lightgbm as lgb

def optimize_lgbm(X, y, n_trials=200, timeout=3600):
    def objective(trial):
        params = {
            'objective': 'binary',
            'metric': 'auc',
            'boosting_type': 'gbdt',
            'n_estimators': trial.suggest_int('n_estimators', 100, 3000),
            'max_depth': trial.suggest_int('max_depth', 3, 12),
            'learning_rate': trial.suggest_float('learning_rate', 0.01, 0.3, log=True),
            'num_leaves': trial.suggest_int('num_leaves', 16, 256),
            'min_child_samples': trial.suggest_int('min_child_samples', 5, 100),
            'subsample': trial.suggest_float('subsample', 0.5, 1.0),
            'colsample_bytree': trial.suggest_float('colsample_bytree', 0.5, 1.0),
            'reg_alpha': trial.suggest_float('reg_alpha', 1e-8, 10.0, log=True),
            'reg_lambda': trial.suggest_float('reg_lambda', 1e-8, 10.0, log=True),
            'min_split_gain': trial.suggest_float('min_split_gain', 0, 1.0),
            'verbose': -1,
            'n_jobs': -1,
            'random_state': 42,
        }
        
        # Pruning callback
        pruning_callback = optuna.integration.LightGBMPruningCallback(trial, 'auc')
        
        model = lgb.LGBMClassifier(**params)
        
        # Cross-validation with pruning
        scores = cross_val_score(
            model, X, y,
            cv=StratifiedKFold(5, shuffle=True, random_state=42),
            scoring='roc_auc',
            fit_params={'callbacks': [pruning_callback]},
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

### 15. **Quick Reference: When to Use Which Algorithm**

| Data Type | Problem | First Try | If Fails Try |
|-----------|---------|-----------|--------------|
| **Tabular, small (<10k)** | Classification | Logistic Reg + CV | XGBoost, Random Forest |
| **Tabular, small** | Regression | Ridge/ElasticNet | XGBoost, NN |
| **Tabular, large (>100k)** | Both | XGBoost/LightGBM/CatBoost | HistGradientBoosting |
| **Text** | Classification | TF-IDF + Logistic/LinearSVC | BERT/Transformers |
| **Images** | Classification | ResNet/EfficientNet (pretrained) | ViT, Custom CNN |
| **Sequences (short)** | Classification | LSTM/GRU | 1D-CNN, Transformer |
| **Sequences (long)** | Both | Transformer | LSTM with attention |
| **Recommendation** | Implicit feedback | Matrix Factorization / BPR | Two-Tower, LightGCN |
| **Anomaly Detection** | Unsupervised | Isolation Forest | Autoencoder, One-Class SVM |
| **Time Series** | Forecasting | Prophet / ARIMA | TFT, N-BEATS, LSTM |

---

## MATHEMATICAL NOTATION QUICK REFERENCE

| Symbol | Meaning |
|--------|---------|
| ‖x‖₂ | L2 norm = √(Σxᵢ²) |
| ‖x‖₁ | L1 norm = Σ|xᵢ| |
| ⟨x, y⟩ | Dot product = xᵀy |
| X ∈ ℝⁿˣᵈ | Matrix with n rows, d columns |
| Xᵀ | Transpose |
| X⁻¹ | Matrix inverse |
| Tr(X) | Trace = Σᵢ Xᵢᵢ |
| det(X) | Determinant |
| λ | Eigenvalue |
| v | Eigenvector |
| σ | Singular value |
| E[·] | Expectation |
| Var[·] | Variance |
| Cov[·,·] | Covariance |
| D_KL(P\|Q) | KL Divergence |
| H(X) | Entropy |
| I(X;Y) | Mutual Information |
| N(μ, Σ) | Multivariate Normal |
| Dir(α) | Dirichlet |
| Beta(α, β) | Beta |
| Bern(p) | Bernoulli |
| Bin(n, p) | Binomial |
| Cat(π) | Categorical |

---

## KEY FORMULAS CHEAT SHEET

### Gradient Descent Variants
```math
# Vanilla GD
θ_{t+1} = θ_t - η ∇L(θ_t)

# Momentum
v_{t+1} = γ v_t + η ∇L(θ_t)
θ_{t+1} = θ_t - v_{t+1}

# Nesterov
v_{t+1} = γ v_t + η ∇L(θ_t + γ v_t)
θ_{t+1} = θ_t - v_{t+1}

# Adam
m_t = β₁ m_{t-1} + (1-β₁) g_t
v_t = β₂ v_{t-1} + (1-β₂) g_t²
m̂ = m_t / (1-β₁ᵗ)
v̂ = v_t / (1-β₂ᵗ)
θ_{t+1} = θ_t - η · m̂ / (√v̂ + ε)
```

### Key Statistical Tests
| Test | When | Assumption |
|------|------|------------|
| **t-test** | Compare 2 means | Normal, equal variance |
| **ANOVA** | Compare >2 means | Normal, equal variance |
| **Chi-square** | Categorical independence | Expected count ≥ 5 |
| **Mann-Whitney** | 2 groups, non-normal | Ordinal/continuous |
| **Kruskal-Wallis** | >2 groups, non-normal | Ordinal/continuous |
| **KS test** | Distribution equality | Continuous |

### Information Criteria
```math
AIC = 2k - 2ln(L)        # Akaike
BIC = k ln(n) - 2ln(L)   # Bayesian (penalizes complexity more)
```