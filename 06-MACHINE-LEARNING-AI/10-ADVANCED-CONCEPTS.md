# ADVANCED ML CONCEPTS
## Exam-Style Study Notes

**Roadmap Section:** Advanced Concepts in ML (NLP, Explainable AI, GANs, Autoencoders, AI Agents)
**Prerequisites:** Deep learning, Transformers, Probability, Statistics

---

## 1. NATURAL LANGUAGE PROCESSING (NLP)

### WHY? (Problem Statement)
**Human language understanding.** NLP enables machines to process, understand, and generate human language.

### HOW? (Core Concepts)

#### Tokenization
```python
# Word-level
from nltk.tokenize import word_tokenize
tokens = word_tokenize("Hello world!")

# Subword (BPE, WordPiece, SentencePiece) - Standard for Transformers
from transformers import AutoTokenizer
tokenizer = AutoTokenizer.from_pretrained('bert-base-uncased')
# BERT: WordPiece
# GPT: BPE
# T5: SentencePiece

encoding = tokenizer("Hello world!", return_tensors='pt')
# encoding['input_ids'], encoding['attention_mask']
```

#### Embeddings
```python
# Static embeddings (Word2Vec, GloVe, FastText)
# Contextual embeddings (BERT, GPT, ELMo)

# Word2Vec (skip-gram with negative sampling)
# Objective: maximize log P(context|center)
# Efficient: hierarchical softmax / negative sampling

# Contextual: each token's embedding depends on context
# BERT: bidirectional, masked language modeling
# GPT: autoregressive, next token prediction
```

#### Pre-trained Models
| Model | Architecture | Pre-training | Best For |
|-------|--------------|--------------|----------|
| **BERT** | Encoder | MLM + NSP | Classification, NER, QA |
| **RoBERTa** | Encoder | MLM (no NSP) | Better BERT |
| **DeBERTa** | Encoder | Disentangled attention | SOTA NLU |
| **GPT** | Decoder | Causal LM | Generation |
| **T5** | Enc-Dec | Span corruption | All NLP (text-to-text) |
| **BART** | Enc-Dec | Denoising | Generation, summarization |
| **LLaMA** | Decoder | Causal LM | Open LLM |
| **Mistral** | Decoder | Sliding window attention | Efficient LLM |

#### Fine-tuning Strategies
```python
# 1. Full Fine-tuning (all params)
model = AutoModelForSequenceClassification.from_pretrained('bert-base-uncased', num_labels=2)

# 2. Feature Extraction (freeze backbone)
for param in model.base_model.parameters():
    param.requires_grad = False
# Train only classifier head

# 3. LoRA (Low-Rank Adaptation) - Parameter Efficient
from peft import LoraConfig, get_peft_model
config = LoraConfig(r=16, lora_alpha=32, lora_dropout=0.1, target_modules=["query", "value"])
model = get_peft_model(model, config)

# 4. Adapter Layers
# Small bottleneck layers between transformer layers

# 5. Prompt Tuning / Prefix Tuning
# Learnable soft prompts prepended to input
```

#### Key NLP Tasks
| Task | Approach | Metrics |
|------|----------|---------|
| **Classification** | [CLS] token + head | Accuracy, F1 |
| **NER** | Token classification | Entity F1 |
| **QA** | Start/end logits | Exact Match, F1 |
| **Summarization** | Seq2seq (T5, BART) | ROUGE |
| **Translation** | Seq2seq | BLEU, COMET |
| **Language Modeling** | Causal/MLM | Perplexity |

---

## 2. EXPLAINABLE AI (XAI)

### WHY? (Problem Statement)
**Trust, debugging, compliance.** Understand why models make predictions.

### HOW? (Methods)

#### Post-hoc Explanations (Model-agnostic)
```python
import shap
import lime
from lime.lime_tabular import LimeTabularExplainer

# SHAP (SHapley Additive exPlanations)
explainer = shap.TreeExplainer(model)  # For tree models
explainer = shap.KernelExplainer(model.predict, X_train)  # Model-agnostic
shap_values = explainer.shap_values(X_test)
shap.summary_plot(shap_values, X_test)

# LIME (Local Interpretable Model-agnostic Explanations)
explainer = LimeTabularExplainer(X_train, feature_names=features, class_names=classes)
exp = explainer.explain_instance(X_test[0], model.predict_proba, num_features=10)
exp.show_in_notebook()

# Permutation Importance
from sklearn.inspection import permutation_importance
result = permutation_importance(model, X_test, y_test, n_repeats=10)
```

