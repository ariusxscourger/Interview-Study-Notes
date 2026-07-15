# STATISTICS FOR MACHINE LEARNING
## Exam-Style Study Notes

**Roadmap Section:** Statistics
**Prerequisites:** Basic probability, calculus, linear algebra

---

## 1. DESCRIPTIVE STATISTICS

### WHY? (Problem Statement)
**First step in any ML project:** Understand your data before modeling. Descriptive statistics reveal:
- Data quality issues (missing, outliers, skew)
- Feature distributions and relationships
- Baseline performance (majority class, mean prediction)

### HOW? (Core Concepts)

#### Measures of Central Tendency
| Measure | Formula | When to Use | Robust? |
|---------|---------|-------------|---------|
| **Mean** | μ = Σxᵢ/n | Symmetric, no outliers | No |
| **Median** | Middle value (sorted) | Skewed, outliers | Yes |
| **Mode** | Most frequent | Categorical, multimodal | Yes |
| **Trimmed Mean** | Mean after removing extremes | Mild outliers | Moderate |
| **Weighted Mean** | Σwᵢxᵢ/Σwᵢ | Survey data, importance sampling | No |

```python
import numpy as np
from scipy import stats

data = np.array([1, 2, 2, 3, 4, 5, 100])  # outlier: 100

print(f"Mean: {np.mean(data):.2f}")           # 16.71 (skewed by outlier)
print(f"Median: {np.median(data):.2f}")       # 3.00 (robust)
print(f"Mode: {stats.mode(data, keepdims=True)[0]}")  # 2.00
print(f"Trimmed (10%): {stats.trim_mean(data, 0.1):.2f}")  # ~3.5
```

#### Measures of Dispersion
| Measure | Formula | Interpretation |
|---------|---------|----------------|
| **Variance** | σ² = Σ(xᵢ-μ)²/n | Average squared deviation |
| **Std Dev** | σ = √σ² | Same units as data |
| **MAD** | median(|xᵢ - median|) | Robust dispersion |
| **IQR** | Q3 - Q1 | Middle 50% spread |
| **Range** | max - min | Total spread (sensitive) |
| **CV** | σ/|μ| | Relative variability |

```python
data = np.array([1, 2, 3, 4, 5, 100])

print(f"Std: {np.std(data):.2f}")              # 36.14 (inflated)
print(f"IQR: {np.percentile(data, 75) - np.percentile(data, 25):.2f}")  # 2.00
print(f"MAD: {stats.median_abs_deviation(data):.2f}")  # 1.48 (robust)
print(f"CV: {np.std(data)/abs(np.mean(data)):.2f}")     # 2.16 (high variability)
```

#### Shape of Distribution
| Measure | Formula | Interpretation |
|---------|---------|----------------|
| **Skewness** | E[(X-μ)³]/σ³ | >0: right tail, <0: left tail |
| **Kurtosis** | E[(X-μ)⁴]/σ⁴ - 3 | >0: heavy tails, <0: light tails |

```python
from scipy.stats import skew, kurtosis

data = np.random.exponential(2, 1000)  # Right-skewed
print(f"Skewness: {skew(data):.2f}")    # ~2.0 (positive)
print(f"Kurtosis: {kurtosis(data):.2f}")  # ~6.0 (heavy tails)

# Normal distribution
normal = np.random.normal(0, 1, 1000)
print(f"Normal skew: {skew(normal):.2f}")    # ~0
print(f"Normal kurtosis: {kurtosis(normal):.2f}")  # ~0
```

---

## 2. INFERENTIAL STATISTICS

### WHY? (Problem Statement)
**Go beyond description.** Make claims about populations from samples:
- Is model A better than model B?
- Is feature X significant?
- What's the confidence in our estimate?

### HOW? (Core Concepts)

#### Sampling Distributions & Central Limit Theorem
```math
# CLT: Sample mean distribution approaches Normal
X̄ ~ N(μ, σ²/n)  for large n (n ≥ 30 typically)

# Standard Error
SE = σ/√n  (known σ)
SE = s/√n  (estimated from sample)
```

```python
import numpy as np
from scipy import stats

# Demonstrate CLT
population = np.random.exponential(2, 100000)  # Non-normal
sample_means = []
for _ in range(1000):
    sample = np.random.choice(population, 50, replace=False)
    sample_means.append(np.mean(sample))

print(f"Population mean: {np.mean(population):.3f}")
print(f"Sampling dist mean: {np.mean(sample_means):.3f}")
print(f"Population std: {np.std(population):.3f}")
print(f"Sampling dist std (SE): {np.std(sample_means):.3f}")
print(f"Theoretical SE: {np.std(population)/np.sqrt(50):.3f}")
```

#### Confidence Intervals
| Parameter | CI Formula | When |
|-----------|------------|------|
| **Mean (σ known)** | X̄ ± z*σ/√n | σ known, n large |
| **Mean (σ unknown)** | X̄ ± t*s/√n | σ unknown, n < 30 |
| **Proportion** | p̂ ± z*√(p̂(1-p̂)/n) | n*p̂ ≥ 10, n(1-p̂) ≥ 10 |
| **Difference of Means** | (X̄₁-X̄₂) ± t*√(s₁²/n₁ + s₂²/n₂) | Independent samples |

```python
import scipy.stats as stats

# Mean CI (t-distribution, unknown variance)
data = np.array([4.5, 5.2, 4.8, 5.1, 4.9, 5.3, 4.7])
n = len(data)
mean = np.mean(data)
se = stats.sem(data)  # std/√n
ci_95 = stats.t.interval(0.95, df=n-1, loc=mean, scale=se)
print(f"95% CI: ({ci_95[0]:.3f}, {ci_95[1]:.3f})")

# Proportion CI (Wilson score interval - better for small n)
from statsmodels.stats.proportion import proportion_confint
ci = proportion_confint(count=45, nobs=100, alpha=0.05, method='wilson')
print(f"Proportion 95% CI: ({ci[0]:.3f}, {ci[1]:.3f})")

# Difference of means CI
group_a = np.random.normal(5, 1, 30)
group_b = np.random.normal(4.5, 1, 30)
# Welch's t-test CI (unequal variances)
t_stat, p_val = stats.ttest_ind(group_a, group_b, equal_var=False)
print(f"t-stat: {t_stat:.3f}, p-value: {p_val:.4f}")
```

#### Hypothesis Testing Framework
```mermaid
H₀ (Null): No effect / no difference
H₁ (Alternative): Effect / difference exists
    │
    ▼
Choose α (typically 0.05)
    │
    ▼
Compute test statistic & p-value
    │
    ▼
p < α → Reject H₀
p ≥ α → Fail to reject H₀
```

| Test | When | Assumptions |
|------|------|-------------|
| **Z-test** | Mean, σ known, n large | Normal, σ known |
| **t-test (1-sample)** | Mean vs constant, σ unknown | Normal (n<30), or n≥30 |
| **t-test (2-sample)** | Compare 2 means | Normal, equal/unequal variance |
| **Paired t-test** | Before/after, matched pairs | Normal differences |
| **ANOVA** | >2 group means | Normal, equal variance |
| **Chi-square** | Categorical independence | Expected count ≥ 5 |
| **Mann-Whitney U** | 2 groups, non-normal | Ordinal/continuous |
| **Kruskal-Wallis** | >2 groups, non-normal | Ordinal/continuous |