#### Model-Specific (Gradient-based)
```python
# Integrated Gradients (for neural nets)
def integrated_gradients(model, input_tensor, baseline=None, target_class=None, steps=50):
    if baseline is None:
        baseline = torch.zeros_like(input_tensor)
    
    # Interpolate
    alphas = torch.linspace(0, 1, steps)
    interpolated = baseline + alphas[:, None] * (input_tensor - baseline)
    interpolated.requires_grad_(True)
    
    outputs = model(interpolated)
    if target_class is not None:
        outputs = outputs[:, target_class]
    
    # Gradients
    grads = torch.autograd.grad(outputs.sum(), interpolated)[0]
    
    # Average gradients × (input - baseline)
    avg_grads = grads.mean(dim=0)
    integrated_grad = (input_tensor - baseline) * avg_grads
    return integrated_grad

# Grad-CAM (for CNNs)
def grad_cam(model, input_tensor, target_layer, target_class=None):
    # Forward hook to get activations
    # Backward hook to get gradients
    # Weight activations
    pass
```

#### Concept-Based Explanations
```python
# TCAV (Testing with Concept Activation Vectors)
# 1. Define concept (e.g., "striped" for zebra classifier)
# 2. Train linear classifier on activations to detect concept
# 3. Measure concept's influence on prediction
```

---

## 3. GENERATIVE ADVERSARIAL NETWORKS (GANs)

### WHY? (Problem Statement)
**Generate realistic data.** Learn data distribution implicitly through adversarial training.

### HOW? (Architecture)

#### Standard GAN
```math
\min_G \max_D V(D,G) = E_{x~p_data}[log D(x)] + E_{z~p_z}[log(1-D(G(z)))]
```

```python
class Generator(nn.Module):
    def __init__(self, z_dim=100, img_channels=3, features_g=64):
        super().__init__()
        self.net = nn.Sequential(
            # Input: z (B, z_dim, 1, 1)
            nn.ConvTranspose2d(z_dim, features_g*16, 4, 1, 0, bias=False),
            nn.BatchNorm2d(features_g*16), nn.ReLU(True),
            # 4x4
            nn.ConvTranspose2d(features_g*16, features_g*8, 4, 2, 1, bias=False),
            nn.BatchNorm2d(features_g*8), nn.ReLU(True),
            # 8x8
            nn.ConvTranspose2d(features_g*8, features_g*4, 4, 2, 1, bias=False),
            nn.BatchNorm2d(features_g*4), nn.ReLU(True),
            # 16x16
            nn.ConvTranspose2d(features_g*4, features_g*2, 4, 2, 1, bias=False),
            nn.BatchNorm2d(features_g*2), nn.ReLU(True),
            # 32x32
            nn.ConvTranspose2d(features_g*2, img_channels, 4, 2, 1, bias=False),
            nn.Tanh()  # [-1, 1]
        )
    
    def forward(self, z):
        return self.net(z)

class Discriminator(nn.Module):
    def __init__(self, img_channels=3, features_d=64):
        super().__init__()
        self.net = nn.Sequential(
            # 64x64
            nn.Conv2d(img_channels, features_d, 4, 2, 1, bias=False),
            nn.LeakyReLU(0.2, True),
            # 32x32
            nn.Conv2d(features_d, features_d*2, 4, 2, 1, bias=False),
            nn.BatchNorm2d(features_d*2), nn.LeakyReLU(0.2, True),
            # 16x16
            nn.Conv2d(features_d*2, features_d*4, 4, 2, 1, bias=False),
            nn.BatchNorm2d(features_d*4), nn.LeakyReLU(0.2, True),
            # 8x8
            nn.Conv2d(features_d*4, features_d*8, 4, 2, 1, bias=False),
            nn.BatchNorm2d(features_d*8), nn.LeakyReLU(0.2, True),
            # 4x4
            nn.Conv2d(features_d*8, 1, 4, 1, 0, bias=False),
            nn.Sigmoid()
        )
    
    def forward(self, x):
        return self.net(x).view(-1, 1)

# Training
def train_step(real, generator, discriminator, opt_g, opt_d, criterion, z_dim):
    batch_size = real.size(0)
    
    # Train Discriminator
    opt_d.zero_grad()
    noise = torch.randn(batch_size, z_dim, 1, 1)
    fake = generator(noise)
    
    d_real = discriminator(real)
    d_fake = discriminator(fake.detach())
    
    loss_d = criterion(d_real, torch.ones_like(d_real)) + \
             criterion(d_fake, torch.zeros_like(d_fake))
    loss_d.backward()
    opt_d.step()
    
    # Train Generator
    opt_g.zero_grad()
    d_fake = discriminator(fake)
    loss_g = criterion(d_fake, torch.ones_like(d_fake))
    loss_g.backward()
    opt_g.step()
    
    return loss_d.item(), loss_g.item()
```