```python
from scipy import stats

# One-sample t-test
sample = np.random.normal(5.2, 1, 25)
t_stat, p_val = stats.ttest_1samp(sample, 5.0)
print(f"One-sample t: t={t_stat:.3f}, p={p_val:.4f}")

# Two-sample t-test (Welch's - unequal variance)
group_a = np.random.normal(5, 1, 30)
group_b = np.random.normal(5.5, 1.2, 30)
t_stat, p_val = stats.ttest_ind(group_a, group_b, equal_var=False)
print(f"Welch's t: t={t_stat:.3f}, p={p_val:.4f}")

# Paired t-test (before/after)
before = np.random.normal(10, 2, 20)
after = before + np.random.normal(0.5, 1, 20)  # improvement
t_stat, p_val = stats.ttest_rel(before, after)
print(f"Paired t: t={t_stat:.3f}, p={p_val:.4f}")

# Chi-square test
observed = np.array([[50, 30], [20, 40]])
chi2, p, dof, expected = stats.chi2_contingency(observed)
print(f"Chi-square: χ²={chi2:.3f}, p={p:.4f}")

# Non-parametric alternatives
# Mann-Whitney U (Mann-Whitney U test)
u_stat, p_val = stats.mannwhitneyu(group_a, group_b, alternative='two-sided')
print(f"Mann-Whitney U: U={u_stat:.1f}, p={p_val:.4f}")

# Kruskal-Wallis (non-parametric ANOVA)
groups = [np.random.normal(5, 1, 20), np.random.normal(5.5, 1, 20), np.random.normal(6, 1, 20)]
h_stat, p_val = stats.kruskal(*groups)
print(f"Kruskal-Wallis: H={h_stat:.3f}, p={p_val:.4f}")
```

---

## 3. PROBABILITY DISTRIBUTIONS

### WHY? (Problem Statement)
**Models assume distributions.** Choosing the right distribution affects:
- Loss function selection
- Regularization interpretation
- Uncertainty quantification
- Sampling strategies

### HOW? (Key Distributions)

#### Discrete Distributions
| Distribution | PMF | Parameters | ML Use |
|--------------|-----|------------|--------|
| **Bernoulli** | pˣ(1-p)¹⁻ˣ | p | Binary classification |
| **Binomial** | C(n,x)pˣ(1-p)ⁿ⁻ˣ | n, p | Count data, A/B tests |
| **Categorical** | ∏πᵢˣⁱ | π (vector) | Multi-class classification |
| **Poisson** | λˣe⁻ˣ/x! | λ | Count data, rate events |

```python
from scipy.stats import bernoulli, binom, poisson, multinomial

# Bernoulli
p = 0.3
print(f"P(X=1)={bernoulli.pmf(1, p):.3f}, P(X=0)={bernoulli.pmf(0, p):.3f}")

# Binomial: n trials, success prob p
n, p = 10, 0.3
print(f"P(X=3)={binom.pmf(3, n, p):.3f}")
print(f"E[X]={n*p:.2f}, Var={n*p*(1-p):.2f}")

# Poisson: rate λ
lam = 2.5
print(f"P(X=3)={poisson.pmf(3, lam):.3f}")
print(f"E[X]=Var={lam}")

# Multinomial
probs = [0.2, 0.3, 0.5]
print(multinomial.pmf([2, 1, 2], n=5, p=probs))
```

#### Continuous Distributions
| Distribution | PDF | Parameters | ML Use |
|--------------|-----|------------|--------|
| **Normal (Gaussian)** | (2πσ²)⁻¹ᐟ²exp(-(x-μ)²/2σ²) | μ, σ² | Regression, priors, noise |
| **Log-Normal** | (xσ√2π)⁻¹exp(-(ln x - μ)²/2σ²) | μ, σ² | Positive skewed data |
| **Exponential** | λe⁻ˡᵃᵐᵇᵈᵃˣ | λ | Survival, inter-arrival |
| **Gamma** | βᵃxᵃ⁻¹e⁻ᵝˣ/Γ(α) | α, β | Positive continuous, waiting times |
| **Beta** | xᵃ⁻¹(1-x)ᵇ⁻¹/B(α,β) | α, β | Probabilities, priors |
| **Dirichlet** | ∏xᵢᵅⁱ⁻¹/B(α) | α (vector) | Topic models, priors |
| **Student's t** | Γ((ν+1)/2)/(√νπ Γ(ν/2)) (1+t²/ν)⁻⁽ᵛ⁺¹⁾/² | ν | Robust regression, small n |

```python
from scipy.stats import norm, lognorm, expon, gamma, beta, t, multivariate_normal

# Normal
mu, sigma = 0, 1
print(f"PDF(0)={norm.pdf(0, mu, sigma):.3f}")
print(f"CDF(1.96)={norm.cdf(1.96, mu, sigma):.3f}")
print(f"95% interval: {norm.interval(0.95, mu, sigma)}")

# Multivariate Normal
mean = np.array([0, 0])
cov = np.array([[1, 0.5], [0.5, 1]])
mvn = multivariate_normal(mean, cov)
print(f"PDF([0,0])={mvn.pdf([0,0]):.3f}")
samples = mvn.rvs(1000)
print(f"Sample corr: {np.corrcoef(samples.T)[0,1]:.3f}")

# Beta (for probabilities)
alpha, beta = 2, 5
print(f"Beta(2,5) mean={alpha/(alpha+beta):.2f}")
```

---

## 4. STATISTICAL LEARNING THEORY

### WHY? (Problem Statement)
**Why does ML generalize?** Understanding the theory behind:
- Generalization bounds
- Model selection
- Overfitting prevention

### HOW? (Core Concepts)

#### PAC Learning (Probably Approximately Correct)
```math
# With probability 1-δ:
R(h) ≤ R̂(h) + √( (VC(H) log(n/VC(H)) + log(1/δ)) / n )

# Where:
# R(h) = True risk (expected loss)
# R̂(h) = Empirical risk (training loss)
# VC(H) = VC dimension (capacity of hypothesis class)
# n = sample size
# δ = confidence parameter
```

#### Bias-Variance Decomposition
```math
E[(y - f̂(x))²] = Bias²[f̂(x)] + Var[f̂(x)] + σ²

Bias = E[f̂(x)] - f(x)          # Systematic error
Variance = E[(f̂(x) - E[f̂(x)])²]  # Sensitivity to training data
Irreducible = σ²                 # Noise in data
```

#### Regularization as Prior
```math
# Ridge (L2): Gaussian prior w ~ N(0, 1/λ)
# Lasso (L1): Laplace prior w ~ Laplace(0, 1/λ)
# Elastic Net: Mixture

# MAP Estimate:
w_MAP = argmax_w P(w|D) = argmax_w P(D|w)P(w)
      = argmin_w -log P(D|w) - log P(w)
      = argmin_w Loss(w) + λ·Regularizer(w)
```

---

## PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Visualize first** | Jump to modeling | Miss skew, outliers, multimodality |
| **Check assumptions** | Blindly apply tests | Wrong p-values, wrong conclusions |
| **Effect sizes** | Only p-values | Statistical ≠ practical significance |
| **Confidence intervals** | Point estimates only | No uncertainty quantification |
| **Multiple testing correction** | Many tests, no correction | False discoveries (FWER/FDR) |
| **Power analysis** | No sample size justification | Underpowered studies |
| **Bayesian thinking** | Only frequentist | Miss prior knowledge, uncertainty |

---

## INTERVIEW QUESTIONS (Top 15)

### 1. **Conceptual**: "When would you use median instead of mean?"

```
MEDIAN WHEN:
✓ Skewed distributions (income, house prices)
✓ Outliers present (sensor errors, fraud)
✓ Ordinal data (survey ratings)
✓ Small samples with potential outliers

MEAN WHEN:
✓ Symmetric distributions (heights, measurement errors)
✓ No outliers, clean data
✓ Need for algebraic manipulation (linearity of expectation)
✓ Comparing group means (ANOVA, t-tests assume normality of means)

EXAMPLE:
House prices: [200k, 220k, 210k, 230k, 2M]
Mean = 572k (misleading)
Median = 220k (representative)
```

### 2. **Code**: "Implement bootstrap confidence interval from scratch."