#### GAN Variants
| Variant | Key Idea | Improvement |
|---------|----------|-------------|
| **DCGAN** | Conv architectures | Stable image generation |
| **WGAN** | Wasserstein distance | Better gradients, no mode collapse |
| **WGAN-GP** | Gradient penalty | Enforce Lipschitz |
| **StyleGAN** | Style mapping + AdaIN | High-res, controllable |
| **CycleGAN** | Cycle consistency | Unpaired image-to-image |
| **Pix2Pix** | Conditional + L1 | Paired image-to-image |
| **BigGAN** | Large scale + tricks | High-fidelity |
| **Diffusion** | Iterative denoising | SOTA generation (replacing GANs) |

---

## 4. AUTOENCODERS

### WHY? (Problem Statement)
**Learn compressed representations.** Dimensionality reduction, denoising, anomaly detection, generative modeling.

### HOW? (Variants)

#### Basic Autoencoder
```python
class Autoencoder(nn.Module):
    def __init__(self, input_dim, latent_dim=32):
        super().__init__()
        self.encoder = nn.Sequential(
            nn.Linear(input_dim, 256), nn.ReLU(),
            nn.Linear(256, 128), nn.ReLU(),
            nn.Linear(128, latent_dim)
        )
        self.decoder = nn.Sequential(
            nn.Linear(latent_dim, 128), nn.ReLU(),
            nn.Linear(128, 256), nn.ReLU(),
            nn.Linear(256, input_dim), nn.Sigmoid()
        )
    
    def forward(self, x):
        z = self.encoder(x)
        return self.decoder(z), z
```

#### Variational Autoencoder (VAE)
```python
class VAE(nn.Module):
    def __init__(self, input_dim, latent_dim=20):
        super().__init__()
        self.latent_dim = latent_dim
        
        # Encoder
        self.enc = nn.Sequential(
            nn.Linear(input_dim, 256), nn.ReLU(),
            nn.Linear(256, 128), nn.ReLU()
        )
        self.fc_mu = nn.Linear(128, latent_dim)
        self.fc_logvar = nn.Linear(128, latent_dim)
        
        # Decoder
        self.dec = nn.Sequential(
            nn.Linear(latent_dim, 128), nn.ReLU(),
            nn.Linear(128, 256), nn.ReLU(),
            nn.Linear(256, input_dim), nn.Sigmoid()
        )
    
    def encode(self, x):
        h = self.enc(x)
        return self.fc_mu(h), self.fc_logvar(h)
    
    def reparameterize(self, mu, logvar):
        std = torch.exp(0.5 * logvar)
        eps = torch.randn_like(std)
        return mu + eps * std
    
    def forward(self, x):
        mu, logvar = self.encode(x)
        z = self.reparameterize(mu, logvar)
        return self.dec(z), mu, logvar

# Loss: Reconstruction + KL Divergence
def vae_loss(recon, x, mu, logvar):
    recon_loss = F.binary_cross_entropy(recon, x, reduction='sum')
    kl_loss = -0.5 * torch.sum(1 + logvar - mu.pow(2) - logvar.exp())
    return recon_loss + kl_loss
```

#### Other Variants
| Type | Key Idea | Use Case |
|------|----------|----------|
| **Denoising AE** | Corrupt input, reconstruct clean | Robust features |
| **Sparse AE** | L1 on activations | Feature learning |
| **Contractive AE** | Jacobian penalty | Manifold learning |
| **VQ-VAE** | Discrete latent (codebook) | Image generation |
| **β-VAE** | Weighted KL (β > 1) | Disentangled representations |

---

## 5. AI AGENTS & LLM APPLICATIONS

### WHY? (Problem Statement)
**Autonomous task execution.** LLMs + tools + planning = agents.

### HOW? (Architecture)

#### ReAct (Reasoning + Acting)
```python
class ReActAgent:
    def __init__(self, llm, tools):
        self.llm = llm
        self.tools = {t.name: t for t in tools}
    
    def run(self, task):
        prompt = f"""Task: {task}
        
Available tools: {', '.join(self.tools.keys())}

Format:
Thought: [reasoning]
Action: [tool_name]
Action Input: [input]
Observation: [result]
... (repeat)
Final Answer: [answer]"""

        while True:
            response = self.llm(prompt)
            # Parse thought, action, action_input
            if "Final Answer:" in response:
                return response.split("Final Answer:")[1].strip()
            
            # Execute tool
            tool = self.tools[action_name]
            result = tool.run(action_input)
            prompt += f"\nObservation: {result}"
```