```python
import numpy as np
from scipy import stats

def bootstrap_ci(data, statistic=np.mean, n_bootstrap=10000, confidence=0.95):
    """Bootstrap confidence interval (percentile method)"""
    n = len(data)
    bootstrap_stats = []
    
    for _ in range(n_bootstrap):
        # Resample with replacement
        sample = np.random.choice(data, size=n, replace=True)
        bootstrap_stats.append(statistic(sample))
    
    alpha = (1 - confidence) / 2
    lower = np.percentile(bootstrap_stats, 100 * alpha)
    upper = np.percentile(bootstrap_stats, 100 * (1 - alpha))
    
    return lower, upper, np.mean(bootstrap_stats)

# Bootstrap CI for median (non-normal data)
data = np.random.exponential(2, 100)
lower, upper, mean_est = bootstrap_ci(data, statistic=np.median, n_bootstrap=5000)
print(f"Median bootstrap CI: ({lower:.3f}, {upper:.3f})")

# Compare with t-interval (assumes normality - wrong for exponential!)
mean = np.mean(data)
se = stats.sem(data)
ci_t = stats.t.interval(0.95, df=len(data)-1, loc=np.mean(data), scale=stats.sem(data))
print(f"t-interval (wrong!): ({ci[0]:.3f}, {ci[1]:.3f})")

# Bootstrap is correct here because it doesn't assume normality
```

### 3. **Design**: "How would you A/B test a new ranking model?"

```
A/B TEST DESIGN:

1. HYPOTHESIS:
   H₀: New model CTR ≤ Old model CTR
   H₁: New model CTR > Old model CTR

2. METRICS:
   Primary: CTR (Click-Through Rate)
   Secondary: Revenue per session, dwell time, conversion rate

3. SAMPLE SIZE CALCULATION:
   - Baseline CTR: 5%
   - Minimum detectable effect: 0.5% (absolute)
   - α = 0.05, Power = 0.8
   - Using proportions formula:
   n = (Z_{α/2} + Z_β)² * (p₁(1-p₁) + p₂(1-p₂)) / (p₁ - p₂)²
   ≈ 30,000 per group

4. RANDOMIZATION:
   - User-level randomization (not session!)
   - Hash user_id → consistent assignment
   - Check balance: demographics, historical activity

4. ANALYSIS:
   - Intention-to-treat (ITT)
   - Chi-square / proportion test
   - Segment analysis (new vs returning users)

5. STOPPING RULES:
   - Minimum sample size reached
   - Sequential testing (O'Brien-Fleming boundaries)
   - Futility stopping

5. COMMON PITFALLS:
✗ Peeking at results early
✗ Not accounting for multiple variants
✗ Ignoring seasonality/day-of-week
✗ Novelty/primacy effects
```

### 4. **Code**: "Implement hypothesis test for two proportions."

```python
from scipy import stats
import numpy as np

def two_proportion_ztest(p1, n1, p2, n2, alternative='two-sided'):
    """Two-proportion z-test with pooled standard error"""
    # Pooled proportion
    p_pool = (p1 * n1 + p2 * n2) / (n1 + n2)
    
    # Standard error under H0
    se = np.sqrt(p_pool * (1 - p_pool) * (1/n1 + 1/n2))
    
    # Z-statistic
    z = (p1 - p2) / se
    
    # P-value
    if alternative == 'two-sided':
        p = 2 * (1 - stats.norm.cdf(abs(z)))
    elif alternative == 'greater':
        p = 1 - stats.norm.cdf(z)
    elif alternative == 'less':
        p = stats.norm.cdf(z)
    
    # Confidence interval (unpooled)
    se_unpooled = np.sqrt(p1*(1-p1)/n1 + p2*(1-p2)/n2)
    margin = stats.norm.ppf(0.975) * se_unpooled
    ci = (p1 - p2 - margin, p1 - p2 + margin)
    
    return z, p, ci

# Example: A/B test
p1, n1 = 0.052, 50000  # Control
p2, n2 = 0.055, 50000  # Treatment
z, p, ci = two_proportion_ztest(p1, n1, p2, n2)
print(f"z={z:.3f}, p={p:.4f}, 95% CI=({ci[0]:.4f}, {ci[1]:.4f})")

# Use statsmodels for more features
from statsmodels.stats.proportion import proportions_ztest, proportion_confint_diff
count = np.array([p1*n1, p2*n2])
nobs = np.array([n1, n2])
stat, p = proportions_ztest(count, nobs, alternative='larger')
print(f"One-sided z={stat:.3f}, p={p:.4f}")
```

### 5. **Conceptual**: "Explain the difference between correlation and causation in ML features."

```
CORRELATION ≠ CAUSATION:

EXAMPLE: Ice cream sales correlate with drowning deaths
- Confounder: Temperature (hot weather → both increase)
- Ice cream doesn't cause drowning

IN ML:
- Feature X correlates with target Y
- But changing X may not change Y
- Spurious correlations from confounders, selection bias

CAUSAL INFERENCE TOOLS:
1. Randomized experiments (A/B tests) - Gold standard
2. Instrumental variables (IV) - Natural experiments
3. Regression discontinuity - Threshold-based assignment
4. Propensity score matching - Observational adjustment
5. Causal graphs (DAGs) - Explicit assumptions

ML PRACTICE:
- Don't assume feature importance = causal importance
- Be careful with proxy features (zip code → race)
- Test interventions, not just predictions
- Monitor for distribution shift (covariate shift = P(X) changes)
```

### 5. **Code**: "Implement permutation test for feature importance."

```python
import numpy as np
from sklearn.base import clone
from sklearn.metrics import roc_auc_score

def permutation_importance(model, X, y, n_permutations=100, scoring=roc_auc_score):
    """Permutation feature importance (model-agnostic)"""
    baseline = scoring(model.predict_proba(X)[:, 1], y)
    n_features = X.shape[1]
    importances = np.zeros((n_features, n_permutations))
    
    for i in range(n_features):
        X_perm = X.copy()
        for _ in range(n_permutations):
            # Shuffle column i
            X_perm[:, i] = np.random.permutation(X_perm[:, i])
            score = scoring(model.predict_proba(X_perm)[:, 1], y)
            importances[i, _] = baseline - score
    
    return {
        'importances_mean': importances.mean(axis=1),
        'importances_std': importances.std(axis=1),
        'importances': importances
    }

# Usage with sklearn
from sklearn.ensemble import RandomForestClassifier
from sklearn.inspection import permutation_importance

model = RandomForestClassifier(n_estimators=100, random_state=42)
model.fit(X_train, y_train)

result = permutation_importance(model, X_test, y_test, 
                                n_repeats=30, random_state=42)

for i in result.importances_mean.argsort()[::-1]:
    if result.importances_mean[i] > 0:
        print(f"{feature_names[i]:<20} {result.importances_mean[i]:.4f} ± {result.importances_std[i]:.4f}")
```

### 6. **Conceptual**: "What is p-hacking and how do you prevent it?"

```
P-HACKING (Data Dredging):
- Testing many hypotheses, reporting only significant ones
- Stopping data collection when p < 0.05
- Excluding outliers post-hoc
- Changing outcome variables after seeing data
- Multiple comparisons without correction

EXAMPLES:
✗ "We tried 20 features, this one was significant!"
✗ "We removed 3 outliers and p became 0.04"
✗ "We stopped collecting data when p < 0.05"
✗ "Primary outcome wasn't significant, but secondary was!"

PREVENTION:
1. PRE-REGISTRATION:
   - Register hypothesis, analysis plan before data collection
   - ClinicalTrials.gov, OSF, AsPredicted

2. MULTIPLE TESTING CORRECTION:
   - Bonferroni: α_adj = α/m (conservative)
   - Benjamini-Hochberg: Control FDR (less conservative)
   - Holm-Bonferroni: Step-down, more power

3. SPLIT DATA:
   - Discovery set (exploratory)
   - Validation set (confirmatory)
   - Test set (final evaluation)

4. CORRECTION METHODS:
   ```python
   from statsmodels.stats.multitest import multipletests
   
   p_values = [0.001, 0.02, 0.03, 0.04, 0.06, 0.08]
   
   # Bonferroni
   reject_bonf, p_bonf, _, _ = multipletests(p_values, alpha=0.05, method='bonferroni')
   
   # Benjamini-Hochberg (FDR control)
   reject_bh, p_bh, _, _ = multipletests(p_values, alpha=0.05, method='fdr_bh')
   
   print(f"Bonferroni: {p_bonf}")
   print(f"BH: {p_bh}")
   ```

5. HOLDOUT SET:
   - Never touch test set until final evaluation
   - Lock it away
```

### 7. **Code**: "Implement statistical power analysis for sample size."

```python
import numpy as np
from scipy import stats
from statsmodels.stats.power import (
    TTestIndPower, TTestPower, NormalIndPower,
    GofChisquarePower, FTestPower
)

def sample_size_two_proportions(p1, p2, alpha=0.05, power=0.8, alternative='two-sided'):
    """Sample size for two-proportion test"""
    from statsmodels.stats.proportion import proportion_effectsize
    effect_size = proportion_effectsize(p1, p2)
    analysis = NormalIndPower()
    n = analysis.solve_power(effect_size=effect_size, nobs1=None, 
                             alpha=alpha, power=power, ratio=1, 
                             alternative=alternative)
    return int(np.ceil(n))

def sample_size_two_means(delta, std, alpha=0.05, power=0.8, alternative='two-sided'):
    """Sample size for two-sample t-test"""
    effect_size = delta / std
    analysis = TTestIndPower()
    n = analysis.solve_power(effect_size=effect_size, nobs1=None,
                             alpha=alpha, power=power, ratio=1,
                             alternative=alternative)
    return int(np.ceil(n))

def sample_size_paired_means(delta, std_diff, alpha=0.05, power=0.8):
    """Sample size for paired t-test"""
    effect_size = delta / std_diff
    analysis = TTestPower()
    n = analysis.solve_power(effect_size=effect_size, nobs=None,
                             alpha=alpha, power=power, alternative='two-sided')
    return int(np.ceil(n))

# Examples
print(f"Proportions (5% vs 5.5%): n={sample_size_two_proportions(0.05, 0.055)} per group")
print(f"Means (δ=0.5, σ=1): n={sample_size_two_means(0.5, 1.0)} per group")
print(f"Paired (δ=0.5, σ_diff=1): n={sample_size_paired_means(0.5, 1.0)}")

# Power curve
import matplotlib.pyplot as plt
effect_sizes = np.linspace(0.1, 0.8, 20)
powers = [TTestIndPower().power(es, nobs1=100, alpha=0.05) for es in effect_sizes]
plt.plot(effect_sizes, powers)
plt.xlabel('Effect Size (Cohen\'s d)')
plt.ylabel('Power')
plt.title('Power Curve (n=100 per group)')
plt.grid(True)
plt.show()
```

### 8. **Conceptual**: "Explain the difference between parametric and non-parametric tests."

```
PARAMETRIC TESTS:
- Assume distribution (usually Normal)
- Test parameters (mean, variance)
- More powerful when assumptions met
- Examples: t-test, ANOVA, Pearson correlation, Linear regression

NON-PARAMETRIC TESTS:
- Few/no distribution assumptions
- Test medians/ranks/distributions
- Less powerful when parametric assumptions hold
- Robust to outliers, skewed data
- Examples: Mann-Whitney, Wilcoxon, Kruskal-Wallis, Spearman

WHEN TO USE NON-PARAMETRIC:
✓ Small samples (n < 30) - can't verify normality
✓ Ordinal data (Likert scales, rankings)
✓ Severe outliers that can't be removed
✓ Severely skewed distributions
✓ Heteroscedasticity (unequal variances)

EQUIVALENT PAIRS:
| Parametric | Non-parametric |
|------------|----------------|
| 1-sample t-test | Wilcoxon signed-rank |
| 2-sample t-test | Mann-Whitney U |
| Paired t-test | Wilcoxon signed-rank |
| ANOVA | Kruskal-Wallis |
| Pearson r | Spearman ρ |

POWER COMPARISON:
- Normal data: Parametric ~5% more powerful
- Heavy-tailed: Non-parametric MUCH more powerful
- n → ∞: Asymptotic relative efficiency (ARE) of Mann-Whitney vs t-test = 3/π ≈ 0.95
```

### 8. **Code**: "Multiple testing correction implementation."

```python
from statsmodels.stats.multitest import multipletests
import numpy as np

def demo_multiple_testing():
    # Simulate 20 null hypotheses (all true null)
    np.random.seed(42)
    p_values = np.random.uniform(0, 1, 20)
    p_values[0] = 0.001  # One "significant" by chance
    
    print("Raw p-values:", np.round(np.sort(p_values), 4))
    
    # Bonferroni (FWER control)
    reject_bonf, p_bonf, _, _ = multipletests(p_values, alpha=0.05, method='bonferroni')
    print(f"Bonferroni significant: {np.sum(reject_bonf)}")
    
    # Holm-Bonferroni (step-down, more power)
    reject_holm, p_holm, _, _ = multipletests(p_values, alpha=0.05, method='holm')
    print(f"Holm-Bonferroni: {np.sum(reject_holm)}")
    
    # Benjamini-Hochberg (FDR control)
    reject_bh, p_bh, _, _ = multipletests(p_values, alpha=0.05, method='fdr_bh')
    print(f"Benjamini-Hochberg: {np.sum(reject_bh)}")
    
    # Benjamini-Yekutieli (FDR under dependence)
    reject_by, p_by, _, _ = multipletests(p_values, alpha=0.05, method='fdr_by')
    print(f"BY: {np.sum(reject_by)}")
    
    print(f"\nAdjusted p-values:")
    for method, p_adj in zip(['Bonf', 'Holm', 'BH', 'BY'], 
                              [p_bonf, p_holm, p_bh, p_by]):
        print(f"  {method}: {np.round(p_adj, 4)}")

demo_multiple_testing()

# When to use which:
# Bonferroni: Few tests, need strict FWER control
# Holm: Better power than Bonferroni, still FWER
# BH: Many tests, FDR control acceptable (genomics, A/B testing)
# BY: Tests are dependent (conservative)
```

### 9. **Code**: "Bootstrap vs parametric confidence intervals."

```python
import numpy as np
from scipy import stats

def bootstrap_ci(data, statistic=np.mean, n_boot=10000, confidence=0.95, method='percentile'):
    """Bootstrap confidence intervals"""
    n = len(data)
    stats_boot = []
    
    for _ in range(n_boot):
        sample = np.random.choice(data, size=n, replace=True)
        stats_boot.append(statistic(sample))
    
    stats_boot = np.array(stats_boot)
    alpha = (1 - confidence) / 2
    
    if method == 'percentile':
        lower = np.percentile(stats_boot, 100 * alpha)
        upper = np.percentile(stats_boot, 100 * (1 - alpha))
    elif method == 'bca':  # Bias-corrected and accelerated
        # BCa requires jackknife - simplified version
        z0 = stats.norm.ppf(np.mean(stats_boot < statistic(data)))
        # Simplified - full BCa needs jackknife
        lower = np.percentile(stats_boot, 100 * stats.norm.cdf(z0 - 1.96))
        upper = np.percentile(stats_boot, 100 * stats.norm.cdf(z0 + 1.96))
    else:
        # Basic bootstrap
        theta = statistic(data)
        lower = 2*theta - np.percentile(stats_boot, 100 * (1 - alpha))
        upper = 2*theta - np.percentile(stats_boot, 100 * alpha)
    
    return lower, upper

# Compare methods
np.random.seed(42)
data = np.random.exponential(2, 50)  # Skewed data

# Parametric (assumes normality - WRONG for exponential!)
mean = np.mean(data)
se = stats.sem(data)
ci_param = stats.t.interval(0.95, df=len(data)-1, loc=mean, scale=se)

# Bootstrap (correct for any distribution)
ci_boot = bootstrap_ci(data, np.mean, method='percentile')
ci_boot_median = bootstrap_ci(data, np.median, method='percentile')

print(f"Parametric CI (mean): ({ci_param[0]:.3f}, {ci_param[1]:.3f})")
print(f"Bootstrap CI (mean): ({ci_boot[0]:.3f}, {ci_boot[1]:.3f})")
print(f"Bootstrap CI (median): ({ci_boot_median[0]:.3f}, {ci_boot_median[1]:.3f})")

# When to use bootstrap:
# - Non-normal data
# - Complex statistics (median, correlation, quantiles)
# - Small samples where CLT doesn't apply
# - When parametric assumptions are violated
```