#### Tool Use Patterns
```python
# 1. Function Calling (OpenAI, Anthropic)
tools = [
    {"type": "function", "function": {
        "name": "search_web",
        "description": "Search the web",
        "parameters": {"type": "object", "properties": {
            "query": {"type": "string"}
        }}
    }}
]

# 2. RAG (Retrieval-Augmented Generation)
from langchain.vectorstores import FAISS
from langchain.embeddings import OpenAIEmbeddings

vectorstore = FAISS.from_documents(docs, OpenAIEmbeddings())
retriever = vectorstore.as_retriever(k=5)

# In prompt: context = retriever.get_relevant_documents(query)

# 3. Chain-of-Thought Prompting
cot_prompt = """Question: {question}
Let's think step by step:
"""

# 4. Self-Consistency
# Sample multiple CoT paths, take majority vote

# 5. Tree of Thoughts
# Explore multiple reasoning paths as tree
```

#### Multi-Agent Systems
```python
class MultiAgentSystem:
    def __init__(self):
        self.agents = {
            'planner': PlannerAgent(),
            'researcher': ResearcherAgent(),
            'coder': CoderAgent(),
            'reviewer': ReviewerAgent()
        }
    
    def solve(self, task):
        plan = self.agents['planner'].plan(task)
        for step in plan.steps:
            if step.type == 'research':
                result = self.agents['researcher'].execute(step)
            elif step.type == 'code':
                result = self.agents['coder'].execute(step)
            # Review
            review = self.agents['reviewer'].review(result)
            if not review.approved:
                # Iterate
                pass
```

---

## 6. MLOPS & PRODUCTION (Bonus)

### WHY? (Problem Statement)
**Reliable ML in production.** Deploy, monitor, maintain models.

### HOW? (Key Components)

#### Model Deployment
```python
# 1. Batch Inference
def batch_predict(model, data_path, output_path):
    df = pd.read_parquet(data_path)
    preds = model.predict(df)
    pd.DataFrame({'prediction': preds}).to_parquet(output_path)

# 2. Real-time API (FastAPI)
from fastapi import FastAPI
app = FastAPI()

@app.post("/predict")
async def predict(request: PredictRequest):
    features = np.array(request.features).reshape(1, -1)
    pred = model.predict_proba(features)[0, 1]
    return {"probability": float(pred)}

# 3. Model Server (Triton, TorchServe, BentoML)
# Triton: multi-framework, concurrent, dynamic batching
```

#### Monitoring
```python
# Data Drift
from evidently.dashboard import Dashboard
from evidently.tabs import DataDriftTab
dashboard = Dashboard(tabs=[DataDriftTab()])
dashboard.calculate(reference_data, current_data)

# Concept Drift (Performance)
# Monitor: prediction distribution, feature importance drift, 
#          target drift (when labels available)

# Model Metrics
from prometheus_client import Counter, Histogram
PREDICTIONS = Counter('predictions_total', 'Total predictions')
LATENCY = Histogram('prediction_latency_seconds', 'Prediction latency')
```

#### CI/CD for ML
```yaml
# .github/workflows/ml-pipeline.yml
jobs:
  test:
    steps:
      - pytest tests/
      - python -m pytest tests/data_validation.py
      
  train:
    needs: test
    steps:
      - python train.py --config prod
      - mlflow run .
      
  evaluate:
    needs: train
    steps:
      - python evaluate.py --model ${{ needs.train.outputs.model_uri }}
      - # Check thresholds
      
  deploy:
    needs: evaluate
    if: success()
    steps:
      - docker build -t model:v${GITHUB_SHA} .
      - kubectl set image deployment/ml-model model=model:v${GITHUB_SHA}
```

---

## PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Subword tokenization** | Word-level (OOV issues) | Handles rare/unknown words |
| **Transfer learning** | Training from scratch | Less data, better performance |
| **LoRA/Adapters** | Full fine-tuning (large models) | Few params, no catastrophic forgetting |
| **SHAP/LIME for explainability** | Raw feature importance | Model-agnostic, theoretically grounded |
| **WGAN-GP** | Standard GAN | Stable training, meaningful loss |
| **VAE for generation** | Deterministic AE | Probabilistic, interpolatable latent |
| **RAG for knowledge** | Fine-tuning on docs | Updatable, attributable, no retraining |
| **Prompt engineering** | Training for each task | Zero/few-shot, flexible |
| **Monitoring drift** | Deploy and forget | Silent degradation |
| **Canary deployment** | Direct production push | Safe rollback |

---

## INTERVIEW QUESTIONS (Top 15)