### 9. **Code**: "Permutation test for A/B testing."

```python
import numpy as np
from scipy import stats

def permutation_test(group_a, group_b, statistic=np.mean, n_permutations=10000):
    """Permutation test for difference in means"""
    observed_diff = statistic(group_a) - statistic(group_b)
    combined = np.concatenate([group_a, group_b])
    n_a = len(group_a)
    
    permuted_diffs = []
    for _ in range(n_permutations):
        np.random.shuffle(combined)
        perm_a = combined[:n_a]
        perm_b = combined[n_a:]
        permuted_diffs.append(statistic(perm_a) - statistic(perm_b))
    
    permuted_diffs = np.array(permuted_diffs)
    
    # Two-sided p-value
    p_value = np.mean(np.abs(permuted_diffs) >= np.abs(observed_diff))
    
    # Confidence interval (percentile bootstrap of permutation dist)
    ci_lower = np.percentile(permuted_diffs, 2.5)
    ci_upper = np.percentile(permuted_diffs, 97.5)
    
    return {
        'observed_diff': observed_diff,
        'p_value': p_value,
        'ci': (ci_lower, ci_upper),
        'permuted_diffs': permuted_diffs
    }

# Example
np.random.seed(42)
control = np.random.normal(100, 15, 100)
treatment = np.random.normal(105, 15, 100)

result = permutation_test(control, treatment)
print(f"Observed diff: {result['observed_diff']:.3f}")
print(f"P-value: {result['p_value']:.4f}")
print(f"95% CI: ({result['ci'][0]:.3f}, {result['ci'][1]:.3f})")

# Compare with t-test
t_stat, p_val = stats.ttest_ind(control, treatment, equal_var=False)
print(f"t-test p-value: {p_val:.4f}")
```

### 10. **Conceptual**: "Explain Type I/II errors, power, and sample size tradeoffs."

```
HYPOTHESIS TESTING DECISION MATRIX:

                    | H₀ True    | H₀ False
        ────────────┼────────────┼────────────
      Reject H₀     │ Type I     │ Correct    │
                    │ (α)        │ (Power)    │
        ────────────┼────────────┼────────────
      Fail to Reject│ Correct    │ Type II    │
                    │ (1-α)      │ (β)        │

ERROR RATES:
- α (Type I): P(Reject H₀ | H₀ true) = False Positive Rate
- β (Type II): P(Fail to Reject | H₀ false) = False Negative Rate
- Power = 1 - β = P(Reject H₀ | H₀ false) = True Positive Rate

TRADEOFFS:
1. α vs β: Decreasing α increases β (for fixed n, effect)
2. Sample size n: Increasing n decreases both α and β
3. Effect size: Larger effect → higher power (easier to detect)
3. Variance: Lower variance → higher power

POWER ANALYSIS:
- Desired power: 0.8 (80%) standard
- α: 0.05 standard
- Effect size: Minimum meaningful difference
- Compute required n

TRADEOFFS IN PRACTICE:
- Medical trials: α = 0.01 or 0.001 (conservative)
- A/B testing: α = 0.05, power = 0.8
- Exploratory: Higher α acceptable, but correct for multiple testing
- Cost of Type I vs Type II determines α choice

POWER FORMULA (Two-sample t-test):
Power = P(t > t_{1-α/2, df} | δ ≠ 0)
Where non-centrality parameter λ = δ / (σ√(2/n))
```

### 10. **Code**: "Implement sequential testing for A/B tests."

```python
import numpy as np
from scipy import stats

class SequentialTest:
    """Sequential probability ratio test (SPRT) for A/B testing"""
    def __init__(self, p0, p1, alpha=0.05, beta=0.2):
        """
        p0: baseline conversion rate (null)
        p1: target conversion rate (alternative)
        alpha: Type I error
        beta: Type II error
        """
        self.p0 = p0
        self.p1 = p1
        self.alpha = alpha
        self.beta = beta
        
        # SPRT boundaries (log-likelihood ratio)
        self.A = np.log((1 - beta) / alpha)      # Upper bound (accept H1)
        self.B = np.log(beta / (1 - alpha))      # Lower bound (accept H0)
        
        self.log_lr = 0.0
        self.n = 0
        self.success_a = 0
        self.success_b = 0
    
    def update(self, success_a: bool, success_b: bool) -> str:
        """Update with new observations, return decision"""
        self.n += 1
        self.success_a += success_a
        self.success_b += success_b
        
        # Log-likelihood ratio for Bernoulli
        if success_a and success_b:
            self.log_lr += np.log(self.p1/self.p0) + np.log((1-self.p1)/(1-self.p0))
        elif success_a and not success_b:
            self.log_lr += np.log(self.p1/self.p0) + np.log((1-self.p1)/(1-self.p0))
        elif not success_a and success_b:
            self.log_lr += np.log((1-self.p1)/(1-self.p0)) + np.log(self.p1/self.p0)
        else:
            self.log_lr += np.log((1-self.p1)/(1-self.p0)) + np.log((1-self.p1)/(1-self.p0))
        
        if self.log_lr >= self.A:
            return 'reject_h0'  # Accept H1 (treatment better)
        elif self.log_lr <= self.B:
            return 'accept_h0'  # Accept H0 (no difference)
        else:
            return 'continue'
    
    def get_current_rates(self):
        return self.success_a / max(1, self.n), self.success_b / max(1, self.n)

# Usage
test = SequentialTest(p0=0.05, p1=0.055, alpha=0.05, beta=0.2)

# Simulate sequential arrivals
for i in range(10000):
    a = np.random.binomial(1, 0.055)  # True rate 5.5%
    b = np.random.binomial(1, 0.05)   # Control 5%
    decision = test.update(a, b)
    if decision != 'continue':
        print(f"Decision at n={test.n}: {decision}")
        print(f"Rates: A={test.success_a/test.n:.4f}, B={test.success_b/test.n:.4f}")
        break
```

### 11. **Code**: "Bayesian A/B testing."

```python
import numpy as np
from scipy.stats import beta
import matplotlib.pyplot as plt

class BayesianABTest:
    """Bayesian A/B testing with Beta-Binomial model"""
    def __init__(self, prior_alpha=1, prior_beta=1):
        self.prior_alpha = prior_alpha
        self.prior_beta = prior_beta
    
    def update(self, successes_a, trials_a, successes_b, trials_b):
        """Update posteriors with observed data"""
        # Posterior: Beta(alpha + successes, beta + failures)
        self.post_a = beta(prior_alpha + successes_a, 
                          prior_beta + trials_a - successes_a)
        self.post_b = beta(prior_alpha + successes_b,
                          prior_beta + trials_b - successes_b)
    
    def prob_b_better(self, n_samples=100000):
        """P(B better than A) via Monte Carlo"""
        samples_a = self.post_a.rvs(n_samples)
        samples_b = self.post_b.rvs(n_samples)
        return np.mean(samples_b > samples_a)
    
    def expected_loss(self, n_samples=100000):
        """Expected loss from choosing B when A is better (or vice versa)"""
        samples_a = self.post_a.rvs(n_samples)
        samples_b = self.post_b.rvs(n_samples)
        loss_a = np.maximum(samples_b - samples_a, 0).mean()
        loss_b = np.maximum(samples_a - samples_b, 0).mean()
        return loss_a, loss_b
    
    def credible_interval(self, alpha=0.95):
        """Credible intervals for each variant"""
        ci_a = self.post_a.interval(alpha)
        ci_b = self.post_b.interval(alpha)
        return ci_a, ci_b
    
    def plot_posteriors(self):
        x = np.linspace(0, 1, 1000)
        plt.plot(x, self.post_a.pdf(x), label='A (Control)')
        plt.plot(x, self.post_b.pdf(x), label='B (Treatment)')
        plt.fill_between(x, self.post_a.pdf(x), alpha=0.3)
        plt.fill_between(x, self.post_b.pdf(x), alpha=0.3)
        plt.xlabel('Conversion Rate')
        plt.ylabel('Density')
        plt.legend()
        plt.title('Posterior Distributions')
        plt.show()

# Example usage
np.random.seed(42)
test = BayesianABTest(prior_alpha=1, prior_beta=1)

# Observed data
success_a, trials_a = 250, 5000  # 5% conversion
success_b, trials_b = 280, 5000  # 5.6% conversion

test.update(success_a, trials_a, success_b, trials_b)

prob_better = test.prob_b_better()
loss_a, loss_b = test.expected_loss()
ci_a, ci_b = test.credible_interval()

print(f"P(B > A): {prob_better:.4f}")
print(f"Expected loss choosing B: {loss_b:.6f}")
print(f"Expected loss choosing A: {loss_a:.6f}")
print(f"A CI: {ci_a}")
print(f"B CI: {ci_b}")

# Decision rule: Choose B if expected loss < threshold
THRESHOLD = 0.0001  # 0.01% absolute loss tolerance
if loss_b < THRESHOLD:
    print("→ Implement variant B")
else:
    print("→ Keep variant A")
```

### 12. **Conceptual**: "What is the difference between confidence interval and credible interval?"

```
CONFIDENCE INTERVAL (Frequentist):
- 95% CI: If we repeated experiment infinitely, 95% of CIs would contain true parameter
- Parameter is FIXED, interval is RANDOM
- "We are 95% confident the true mean lies in [a,b]"
- Does NOT mean P(θ ∈ [a,b]) = 0.95
- Based on sampling distribution

CREDIBLE INTERVAL (Bayesian):
- 95% CrI: There is 95% probability the parameter lies in [a,b]
- Parameter is RANDOM, interval is FIXED
- "There is 95% probability θ ∈ [a,b]"
- Based on posterior distribution P(θ|data)

KEY DIFFERENCES:
| Aspect | Confidence Interval | Credible Interval |
|--------|---------------------|-------------------|
| Interpretation | Long-run frequency | Degree of belief |
| Parameter | Fixed unknown | Random variable |
| Probability | About interval | About parameter |
| Prior | None | Required |
| Computation | Analytic/asymptotic | MCMC/analytical |

EXAMPLE:
Coin flip: 3 heads, 2 tails

Frequentist (Wald):
p̂ = 0.6, SE = 0.22
95% CI: 0.6 ± 1.96×0.22 = [0.17, 1.0] (includes impossible >1!)

Bayesian (Beta(1,1) prior):
Posterior = Beta(4, 3)
95% CrI: [0.21, 0.89] (properly bounded)

WHEN THEY AGREE:
- Flat/uninformative prior + large n → CI ≈ CrI
- Normal likelihood + flat prior → Exact match

WHEN THEY DIFFER:
- Small samples
- Informative priors
- Constrained parameters (probabilities, variances)
- Skewed likelihoods

WHEN TO USE:
- Frequentist: Regulatory, standard reporting, no prior info
- Bayesian: Small data, prior knowledge, decision-making, sequential testing
```

### 11. **Code**: "Implement cross-validation with proper statistical testing."

```python
from sklearn.model_selection import (
    cross_val_score, cross_validate, StratifiedKFold, 
    cross_val_predict, KFold, GroupKFold
)
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.feature_selection import SelectKBest, f_classif
from sklearn.linear_model import LogisticRegression
from scipy import stats
import numpy as np

def cv_with_statistical_test(model, X, y, cv=5, scoring='roc_auc'):
    """Cross-validation with proper statistical testing"""
    
    # Stratified K-Fold for classification
    cv_strategy = StratifiedKFold(n_splits=cv, shuffle=True, random_state=42)
    
    # Cross-validate with multiple metrics
    scoring_dict = {
        'roc_auc': 'roc_auc',
        'accuracy': 'accuracy',
        'f1': 'f1',
        'precision': 'precision',
        'recall': 'recall'
    }
    
    cv_results = cross_validate(
        model, X, y, 
        cv=cv_strategy,
        scoring=scoring_dict,
        return_train_score=True,
        return_estimator=True
    )
    
    # Statistical significance test
    test_scores = cv_results['test_roc_auc']
    train_scores = cv_results['train_roc_auc']
    
    # Paired t-test: train vs test performance
    t_stat, p_val = stats.ttest_rel(train_scores, test_scores)
    
    print(f"Test AUC: {test_scores.mean():.4f} ± {test_scores.std():.4f}")
    print(f"Train AUC: {train_scores.mean():.4f} ± {train_scores.std():.4f}")
    print(f"Overfitting gap: {train_scores.mean() - test_scores.mean():.4f}")
    print(f"Paired t-test p-value: {p_val:.4f}")
    
    # 5x2 CV test (Dietterich) for comparing two models
    def five_two_cv(model1, model2, X, y):
        """5x2 CV test for comparing two classifiers"""
        from sklearn.model_selection import train_test_split
        diffs = []
        
        for i in range(5):
            # Split 1
            X1, X2, y1, y2 = train_test_split(X, y, test_size=0.5, random_state=i)
            
            # Train on 1, test on 2
            m1 = clone(model1).fit(X1, y1)
            m2 = clone(model2).fit(X1, y1)
            p1 = m1.predict_proba(X2)[:, 1]
            p2 = m2.predict_proba(X2)[:, 1]
            diff1 = roc_auc_score(y2, p1) - roc_auc_score(y2, p2)
            
            # Train on 2, test on 1
            m1 = clone(model1).fit(X2, y2)
            m2 = clone(model2).fit(X2, y2)
            p1 = m1.predict_proba(X1)[:, 1]
            p2 = m2.predict_proba(X1)[:, 1]
            diff2 = roc_auc_score(y1, p1) - roc_auc_score(y1, p2)
            
            diffs.extend([diff1, diff2])
        
        diffs = np.array(diffs)
        # Variance estimator for 5x2 CV
        var = np.sum((diffs - np.mean(diffs))**2) / 5
        t_stat = np.mean(diffs) / np.sqrt(var/5)
        p_val = 2 * (1 - stats.t.cdf(abs(t_stat), df=5))
        
        return t_stat, p_val
    
    return cv_results

# McNemar's test for comparing two classifiers on same test set
from statsmodels.stats.contingency_tables import mcnemar

def mcnemar_test(y_true, pred1, pred2):
    """Compare two classifiers on same test set"""
    # pred1, pred2 are binary predictions
    correct1 = (pred1 == y_true).astype(int)
    correct2 = (pred2 == y_true).astype(int)
    
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
    
    print(f"McNemar's test: chi2={result.statistic:.4f}, p={result.pvalue:.4f}")
    return result
```

### 12. **Conceptual**: "Explain the bias-variance tradeoff in model selection."

```
BIAS-VARIANCE DECOMPOSITION:
E[(y - f̂(x))²] = [E[f̂(x)] - f(x)]² + E[(f̂(x) - E[f̂(x)])²] + σ²
                      Bias²                    Variance      Noise

VISUAL:
Model Complexity →
      │
Error │    ┌─────────────────────────────────────┐
      │    │          Total Error                │
      │    │           / \                       │
      │    │          /   \                      │
      │    │         /     \                     │
      │    │        /       \                    │
      │    │       /         \                   │
      │    │      /           \                  │
      │    │     /             \                 │
      │    │    /               \                │
      │    │   /                 \               │
      │    │  /                   \              │
      │    │  / Variance          \ Bias²        │
      │    │ /                       \            │
      └────┴──────────────────────────────────────┘
         Low                          High
         Complexity                   Complexity

MODEL SELECTION STRATEGIES:

1. CROSS-VALIDATION:
   - K-fold CV estimates test error
   - Choose model with lowest CV error
   - Nested CV for unbiased selection

2. REGULARIZATION PATH:
   - Vary λ (regularization strength)
   - Plot train/val error vs λ
   - Choose λ at minimum val error

3. INFORMATION CRITERIA:
   AIC = 2k - 2ln(L)          # Penalizes parameters
   BIC = k ln(n) - 2ln(L)     # Stronger penalty
   Lower = better

4. LEARNING CURVES:
   - Plot train/val error vs training set size
   - High bias: Both errors high, converge
   - High variance: Gap between train/val

PRACTICAL GUIDELINES:
- High bias: Add features, decrease regularization, more complex model
- High variance: More data, regularization, simpler model, feature selection
- Both high: Wrong model class, bad features, data quality issues
```