### 1. **NLP**: "How does BERT differ from GPT?"
```
BERT (Bidirectional Encoder):
- Architecture: Encoder-only (Transformer encoder)
- Pre-training: MLM (Masked LM) + NSP (Next Sentence Prediction)
- Bidirectional: Sees full context (left + right)
- Use: Understanding tasks (classification, NER, QA)

GPT (Generative Pre-trained Transformer):
- Architecture: Decoder-only (Transformer decoder)
- Pre-training: Causal LM (next token prediction)
- Unidirectional: Only sees left context
- Use: Generation tasks (text completion, chat)

KEY DIFFERENCES:
| Aspect | BERT | GPT |
|--------|------|-----|
| Direction | Bidirectional | Autoregressive |
| Pre-training | MLM + NSP | Causal LM |
| Architecture | Encoder | Decoder |
| Fine-tuning | Task-specific head | Prompting / head |
| Inference | Single forward | Sequential (token-by-token) |
```

### 2. **XAI**: "SHAP vs LIME - when to use which?"
```
SHAP (SHapley Additive exPlanations):
- Based on Shapley values from game theory
- Theoretical guarantees: local accuracy, missingness, consistency
- Global + local explanations
- TreeSHAP: fast for tree models (exact)
- KernelSHAP: model-agnostic (approximate)
- Consistent: feature with higher importance always gets higher SHAP

LIME (Local Interpretable Model-agnostic Explanations):
- Perturbs input, fits local surrogate (linear)
- No theoretical guarantees
- Faster for single predictions
- Instability: different runs → different explanations
- Good for quick debugging

WHEN:
- Need theoretical guarantees → SHAP
- Tree models → TreeSHAP (fast, exact)
- Quick sanity check → LIME
- Need global feature importance → SHAP summary plot
```

### 3. **GANs**: "Why do GANs suffer from mode collapse?"
```
MODE COLLAPSE:
Generator produces limited variety of outputs, 
ignoring modes of true data distribution.

CAUSES:
1. Generator finds single output that fools D
2. D can't distinguish → no gradient to explore
3. Min-max optimization unstable

SOLUTIONS:
1. WGAN: Wasserstein distance provides meaningful gradients everywhere
2. WGAN-GP: Gradient penalty enforces Lipschitz
3. Unrolled GAN: Unroll D steps for better G gradients
4. Minibatch Discrimination: D sees batch statistics
5. PacGAN: Pack multiple samples for D
6. Spectral Normalization: Stabilize D
7. Feature Matching: G matches feature statistics

DIFFUSION MODELS:
- Replace adversarial training with iterative denoising
- No mode collapse (explicit likelihood)
- Current SOTA for generation
```

### 4. **Autoencoders**: "VAE vs AE - what's the difference?"
```
AUTOENCODER (AE):
- Deterministic: x → z → x̂
- Loss: Reconstruction (MSE, BCE)
- Latent space: No structure, can't sample
- Use: Dimensionality reduction, denoising

VARIATIONAL AUTOENCODER (VAE):
- Probabilistic: x → q(z|x) → p(x|z)
- Latent: z ~ N(μ(x), σ²(x))
- Loss: Reconstruction + KL(q(z|x) || p(z))
- Reparameterization trick: z = μ + σ ⊙ ε
- Can sample: z ~ N(0,I) → decode → generate
- Latent space: Structured, interpolatable

TRADE-OFFS:
- VAE reconstructions blurrier (KL forces smoothness)
- AE: Sharper but no generative capability
- β-VAE: Increase KL weight → better disentanglement
- VQ-VAE: Discrete latents → better quality
```

### 5. **NLP**: "How does tokenization work in BERT vs GPT?"
```
BERT (WordPiece):
- Vocabulary: ~30K tokens
- Algorithm: Greedy longest-match from vocab
- Special tokens: [CLS], [SEP], [MASK], [PAD], [UNK]
- Example: "embedding" → ["em", "##bed", "##ding"]

GPT (BPE - Byte Pair Encoding):
- Vocabulary: ~50K tokens  
- Algorithm: Iteratively merge most frequent pairs
- No special [CLS]/[SEP] (uses position)
- Example: "embedding" → ["em", "bed", "ding"] or single token

T5 (SentencePiece / Unigram LM):
- Trained as probabilistic model
- Can handle any language without pre-tokenization
- Better for multilingual

KEY IMPLICATIONS:
- Same text → different token sequences across models
- Can't directly share embeddings
- Vocabulary size affects model capacity
- Subword handles OOV, reduces sequence length vs chars
```