### 13. **Code**: "Implement statistical comparison of multiple models."

```python
import numpy as np
from scipy import stats
from sklearn.model_selection import cross_val_score, StratifiedKFold
from sklearn.base import clone
from sklearn.datasets import make_classification

def compare_models(models, X, y, cv=5, metric='roc_auc'):
    """Statistical comparison of multiple models"""
    cv_strategy = StratifiedKFold(n_splits=cv, shuffle=True, random_state=42)
    results = {}
    
    for name, model in models.items():
        scores = cross_val_score(model, X, y, cv=cv_strategy, scoring=metric)
        results[name] = {
            'mean': np.mean(scores),
            'std': np.std(scores),
            'scores': scores
        }
        print(f"{name}: {np.mean(scores):.4f} ± {np.std(scores):.4f}")
    
    # Pairwise statistical tests (with Bonferroni correction)
    model_names = list(models.keys())
    n_comparisons = len(model_names) * (len(model_names) - 1) // 2
    alpha_adj = 0.05 / n_comparisons  # Bonferroni
    
    print(f"\nPairwise comparisons (α={alpha_adj:.4f} with Bonferroni):")
    for i, name1 in enumerate(model_names):
        for name2 in model_names[i+1:]:
            scores1 = results[name1]['scores']
            scores2 = results[name2]['scores']
            
            # Paired t-test (same CV folds)
            t_stat, p_val = stats.ttest_rel(scores1, scores2)
            
            significant = p_val < alpha_adj
            direction = ">" if np.mean(scores1) > np.mean(scores2) else "<"
            
            print(f"  {name1} vs {name2}: p={p_val:.4f} {'*' if significant else ''} "
                  f"({np.mean(scores1):.4f} {direction} {np.mean(scores2):.4f})")
    
    return results

# Nemenyi test (post-hoc for Friedman test)
def friedman_nemenyi(models, X, y, cv=5):
    """Non-parametric comparison of multiple classifiers"""
    from scipy.stats import friedmanchisquare
    
    cv_strategy = StratifiedKFold(n_splits=cv, shuffle=True, random_state=42)
    n_models = len(models)
    n_splits = cv
    
    # Rank models on each fold (1 = best)
    ranks = np.zeros((n_splits, n_models))
    model_names = list(models.keys())
    
    for fold_idx, (train_idx, val_idx) in enumerate(StratifiedKFold(cv, shuffle=True, random_state=42).split(X, y)):
        X_train, X_val = X[train_idx], X[val_idx]
        y_train, y_val = y[train_idx], y[val_idx]
        
        scores = []
        for name, model in models.items():
            model.fit(X[train_idx], y[train_idx])
            pred = model.predict_proba(X[val_idx])[:, 1]
            score = roc_auc_score(y[val_idx], pred)
            scores.append(score)
        
        # Rank (1 = best)
        ranks[fold_idx] = len(scores) - np.argsort(np.argsort(scores))
    
    # Friedman test
    friedman_stat, p_value = friedmanchisquare(*[ranks[:, i] for i in range(n_models)])
    print(f"Friedman test: χ²={friedman_stat:.3f}, p={p_value:.4f}")
    
    if p_value < 0.05:
        # Nemenyi post-hoc test
        avg_ranks = np.mean(ranks, axis=0)
        cd = 1.96 * np.sqrt(n_models * (n_models + 1) / (6 * n_splits))  # Nemenyi critical difference
        
        print(f"\nAverage ranks: {dict(zip(model_names, avg_ranks))}")
        print(f"Critical difference (CD): {cd:.3f}")
        
        for i, name1 in enumerate(model_names):
            for name2 in model_names[i+1:]:
                diff = abs(avg_ranks[i] - avg_ranks[list(model_names).index(name2)])
                significant = diff > cd
                print(f"  {name1} vs {name2}: rank diff={diff:.3f} {'*' if significant else ''}")
    
    return friedman_stat, p_value

# Example usage
from sklearn.ensemble import RandomForestClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.svm import SVC
from sklearn.neural_network import MLPClassifier

X, y = make_classification(n_samples=1000, n_features=20, n_informative=10, random_state=42)

models = {
    'Logistic': LogisticRegression(max_iter=1000, random_state=42),
    'RandomForest': RandomForestClassifier(n_estimators=100, random_state=42),
    'SVM': SVC(probability=True, random_state=42),
    'MLP': MLPClassifier(hidden_layer_sizes=(64, 32), random_state=42, max_iter=500)
}

results = compare_models(models, X, y)
friedman_nemenyi(models, X, y)
```

### 14. **Code**: "Outlier detection methods comparison."

```python
import numpy as np
from scipy import stats
from sklearn.ensemble import IsolationForest
from sklearn.neighbors import LocalOutlierFactor
from sklearn.svm import OneClassSVM
from sklearn.covariance import EllipticEnvelope
import matplotlib.pyplot as plt

def compare_outlier_detectors(X, contamination=0.1):
    """Compare different outlier detection methods"""
    detectors = {
        'Z-Score (>3)': lambda x: np.abs(stats.zscore(x)) > 3,
        'Modified Z-Score (>3.5)': lambda x: 0.6745 * np.abs(x - np.median(x)) / stats.median_abs_deviation(x) > 3.5,
        'IQR (1.5x)': lambda x: (x < np.percentile(x, 25) - 1.5 * (np.percentile(x, 75) - np.percentile(x, 25))) | (x > np.percentile(x, 75) + 1.5 * (np.percentile(x, 75) - np.percentile(x, 25))),
        'Isolation Forest': IsolationForest(contamination=contamination, random_state=42).fit_predict,
        'Local Outlier Factor': LocalOutlierFactor(n_neighbors=20, contamination=contamination).fit_predict,
        'One-Class SVM': OneClassSVM(nu=contamination, kernel='rbf', gamma='scale').fit_predict,
        'Elliptic Envelope': EllipticEnvelope(contamination=contamination, random_state=42).fit_predict,
    }
    
    results = {}
    for name, detector in detectors.items():
        try:
            if name in ['Isolation Forest', 'Local Outlier Factor', 'One-Class SVM', 'Elliptic Envelope']:
                preds = detector(X.reshape(-1, 1))
                outliers = preds == -1
            else:
                outliers = detector(X)
            results[name] = {
                'n_outliers': np.sum(outliers),
                'outlier_pct': np.mean(outliers) * 100,
                'indices': np.where(outliers)[0]
            }
        except Exception as e:
            results[name] = {'error': str(e)}
    
    return results

# Generate test data with outliers
np.random.seed(42)
normal_data = np.random.normal(0, 1, 1000)
outliers = np.random.normal(5, 1, 50)
data = np.concatenate([normal_data, outliers])

results = compare_outlier_detectors(data)
print("Outlier Detection Comparison:")
for name, result in results.items():
    if 'error' not in result:
        print(f"{name:<25} Outliers: {result['n_outliers']:3d} ({result['outlier_pct']:.1f}%)")
    else:
        print(f"{name:<25} Error: {result['error']}")

# Visualization
fig, axes = plt.subplots(2, 4, figsize=(16, 8))
axes = axes.ravel()

for i, (name, result) in enumerate(results.items()):
    if 'error' not in result:
        ax = axes[i]
        ax.hist(data, bins=50, alpha=0.7, density=True, label='Data')
        if len(result['indices']) > 0:
            ax.scatter(data[result['indices']], np.zeros_like(data[result['indices']]), 
                      color='red', s=20, label='Outliers', zorder=5)
        ax.set_title(name)
        ax.legend(fontsize=8)
        ax.set_ylabel('Density')

plt.tight_layout()
plt.show()
```

### 13. **Code**: "Effect size calculations and interpretation."

```python
import numpy as np
from scipy import stats

def cohens_d(group1, group2):
    """Cohen's d for two independent groups"""
    n1, n2 = len(group1), len(group2)
    var1, var2 = np.var(group1, ddof=1), np.var(group2, ddof=1)
    pooled_std = np.sqrt(((n1-1)*var1 + (n2-1)*var2) / (n1 + n2 - 2))
    d = (np.mean(group1) - np.mean(group2)) / pooled_std
    return d

def hedges_g(group1, group2):
    """Hedges' g (bias-corrected Cohen's d)"""
    d = cohens_d(group1, group2)
    n1, n2 = len(group1), len(group2)
    correction = 1 - 3/(4*(n1+n2) - 9)
    return d * correction

def cohens_h(p1, p2):
    """Cohen's h for two proportions"""
    return 2 * (np.arcsin(np.sqrt(p1)) - np.arcsin(np.sqrt(p2)))

def eta_squared(ss_between, ss_total):
    """Eta-squared for ANOVA"""
    return ss_between / ss_total

def omega_squared(ss_between, ss_total, df_between, ms_within):
    """Omega-squared (less biased than eta-squared)"""
    return (ss_between - df_between * ms_within) / (ss_total + ms_within)

def r_squared_from_f(f_stat, df1, df2):
    """Convert F-statistic to R²"""
    return (f_stat * df1) / (f_stat * df1 + df2)

def interpret_effect_size(d, metric='cohen'):
    """Interpret effect size magnitude"""
    thresholds = {
        'cohen': {'small': 0.2, 'medium': 0.5, 'large': 0.8},
        'pearson': {'small': 0.1, 'medium': 0.3, 'large': 0.5},
        'eta_squared': {'small': 0.01, 'medium': 0.06, 'large': 0.14},
        'cohens_h': {'small': 0.2, 'medium': 0.5, 'large': 0.8},
    }
    
    thresholds = thresholds.get(metric, thresholds['cohen'])
    abs_d = abs(d)
    
    if abs_d < thresholds['small']:
        return 'negligible'
    elif abs_d < thresholds['medium']:
        return 'small'
    elif abs_d < thresholds['large']:
        return 'medium'
    else:
        return 'large'

# Example
group_a = np.random.normal(100, 15, 100)
group_b = np.random.normal(105, 15, 100)

d = cohens_d(group_a, group_b)
g = hedges_g(group_a, group_b)
print(f"Cohen's d: {d:.3f} ({interpret_effect_size(d)})")
print(f"Hedges' g: {g:.3f} ({interpret_effect_size(g)})")

# For proportions
p1, p2 = 0.05, 0.055
h = cohens_h(p1, p2)
print(f"Cohen's h: {h:.3f} ({interpret_effect_size(h, 'cohens_h')})")

# Correlation effect size
r = 0.3
print(f"Correlation r={r}: {interpret_effect_size(r, 'pearson')}")

# ANOVA effect size
# eta_squared = SS_between / SS_total
# omega_squared (less biased)
```

### 14. **Conceptual**: "When do you use Bayesian vs Frequentist methods?"

```
FREQUENTIST (Classical):
- Parameters are FIXED, data is RANDOM
- Probability = long-run frequency
- No prior beliefs incorporated
- Inference: p-values, confidence intervals, hypothesis tests
- Properties: Unbiasedness, consistency, coverage guarantees

BAYESIAN:
- Parameters are RANDOM, data is FIXED (observed)
- Probability = degree of belief
- Prior beliefs incorporated via prior distribution
- Inference: posterior distribution, credible intervals, posterior probabilities
- Properties: Coherence, decision-theoretic optimality

WHEN FREQUENTIST:
✓ Regulatory requirements (FDA, clinical trials)
✓ Large sample sizes, simple models
✓ No reliable prior information
✓ Standard reporting requirements
✓ Need guarantees (Type I error control)

WHEN BAYESIAN:
✓ Small sample sizes
✓ Informative priors available (domain knowledge)
✓ Complex hierarchical models
✓ Sequential analysis (A/B testing)
✓ Need probability statements about parameters
✓ Decision-making under uncertainty
✓ Model averaging, model comparison

HYBRID APPROACHES:
- Empirical Bayes: Estimate prior from data
- Bayesian with flat priors → Frequentist-like results
- Frequentist properties of Bayesian procedures (calibration)
- Penalized likelihood = MAP estimation with priors

PRACTICAL ADVICE:
1. If prior is strong & reliable → Bayesian
2. If regulatory → Frequentist
3. If sequential testing → Bayesian
4. If hierarchical/multilevel → Bayesian
3. If simple hypothesis test, large n → Frequentist
4. If decision-making with costs → Bayesian
4. If regulatory submission → Frequentist (usually)
```

---

## QUICK REFERENCE: STATISTICS CHEAT SHEET

### Common Statistical Tests
| Scenario | Test | Python |
|----------|------|--------|
| Mean vs constant (σ known) | Z-test | `stats.norm` |
| Mean vs constant (σ unknown) | t-test | `stats.ttest_1samp` |
| Two independent means | t-test | `stats.ttest_ind` |
| Paired samples | Paired t-test | `stats.ttest_rel` |
| >2 group means | ANOVA | `stats.f_oneway` |
| Non-parametric (2 groups) | Mann-Whitney U | `stats.mannwhitneyu` |
| Non-parametric (>2 groups) | Kruskal-Wallis | `stats.kruskal` |
| Categorical independence | Chi-square | `stats.chi2_contingency` |
| Correlation | Pearson/Spearman | `stats.pearsonr/spearmanr` |
| Proportions (1 sample) | Binomial test | `stats.binom_test` |
| Proportions (2 samples) | Z-test | `proportions_ztest` |

### Effect Size Interpretation
| Measure | Small | Medium | Large |
|---------|-------|--------|-------|
| Cohen's d | 0.2 | 0.5 | 0.8 |
| Pearson r | 0.1 | 0.3 | 0.5 |
| Cohen's h | 0.2 | 0.5 | 0.8 |
| η² | 0.01 | 0.06 | 0.14 |
| ω² | 0.01 | 0.06 | 0.14 |
| Cohen's f | 0.1 | 0.25 | 0.4 |

### Multiple Testing Correction
| Method | Controls | When |
|---------|----------|------|
| Bonferroni | FWER | Few tests, strict |
| Holm-Bonferroni | FWER | Better power |
| Benjamini-Hochberg | FDR | Many tests, genomics |
| Benjamini-Yekutieli | FDR | Dependent tests |

### Confidence Interval Types
| Type | Assumptions | When |
|------|-------------|------|
| Wald (Normal approx) | Large n, p not near 0/1 | Quick approx |
| Wilson Score | Any n, p near 0/1 | Binomial CI |
| Clopper-Pearson (Exact) | Any n | Conservative |
| Bootstrap Percentile | Any distribution | Non-normal |
| BCa Bootstrap | Any distribution | Best for skew |

### P-Value Misconceptions
| Misconception | Reality |
|--------------|---------|
| P = P(H₀ true) | P(data | H₀) |
| P = P(results by chance) | P(extreme data | H₀) |
| 1-P = P(H₁ true) | No direct relationship |
| Significant = Important | Statistical ≠ practical |
| Non-significant = No effect | Absence of evidence ≠ evidence of absence |

### A/B Testing Checklist
- [ ] Pre-register hypothesis & analysis plan
- [ ] Calculate sample size (power analysis)
- [ ] Randomize at correct unit (user, not session)
- [ ] Check randomization balance
- [ ] Pre-specify stopping rules
- [ ] Use sequential testing if peeking
- [ ] Correct for multiple variants
- [ ] Segment analysis (new/returning, platforms)
- [ ] Check for novelty effects
- [ ] Document all decisions