### 6. **XAI**: "How would you explain a model's prediction to a non-technical stakeholder?"
```
APPROACH:
1. GLOBAL: What drives predictions overall?
   - Feature importance (SHAP summary)
   - Partial dependence plots
   - "Top 3 factors: income, credit_history, debt_ratio"

2. LOCAL: Why this specific prediction?
   - SHAP force plot / waterfall
   - Counterfactual: "If income was $5k higher, prediction would flip"
   - LIME explanation

3. BUSINESS LANGUAGE:
   - "Model predicts default risk"
   - "Primary driver: debt-to-income ratio > 0.5"
   - "Similar customers with lower DTI were approved"
   - "Confidence: 87% (based on similar cases)"

4. ACTIONABLE INSIGHTS:
   - "To improve approval: reduce DTI below 0.4"
   - "Model uncertainty high for new customers (few similar cases)"

TOOLS:
- SHAP force_plot (visual)
- Counterfactual explanations (DiCE, Alibi)
- Plain English templates
```

### 7. **LLM Agents**: "How do you prevent hallucination in RAG?"
```
HALLUCINATION SOURCES:
1. LLM generates unsupported claims
2. Retrieval brings irrelevant docs
3. Context too long → lost in middle
4. Prompt doesn't constrain generation

MITIGATION:
1. RETRIEVAL QUALITY:
   - Hybrid search (BM25 + dense)
   - Reranker (cross-encoder)
   - Query expansion / decomposition

2. PROMPT ENGINEERING:
   - "Answer ONLY using provided context"
   - "If unsure, say 'I don't know'"
   - Citation required: [doc_id]

3. GENERATION CONSTRAINTS:
   - Lower temperature (0.1-0.3)
   - Top-p sampling
   - Structured output (JSON)

4. VERIFICATION:
   - Self-consistency (multiple samples)
   - Fact-checking against retrieved docs
   - Citation validation

5. ARCHITECTURE:
   - FiD (Fusion-in-Decoder): encode docs separately
   - RETRO: Retrieve during generation
   - RAG with citations (REALM, ATLAS)
```

### 8. **Diffusion**: "How do diffusion models work?"
```
FORWARD PROCESS (noising):
x_0 → x_1 → ... → x_T ≈ N(0, I)
q(x_t|x_{t-1}) = N(x_t; √(1-β_t)x_{t-1}, β_t I)

REVERSE PROCESS (denoising):
p_θ(x_{t-1}|x_t) = N(x_{t-1}; μ_θ(x_t,t), Σ_θ(x_t,t))

TRAINING:
- Predict noise ε_θ(x_t, t) ≈ ε
- Loss: ||ε - ε_θ(x_t, t)||²
- x_t = √ᾱ_t x_0 + √(1-ᾱ_t) ε

SAMPLING:
- Start x_T ~ N(0,I)
- For t=T...1: x_{t-1} = 1/√α_t (x_t - β_t/√(1-ᾱ_t) ε_θ(x_t,t)) + σ_t z

KEY INNOVATIONS:
- U-Net architecture for ε_θ
- Classifier-free guidance: ε = ε_cond + w(ε_cond - ε_uncond)
- Latent diffusion (Stable Diffusion): Compress to latent space first
- DDIM: Deterministic sampling, faster

ADVANTAGES OVER GANs:
- Stable training (no adversarial)
- Better mode coverage
- Controllable generation
```

### 9. **Agents**: "What's the difference between ReAct and Plan-and-Execute?"
```
ReAct (Reasoning + Acting):
- Interleaved: Thought → Action → Observation → Thought...
- Single agent, reactive
- Good for: Simple tasks, unknown steps, exploration
- Limitation: Can get stuck in loops, no global plan

Plan-and-Execute:
- Phase 1: Planner creates full plan (steps)
- Phase 2: Executor follows plan step-by-step
- Can use different models for planning vs execution
- Better for: Complex multi-step, known procedures
- Limitation: Rigid, can't adapt to unexpected observations

REWOO (Reasoning without Observation):
- Separates reasoning from external calls
- Planner creates plan with variables
- Worker fills variables via tools
- More efficient (fewer LLM calls)

LATS (Language Agent Tree Search):
- MCTS over reasoning paths
- Backtracking on failure
- Best for: Hard reasoning, math, coding

CHOICE:
- Simple QA → ReAct
- Known workflow → Plan-and-Execute
- Complex reasoning → LATS/Tree of Thoughts
- Efficiency critical → REWOO
```

### 10. **Production**: "How do you detect data drift in production?"
```
DATA DRIFT TYPES:
1. Covariate Shift: P(X) changes, P(Y|X) same
   - Feature distribution shift
   - Detection: KS test, PSI, KL divergence per feature
   
2. Prior Probability Shift: P(Y) changes
   - Class balance shift
   - Detection: Monitor prediction distribution
   
3. Concept Drift: P(Y|X) changes
   - Relationship changes
   - Detection: Performance drop (when labels available)
              Proxy: prediction confidence, uncertainty

MONITORING STACK:
1. FEATURE DRIFT:
   - Evidently AI / WhyLabs / custom
   - PSI > 0.2 = significant drift
   - KS test p < 0.05 per feature
   - Alert on top drifting features

2. PREDICTION DRIFT:
   - Monitor prediction distribution (weekly/daily)
   - Compare to training distribution
   - Alert on KL divergence threshold

3. PERFORMANCE (when labels available):
   - Rolling window metrics
   - Statistical process control
   - Alert on metric degradation

4. AUTOMATED RESPONSE:
   - Retrain trigger
   - Feature store version rollback
   - Fallback to simpler model
   - Human-in-the-loop review
```

### 11. **Fine-tuning**: "When to use LoRA vs Full Fine-tuning?"
```
FULL FINE-TUNING:
- Updates all parameters
- Best performance (if enough data/compute)
- Risk: Catastrophic forgetting
- Memory: 4× params (weights + gradients + optimizer states)
- Time: Slow

LoRA (Low-Rank Adaptation):
- Freeze base model, add low-rank adapters
- W' = W + BA, where B∈ℝ^{d×r}, A∈ℝ^{r×d}, r << d
- Trainable params: ~0.1-1% of model
- Memory: Much lower (no optimizer for frozen weights)
- No inference latency (merge weights)
- Multiple adapters for different tasks

QLoRA:
- 4-bit quantization + LoRA
- Single GPU fine-tuning of 65B models

DECISION GUIDE:
| Scenario | Recommendation |
|----------|----------------|
| Small model (<1B), lots of data | Full FT |
| Large model (>7B), limited data | LoRA/QLoRA |
| Multiple tasks | LoRA (separate adapters) |
| Production, need speed | LoRA (merge) |
| Maximum quality critical | Full FT |
| Limited GPU memory | QLoRA |
| Continual learning | LoRA (no forgetting) |
```

### 12. **RAG**: "How do you evaluate a RAG system?"
```
EVALUATION DIMENSIONS:

1. RETRIEVAL QUALITY:
   - Precision@k / Recall@k (ground truth docs)
   - MRR / NDCG (ranking)
   - Hit rate (correct doc in top-k)

2. GENERATION QUALITY:
   - Faithfulness: Answer supported by context?
   - Answer relevance: Addresses question?
   - Hallucination rate: Unsupported claims

3. END-TO-END:
   - Exact match / F1 (QA)
   - BLEU/ROUGE (generation)
   - Human evaluation (gold standard)

METRICS & TOOLS:
- RAGAS: Faithfulness, Answer Relevancy, Context Precision, Context Recall
- TruLens: Groundedness, Context Relevance
- LangSmith: Custom evaluators
- Custom: LLM-as-judge (GPT-4 eval)

BENCHMARKS:
- HotpotQA (multi-hop)
- Natural Questions
- MS MARCO
- FEVER (fact verification)
- Custom domain benchmarks

ABLATION STUDIES:
- Chunk size / overlap
- Embedding model
- Retriever (dense/sparse/hybrid)
- Reranker
- Top-k
- Prompt template
```

### 13. **MLOps**: "How do you version ML models and data?"
```
MODEL VERSIONING:
1. MLflow / Weights & Biases / Neptune
   - Log params, metrics, artifacts
   - Model registry (staging/production)
   - Lineage: code version → data version → model

2. Model Artifacts:
   - Model weights + config + tokenizer
   - Docker image (code + deps + model)
   - ONNX / TorchScript for portability

DATA VERSIONING:
1. DVC (Data Version Control)
   - .dvc files track data files
   - Remote storage (S3, GCS, Azure)
   - Git for code, DVC for data

2. LakeFS / Delta Lake / Iceberg
   - Git-like semantics for data lakes
   - Time travel, branching, merging
   - ACID transactions

FEATURE STORE:
- Feast / Tecton / Hopsworks
- Feature definitions versioned
- Offline (training) + Online (serving) sync

BEST PRACTICES:
- Immutable artifacts
- Semantic versioning (v1.2.3)
- Automated promotion (staging → prod)
- Rollback capability
- Audit trail
```

### 14. **Monitoring**: "What metrics do you monitor for a classification model in production?"
```
TIER 1 - IMMEDIATE (Alert on breach):
1. Latency: p50, p95, p99 < SLA
2. Error rate: 5xx < 0.1%, 4xx < 1%
3. Throughput: Requests/sec
4. Availability: Uptime > 99.9%

TIER 2 - DAILY (Data Science):
1. Prediction Distribution:
   - Mean, std, quantiles of predicted probabilities
   - Shift from training distribution (KL, PSI)
   
2. Feature Drift:
   - Per-feature PSI / KS test
   - Top 10 drifting features
   - Missing value rates

3. Performance (when labels available):
   - Accuracy, AUC, F1, Precision, Recall
   - Per-segment (user tier, geography, device)
   - Confusion matrix drift

TIER 3 - WEEKLY (Business):
1. Business Metrics:
   - Conversion rate, revenue per user
   - False positive cost, false negative cost
   - Model ROI

2. Data Quality:
   - Schema violations
   - Completeness, freshness
   - Upstream system health

TOOLS:
- Prometheus + Grafana (infrastructure)
- Evidently / WhyLabs / Arize (ML monitoring)
- Custom dashboards (business metrics)
```

### 15. **Ethics**: "How do you ensure fairness in ML models?"
```
FAIRNESS DEFINITIONS:
1. Demographic Parity: P(Ŷ=1|A=a) = P(Ŷ=1|A=b)
   - Equal positive rates across groups
   
2. Equal Opportunity: P(Ŷ=1|Y=1,A=a) = P(Ŷ=1|Y=1,A=b)
   - Equal TPR across groups
   
3. Equalized Odds: TPR + FPR equal across groups
4. Calibration: P(Y=1|Ŷ=p,A=a) = p for all groups
5. Individual Fairness: Similar individuals → similar predictions

MITIGATION STRATEGIES:

PRE-PROCESSING:
- Reweighting / resampling
- Disparate impact remover
- Fair representation learning

IN-PROCESSING:
- Constrained optimization (fairness as constraint)
- Adversarial debiasing (remove protected info from features)
- Fairness-aware loss (add fairness penalty)

POST-PROCESSING:
- Threshold optimization per group
- Equalized odds post-processing
- Calibrated equalized odds

MEASUREMENT:
- Aequitas, Fairlearn, AI Fairness 360
- Disaggregated metrics by protected attributes
- Intersectional analysis (race × gender)

GOVERNANCE:
- Fairness dashboard in monitoring
- Bias audit before deployment
- Diverse training data
- Diverse development team
- Regular fairness reviews
```

---

## QUICK REFERENCE: ADVANCED ML CHEAT SHEET

### NLP Model Selection
| Task | Model | Fine-tuning |
|------|-------|-------------|
| Classification | BERT/DeBERTa | Full / LoRA |
| NER | BERT + CRF | Full / LoRA |
| QA | BERT / T5 | Full / LoRA |
| Summarization | BART / T5 / PEGASUS | LoRA |
| Translation | NLLB / M2M-100 | LoRA |
| Generation | GPT / LLaMA / Mistral | LoRA / QLoRA |
| Code | CodeLlama / StarCoder | LoRA |

### XAI Tool Selection
| Need | Tool |
|------|------|
| Tree models (global+local) | TreeSHAP |
| Any model (local) | KernelSHAP / LIME |
| Neural nets (gradients) | Integrated Gradients / DeepSHAP |
| CNNs | Grad-CAM / Grad-CAM++ |
| Text | Attention visualization / LIME |
| Tabular global | Permutation Importance / SHAP |

### Generative Model Selection
| Requirement | Model |
|-------------|-------|
| Images, highest quality | Diffusion (SDXL, Midjourney, DALL-E) |
| Images, controllable | StyleGAN / ControlNet |
| Images, fast | GAN (StyleGAN, GigaGAN) |
| Text → Image | Diffusion |
| Image → Image | CycleGAN / Pix2Pix / ControlNet |
| Tabular synthesis | TVAE / CTGAN / Diffusion |
| Anomaly detection | VAE / GAN / Isolation Forest |

### Agent Pattern Selection
| Complexity | Pattern |
|------------|---------|
| Simple QA, tools | ReAct |
| Known multi-step | Plan-and-Execute |
| Complex reasoning | Tree of Thoughts / LATS |
| Efficiency critical | REWOO |
| Multi-domain | Multi-Agent (specialized) |
| Code generation | CodeAct / Interpreter |

### MLOps Maturity Levels
| Level | Characteristics |
|-------|-----------------|
| 0 | Manual, notebooks, no versioning |
| 1 | Automated training, manual deploy |
| 2 | CI/CD for model, basic monitoring |
| 3 | Automated retraining, feature store, A/B |
| 4 | Full automation, self-healing, governance |