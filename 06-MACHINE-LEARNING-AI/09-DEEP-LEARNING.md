# DEEP LEARNING
## Exam-Style Study Notes

**Roadmap Section:** Deep Learning (NN Basics, CNNs, RNNs, Transformers, Attention)
**Prerequisites:** Linear algebra, calculus, probability, Python, PyTorch/TensorFlow

---

## 1. NEURAL NETWORK BASICS

### WHY? (Problem Statement)
**Universal function approximators.** Neural networks learn hierarchical representations from raw data, enabling breakthroughs in vision, NLP, and beyond.

### HOW? (Core Mechanics)

#### Perceptron & MLP
```math
\text{Perceptron: } y = \sigma(w^T x + b)
\text{MLP: } y = \sigma(W_L \sigma(... \sigma(W_1 x + b_1) ...) + b_L)
```

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

# Basic MLP
class MLP(nn.Module):
    def __init__(self, input_dim, hidden_dims, output_dim, dropout=0.1):
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
        
        layers.append(nn.Linear(prev_dim, output_dim))
        self.net = nn.Sequential(*layers)
    
    def forward(self, x):
        return self.net(x)
```

#### Activation Functions
| Function | Formula | Range | Use Case |
|----------|---------|-------|----------|
| **ReLU** | max(0, x) | [0, ∞) | Default hidden layers |
| **Leaky ReLU** | max(αx, x) | (-∞, ∞) | Avoid dead neurons |
| **GELU** | xΦ(x) | (-∞, ∞) | Transformers, BERT |
| **Swish** | x·σ(x) | (-∞, ∞) | Deep networks |
| **Sigmoid** | 1/(1+e⁻ˣ) | (0, 1) | Binary output |
| **Tanh** | (eˣ-e⁻ˣ)/(eˣ+e⁻ˣ) | (-1, 1) | RNN gates |
| **Softmax** | eˣᵢ/Σeˣⱼ | (0, 1) | Multi-class output |

#### Loss Functions
```python
# Classification
nn.CrossEntropyLoss()           # Softmax + NLL (multi-class)
nn.BCEWithLogitsLoss()          # Sigmoid + BCE (multi-label)
nn.NLLLoss()                    # Log softmax input

# Regression
nn.MSELoss()                    # L2
nn.L1Loss()                     # L1 (MAE)
nn.HuberLoss(delta=1.0)         # Robust
nn.SmoothL1Loss()               # Huber with delta=1

# Custom
class FocalLoss(nn.Module):
    def __init__(self, alpha=0.25, gamma=2.0):
        super().__init__()
        self.alpha = alpha
        self.gamma = gamma
    
    def forward(self, inputs, targets):
        ce_loss = F.cross_entropy(inputs, targets, reduction='none')
        pt = torch.exp(-ce_loss)
        focal_loss = self.alpha * (1 - pt) ** self.gamma * ce_loss
        return focal_loss.mean()
```

#### Backpropagation & Gradient Flow
```math
\frac{\partial L}{\partial W} = \frac{\partial L}{\partial y} \cdot \frac{\partial y}{\partial W}
```

```python
# Gradient clipping (prevent explosion)
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)

# Gradient accumulation (large effective batch)
for i, (x, y) in enumerate(dataloader):
    loss = model(x, y) / accumulation_steps
    loss.backward()
    
    if (i + 1) % accumulation_steps == 0:
        optimizer.step()
        optimizer.zero_grad()
```

---

## 2. CONVOLUTIONAL NEURAL NETWORKS (CNNs)

### WHY? (Problem Statement)
**Translation equivariance + parameter sharing.** CNNs exploit spatial locality for images.

### HOW? (Architecture)

#### Convolution Operations
```python
# Conv2d parameters
nn.Conv2d(
    in_channels, out_channels, 
    kernel_size=3, stride=1, padding=1,  # padding='same' for same spatial dims
    dilation=1, groups=1, bias=True
)

# Output size: H_out = floor((H_in + 2*padding - dilation*(kernel-1) - 1)/stride + 1)
```

#### Key Concepts
| Concept | Purpose |
|---------|---------|
| **Padding** | Maintain spatial dimensions, border handling |
| **Stride** | Downsampling, receptive field |
| **Dilation** | Expand receptive field without params |
| **Groups** | Depthwise separable (groups=in_channels) |
| **1x1 Conv** | Channel mixing, dimensionality reduction |

#### Modern CNN Architectures
```python
# Residual Block (ResNet)
class ResidualBlock(nn.Module):
    def __init__(self, in_channels, out_channels, stride=1):
        super().__init__()
        self.conv1 = nn.Conv2d(in_channels, out_channels, 3, stride, 1, bias=False)
        self.bn1 = nn.BatchNorm2d(out_channels)
        self.conv2 = nn.Conv2d(out_channels, out_channels, 3, 1, 1, bias=False)
        self.bn2 = nn.BatchNorm2d(out_channels)
        
        self.shortcut = nn.Sequential()
        if stride != 1 or in_channels != out_channels:
            self.shortcut = nn.Sequential(
                nn.Conv2d(in_channels, out_channels, 1, stride, bias=False),
                nn.BatchNorm2d(out_channels)
            )
    
    def forward(self, x):
        out = F.relu(self.bn1(self.conv1(x)))
        out = self.bn2(self.conv2(out))
        out += self.shortcut(x)
        return F.relu(out)

# Depthwise Separable Conv (MobileNet)
class DepthwiseSeparableConv(nn.Module):
    def __init__(self, in_channels, out_channels, stride=1):
        super().__init__()
        self.depthwise = nn.Conv2d(in_channels, in_channels, 3, stride, 1, 
                                    groups=in_channels, bias=False)
        self.pointwise = nn.Conv2d(in_channels, out_channels, 1, bias=False)
    
    def forward(self, x):
        return self.pointwise(self.depthwise(x))

# Squeeze-and-Excitation (SE Block)
class SEBlock(nn.Module):
    def __init__(self, channels, reduction=16):
        super().__init__()
        self.avg_pool = nn.AdaptiveAvgPool2d(1)
        self.fc = nn.Sequential(
            nn.Linear(channels, channels // reduction),
            nn.ReLU(),
            nn.Linear(channels // reduction, channels),
            nn.Sigmoid()
        )
    
    def forward(self, x):
        b, c, _, _ = x.size()
        y = self.avg_pool(x).view(b, c)
        y = self.fc(y).view(b, c, 1, 1)
        return x * y.expand_as(x)
```

#### Popular Architectures Summary
| Model | Year | Key Innovation | Params |
|-------|------|----------------|--------|
| **LeNet-5** | 1998 | First CNN | 60K |
| **AlexNet** | 2012 | ReLU, Dropout, GPU | 60M |
| **VGG** | 2014 | 3x3 conv stacks, depth | 138M |
| **GoogLeNet** | 2014 | Inception modules | 6.8M |
| **ResNet** | 2015 | Residual connections | 25M |
| **DenseNet** | 2017 | Dense connections | 20M |
| **EfficientNet** | 2019 | Compound scaling | 5.3M |
| **ConvNeXt** | 2022 | Modernized ResNet | 28M |
| **ViT** | 2020 | Transformer for images | 86M |

---

## 3. RECURRENT NEURAL NETWORKS (RNNs)

### WHY? (Problem Statement)
**Sequential data processing.** Handle variable-length sequences with temporal dependencies.

### HOW? (Architectures)

#### Vanilla RNN Problems
```math
h_t = \tanh(W_h h_{t-1} + W_x x_t + b)
```
**Vanishing/Exploding Gradients:** ∏(∂h/∂h) → 0 or ∞ over long sequences

#### LSTM (Long Short-Term Memory)
```python
class LSTMCell(nn.Module):
    def __init__(self, input_size, hidden_size):
        super().__init__()
        self.hidden_size = hidden_size
        # 4 gates: input, forget, cell, output
        self.gates = nn.Linear(input_size + hidden_size, 4 * hidden_size)
    
    def forward(self, x, hidden):
        h, c = hidden
        combined = torch.cat([x, h], dim=1)
        gates = self.gates(combined).chunk(4, dim=1)
        
        i, f, g, o = gates
        i = torch.sigmoid(i)
        f = torch.sigmoid(f)
        g = torch.tanh(g)
        o = torch.sigmoid(o)
        
        c_new = f * c + i * g
        h_new = o * torch.tanh(c_new)
        return h_new, c_new

# PyTorch built-in (optimized)
lstm = nn.LSTM(input_size, hidden_size, num_layers=2, 
               batch_first=True, dropout=0.1, bidirectional=True)
output, (h_n, c_n) = lstm(x)  # x: (batch, seq, features)
```

#### GRU (Gated Recurrent Unit) - Simpler LSTM
```python
gru = nn.GRU(input_size, hidden_size, num_layers=2, 
             batch_first=True, dropout=0.1, bidirectional=True)
```

#### RNN Best Practices
```python
# Pack sequences for variable lengths (efficiency)
from torch.nn.utils.rnn import pack_padded_sequence, pad_packed_sequence

lengths = [len(seq) for seq in sequences]
packed = pack_padded_sequence(padded_seqs, lengths, batch_first=True, enforce_sorted=False)
output, hidden = lstm(packed)
output, _ = pad_packed_sequence(output, batch_first=True)

# Gradient clipping (critical for RNNs)
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)

# Teacher forcing (training)
# At step t: use ground truth with prob p, else model prediction
```

---

## 4. TRANSFORMERS & ATTENTION

### WHY? (Problem Statement)
**Parallelization + long-range dependencies.** Transformers replaced RNNs for sequence modeling.

### HOW? (Architecture)

#### Self-Attention
```math
\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V
```

```python
class MultiHeadAttention(nn.Module):
    def __init__(self, d_model, n_heads, dropout=0.1):
        super().__init__()
        assert d_model % n_heads == 0
        self.d_model = d_model
        self.n_heads = n_heads
        self.d_k = d_model // n_heads
        
        self.w_q = nn.Linear(d_model, d_model)
        self.w_k = nn.Linear(d_model, d_model)
        self.w_v = nn.Linear(d_model, d_model)
        self.w_o = nn.Linear(d_model, d_model)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, q, k, v, mask=None):
        batch_size = q.size(0)
        
        # Linear projections + split heads
        q = self.w_q(q).view(batch_size, -1, self.n_heads, self.d_k).transpose(1, 2)
        k = self.w_k(k).view(batch_size, -1, self.n_heads, self.d_k).transpose(1, 2)
        v = self.w_v(v).view(batch_size, -1, self.n_heads, self.d_k).transpose(1, 2)
        
        # Scaled dot-product attention
        scores = torch.matmul(q, k.transpose(-2, -1)) / math.sqrt(self.d_k)
        
        if mask is not None:
            scores = scores.masked_fill(mask == 0, -1e9)
        
        attn = F.softmax(scores, dim=-1)
        attn = self.dropout(attn)
        
        out = torch.matmul(attn, v)
        out = out.transpose(1, 2).contiguous().view(batch_size, -1, self.d_model)
        return self.w_o(out)
```

#### Transformer Encoder/Decoder
```python
class TransformerEncoderLayer(nn.Module):
    def __init__(self, d_model, n_heads, d_ff, dropout=0.1):
        super().__init__()
        self.self_attn = MultiHeadAttention(d_model, n_heads, dropout)
        self.ffn = nn.Sequential(
            nn.Linear(d_model, d_ff),
            nn.ReLU(),
            nn.Dropout(dropout),
            nn.Linear(d_ff, d_model)
        )
        self.norm1 = nn.LayerNorm(d_model)
        self.norm2 = nn.LayerNorm(d_model)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, x, mask=None):
        # Self-attention + residual + norm
        attn_out = self.self_attn(x, x, x, mask)
        x = self.norm1(x + self.dropout(attn_out))
        
        # FFN + residual + norm
        ffn_out = self.ffn(x)
        x = self.norm2(x + self.dropout(ffn_out))
        return x

# BERT-style: Encoder only
# GPT-style: Decoder only (causal mask)
# T5/BART: Encoder-Decoder
```

#### Positional Encoding
```python
class PositionalEncoding(nn.Module):
    def __init__(self, d_model, max_len=5000):
        super().__init__()
        pe = torch.zeros(max_len, d_model)
        position = torch.arange(0, max_len).unsqueeze(1).float()
        div_term = torch.exp(torch.arange(0, d_model, 2).float() * 
                             -(math.log(10000.0) / d_model))
        pe[:, 0::2] = torch.sin(position * div_term)
        pe[:, 1::2] = torch.cos(position * div_term)
        self.register_buffer('pe', pe.unsqueeze(0))
    
    def forward(self, x):
        return x + self.pe[:, :x.size(1)]
```

#### Modern Transformer Variants
| Variant | Key Idea | Use Case |
|---------|----------|----------|
| **BERT** | Bidirectional encoder, MLM + NSP | NLU, classification |
| **GPT** | Causal decoder, autoregressive | Generation |
| **T5** | Encoder-decoder, text-to-text | All NLP tasks |
| **ViT** | Image patches as tokens | Vision |
| **Swin** | Hierarchical + shifted windows | Vision |
| **Performer** | Linear attention (kernel approx) | Long sequences |
| **Longformer** | Local + global attention | Long documents |
| **FlashAttention** | IO-aware exact attention | Training speed |

---

## 5. ADVANCED TECHNIQUES

### Normalization
```python
nn.BatchNorm1d(num_features)      # 1D (N, C)
nn.BatchNorm2d(num_features)      # 2D (N, C, H, W)
nn.LayerNorm(normalized_shape)    # Per-sample (transformer)
nn.GroupNorm(num_groups, channels) # Between BN & LN
nn.InstanceNorm2d(num_features)   # Style transfer
```

### Regularization
```python
# Dropout
nn.Dropout(p=0.1)              # Standard
nn.Dropout2d(p=0.1)            # Spatial (channels)
nn.AlphaDropout(p=0.1)         # For SELU

# Weight Decay (L2) - in optimizer
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-3, weight_decay=0.01)

# Label Smoothing
nn.CrossEntropyLoss(label_smoothing=0.1)

# Mixup / CutMix (data augmentation)
def mixup_data(x, y, alpha=1.0):
    lam = np.random.beta(alpha, alpha)
    idx = torch.randperm(x.size(0))
    mixed_x = lam * x + (1 - lam) * x[idx]
    y_a, y_b = y, y[idx]
    return mixed_x, y_a, y_b, lam
```

### Optimizers & Schedulers
```python
# AdamW (decoupled weight decay)
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-3, weight_decay=0.01)

# Learning rate schedules
scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=epochs)
scheduler = torch.optim.lr_scheduler.OneCycleLR(optimizer, max_lr=1e-3, 
                                                steps_per_epoch=len(loader), epochs=epochs)
scheduler = torch.optim.lr_scheduler.ReduceLROnPlateau(optimizer, patience=5, factor=0.5)
scheduler = torch.optim.lr_scheduler.CosineAnnealingWarmRestarts(optimizer, T_0=10, T_mult=2)

# Gradient accumulation
accumulation_steps = 4
for i, (x, y) in enumerate(loader):
    loss = model(x, y) / accumulation_steps
    loss.backward()
    if (i + 1) % accumulation_steps == 0:
        optimizer.step()
        optimizer.zero_grad()
```

---

## 6. TRAINING BEST PRACTICES

### Initialization
```python
def init_weights(m):
    if isinstance(m, nn.Linear):
        nn.init.xavier_uniform_(m.weight)
        if m.bias is not None:
            nn.init.zeros_(m.bias)
    elif isinstance(m, nn.Conv2d):
        nn.init.kaiming_normal_(m.weight, mode='fan_out', nonlinearity='relu')
    elif isinstance(m, (nn.BatchNorm1d, nn.BatchNorm2d, nn.LayerNorm)):
        nn.init.ones_(m.weight)
        nn.init.zeros_(m.bias)

model.apply(init_weights)
```

### Mixed Precision Training
```python
from torch.cuda.amp import autocast, GradScaler

scaler = GradScaler()
for x, y in loader:
    optimizer.zero_grad()
    with autocast():
        output = model(x)
        loss = criterion(output, y)
    scaler.scale(loss).backward()
    scaler.step(optimizer)
    scaler.update()
```

### Distributed Training
```python
# DDP (DistributedDataParallel) - preferred
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP

dist.init_process_group(backend='nccl')
model = DDP(model.to(device), device_ids=[local_rank])

# FSDP (Fully Sharded Data Parallel) - for huge models
from torch.distributed.fsdp import FullyShardedDataParallel as FSDP
model = FSDP(model)
```

---

## PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **BatchNorm after Conv, before ReLU** | BN after ReLU | Better gradient flow |
| **Residual connections** | Plain deep nets | Vanishing gradients |
| **Gradient clipping (1.0)** | No clipping | Exploding gradients (RNN) |
| **Learning rate warmup** | Constant LR from start | Transformer instability |
| **Weight decay in AdamW** | L2 in Adam | Decoupled better |
| **Mixed precision (AMP)** | FP32 only | 2x speed, half memory |
| **Proper initialization** | Default init | Slow convergence |
| **Validation during training** | Only final eval | Miss overfitting |

---

## INTERVIEW QUESTIONS (Top 15)

### 1. **Conceptual**: "Why do we need residual connections?"
```
PROBLEM: Deep networks suffer from vanishing gradients.
∂L/∂W₁ = ∂L/∂y · ∂y/∂h_L · ∂h_L/∂h_{L-1} · ... · ∂h_2/∂h_1 · ∂h_1/∂W₁

Each ∂h/∂h ≈ W (if linear) or σ'(W) (if nonlinear)
Product of many small numbers → 0 (vanishing)
Product of many large numbers → ∞ (exploding)

RESIDUAL SOLUTION:
h_{l+1} = h_l + F(h_l)
∂h_{l+1}/∂h_l = I + ∂F/∂h_l

Identity path preserves gradient flow!
Even if F → 0, gradient flows through identity.
Enables 1000+ layer networks (ResNet).
```

### 2. **Code**: "Implement Multi-Head Attention from scratch."
```python
# See complete implementation above in Section 4
# Key points:
# 1. Split Q,K,V into heads: (B, T, D) → (B, H, T, D/H)
# 2. Scaled dot-product: QK^T / √d_k
# 3. Mask (causal/padding) before softmax
# 4. Concat heads: (B, H, T, D/H) → (B, T, D)
# 5. Output projection
```

### 3. **Design**: "How would you build a CNN for medical image classification (small dataset, high-res)?"
```
CHALLENGES:
- Small dataset (1000 images)
- High resolution (2048x2048)
- Class imbalance
- Need interpretability

SOLUTION:
1. TRANSFER LEARNING:
   - Pretrained EfficientNet-B0/B1 on ImageNet
   - Freeze early layers, fine-tune last blocks
   - Replace classifier head

2. PATCH-BASED APPROACH:
   - Extract 256x256 patches with overlap
   - Train on patches, aggregate predictions
   - Handles high-res without downsampling

3. DATA AUGMENTATION (critical):
   - Albumentations: rotate, flip, elastic, brightness
   - Mixup, CutMix
   - Test-time augmentation (TTA)

4. CLASS IMBALANCE:
   - Weighted loss (class_weight)
   - Focal Loss
   - Oversampling minority

5. INTERPRETABILITY:
   - Grad-CAM heatmaps
   - Attention maps
   - Uncertainty estimation (MC Dropout)

CODE:
model = timm.create_model('efficientnet_b0', pretrained=True, num_classes=num_classes)
# Freeze backbone
for param in model.parameters():
    param.requires_grad = False
# Unfreeze last blocks
for param in model.blocks[-2:].parameters():
    param.requires_grad = True
# New classifier
model.classifier = nn.Linear(model.classifier.in_features, num_classes)
```

### 4. **Code**: "Implement LSTM with packed sequences for variable lengths."
```python
class LSTMClassifier(nn.Module):
    def __init__(self, vocab_size, embed_dim, hidden_dim, num_layers, num_classes, dropout=0.1):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embed_dim, padding_idx=0)
        self.lstm = nn.LSTM(embed_dim, hidden_dim, num_layers, 
                            batch_first=True, bidirectional=True, dropout=dropout)
        self.dropout = nn.Dropout(dropout)
        self.fc = nn.Linear(hidden_dim * 2, num_classes)
    
    def forward(self, x, lengths):
        # x: (batch, seq_len)
        embedded = self.embedding(x)  # (batch, seq, embed)
        
        # Pack
        packed = pack_padded_sequence(embedded, lengths.cpu(), 
                                       batch_first=True, enforce_sorted=False)
        
        # LSTM
        packed_out, (h_n, c_n) = self.lstm(packed)
        
        # Unpack (if needed) or use final hidden
        # For bidirectional: concat last forward + first backward
        h_n = h_n.view(self.lstm.num_layers, 2, -1, self.lstm.hidden_size)
        h_last = torch.cat([h_n[-1, 0], h_n[-1, 1]], dim=1)  # (batch, 2*hidden)
        
        out = self.dropout(h_last)
        return self.fc(out)
```

### 5. **Conceptual**: "Transformer vs RNN - when to use which?"

```
RNN (LSTM/GRU):
✓ Sequential processing (online/streaming)
✓ Low latency per token (no full sequence needed)
✓ Smaller memory for long sequences (O(1) state)
✗ No parallelization (sequential dependency)
✗ Vanishing gradients over long range
✗ Hard to capture very long dependencies

TRANSFORMER:
✓ Full parallelization (all tokens simultaneously)
✓ Global receptive field (every token sees every token)
✓ Better long-range dependencies
✗ O(n²) memory/compute for attention
✗ Needs full sequence (batch inference)
✗ Positional encoding needed (no inherent order)

USE RNN WHEN:
- Streaming/real-time (speech recognition, online)
- Very long sequences (memory constrained)
- Simple sequence tasks

USE TRANSFORMER WHEN:
- Batch processing (NLP, offline)
- Long-range dependencies critical
- Parallel training needed
- State-of-the-art performance needed

HYBRID:
- Transformer encoder + RNN decoder
- RNN for encoding, attention for decoding
- Transformer with recurrent memory (Transformer-XL)
```

### 6. **Code**: "Implement a Vision Transformer (ViT) from scratch."
```python
class ViT(nn.Module):
    def __init__(self, img_size=224, patch_size=16, in_channels=3, 
                 num_classes=1000, d_model=768, depth=12, n_heads=12, mlp_ratio=4):
        super().__init__()
        self.patch_size = patch_size
        self.num_patches = (img_size // patch_size) ** 2
        
        # Patch embedding
        self.patch_embed = nn.Conv2d(in_channels, d_model, 
                                      kernel_size=patch_size, stride=patch_size)
        
        # Class token + position embedding
        self.cls_token = nn.Parameter(torch.zeros(1, 1, d_model))
        self.pos_embed = nn.Parameter(torch.zeros(1, self.num_patches + 1, d_model))
        self.pos_drop = nn.Dropout(0.1)
        
        # Transformer encoder
        encoder_layer = nn.TransformerEncoderLayer(
            d_model, n_heads, d_model * mlp_ratio, dropout=0.1, 
            activation='gelu', batch_first=True, norm_first=True
        )
        self.encoder = nn.TransformerEncoder(encoder_layer, depth)
        
        # Classifier head
        self.norm = nn.LayerNorm(d_model)
        self.head = nn.Linear(d_model, num_classes)
        
        nn.init.trunc_normal_(self.pos_embed, std=0.02)
        nn.init.trunc_normal_(self.cls_token, std=0.02)
    
    def forward(self, x):
        # x: (B, C, H, W)
        B = x.shape[0]
        
        # Patch embedding
        x = self.patch_embed(x)  # (B, D, H/P, W/P)
        x = x.flatten(2).transpose(1, 2)  # (B, N, D)
        
        # Add cls token
        cls_tokens = self.cls_token.expand(B, -1, -1)
        x = torch.cat([cls_tokens, x], dim=1)
        
        # Position embedding
        x = x + self.pos_embed
        x = self.pos_drop(x)
        
        # Transformer
        x = self.encoder(x)
        
        # Classify using cls token
        x = self.norm(x[:, 0])
        return self.head(x)
```

### 7. **Conceptual**: "Explain attention mechanism and why it works."
```
ATTENTION = Weighted sum of values based on query-key compatibility.

Q (Query): What am I looking for?
K (Key): What do I contain?
V (Value): What do I contribute?

SCORES = QK^T / √d_k
WEIGHTS = softmax(SCORES)
OUTPUT = WEIGHTS × V

WHY SCALED DOT PRODUCT?
- Dot product grows with dimension d_k
- Large values push softmax to saturation (small gradients)
- 1/√d_k normalizes variance

WHY MULTI-HEAD?
- Different heads learn different relationships
- Head 1: syntactic, Head 2: semantic, Head 3: positional
- Concatenated → output projection mixes information

SELF-ATTENTION:
Q=K=V=X (same input)
Each position attends to all positions
Captures global context in 1 layer

CAUSAL ATTENTION (GPT):
Mask future positions (upper triangular)
Ensures autoregressive property

CROSS-ATTENTION (Encoder-Decoder):
Q from decoder, K,V from encoder
Decoder attends to encoder representations
```

### 8. **Design**: "How to train a model on multiple GPUs?"
```
SINGLE NODE, MULTI-GPU:

1. DATA PARALLEL (DDP - Recommended):
   - Replicate model on each GPU
   - Split batch across GPUs
   - All-reduce gradients
   
   torchrun --nproc_per_node=4 train.py
   
   model = DDP(model.to(device), device_ids=[local_rank])

2. MODEL PARALLEL:
   - Split model across GPUs (layer-wise)
   - Pipeline parallelism (GPipe)
   - For models too large for single GPU

3. ZERO REDUNDANCY (ZeRO/DeepSpeed):
   - Partition optimizer states, gradients, params
   - ZeRO-1: optimizer states
   - ZeRO-2: + gradients
   - ZeRO-3: + parameters
   
   from deepspeed import initialize
   model, optimizer, _, _ = initialize(model, optimizer, config=ds_config)

MULTI-NODE:
- NCCL backend for GPU communication
- DistributedSampler for data
- SyncBatchNorm for BN

BATCH SIZE SCALING:
- Linear scaling rule: lr × √(batch_size)
- Warmup for large batches
- LARS/LAMB optimizers for very large batches
```

### 9. **Code**: "Implement gradient accumulation + mixed precision training loop."
```python
def train_epoch(model, loader, optimizer, criterion, scaler, 
                accumulation_steps=4, device='cuda', clip_grad=1.0):
    model.train()
    total_loss = 0
    
    for i, (x, y) in enumerate(loader):
        x, y = x.to(device), y.to(device)
        
        with torch.cuda.amp.autocast():
            output = model(x)
            loss = criterion(output, y) / accumulation_steps
        
        scaler.scale(loss).backward()
        
        if (i + 1) % accumulation_steps == 0:
            # Unscale for clipping
            scaler.unscale_(optimizer)
            torch.nn.utils.clip_grad_norm_(model.parameters(), clip_grad)
            
            scaler.step(optimizer)
            scaler.update()
            optimizer.zero_grad()
        
        total_loss += loss.item() * accumulation_steps
    
    return total_loss / len(loader)
```

### 10. **Conceptual**: "What is the difference between BatchNorm and LayerNorm?"

```
BATCH NORM (BN):
- Normalizes over batch dimension: (N, C, H, W) → mean/var per channel
- Requires batch statistics (running mean/var during eval)
- Batch size dependent (small batch → poor estimates)
- Works well for CNNs, fixed batch sizes
- Problem: RNNs, transformers, small batches, variable lengths

LAYER NORM (LN):
- Normalizes over feature dimension: (N, *, D) → mean/var per sample
- No batch statistics needed (same train/eval)
- Works for any batch size, sequence length
- Standard for Transformers, RNNs
- Independent samples

INSTANCE NORM (IN):
- Normalizes per sample per channel: (N, C, H, W) → per channel per sample
- Style transfer, GANs

GROUP NORM (GN):
- Divides channels into groups, normalizes within group
- Between BN and LN
- Good for small batches, detection/segmentation

WHEN TO USE:
- CNN, large batch → BN
- Transformer, RNN, small batch → LN
- Detection/Seg, small batch → GN (groups=32)
- Style transfer → IN
```

### 11. **Code**: "Implement Focal Loss for imbalanced classification."
```python
class FocalLoss(nn.Module):
    """Focal Loss for Dense Object Detection (Lin et al., 2017)
    FL(p_t) = -α_t (1 - p_t)^γ log(p_t)
    """
    def __init__(self, alpha=0.25, gamma=2.0, reduction='mean'):
        super().__init__()
        self.alpha = alpha
        self.gamma = gamma
        self.reduction = reduction
    
    def forward(self, inputs, targets):
        # inputs: (N, C) logits, targets: (N,) class indices
        ce_loss = F.cross_entropy(inputs, targets, reduction='none')
        pt = torch.exp(-ce_loss)  # p_t
        
        # Alpha weighting (can be per-class tensor)
        if isinstance(self.alpha, (list, tuple, torch.Tensor)):
            alpha_t = self.alpha[targets]
        else:
            alpha_t = self.alpha
        
        focal_loss = alpha_t * (1 - pt) ** self.gamma * ce_loss
        
        if self.reduction == 'mean':
            return focal_loss.mean()
        elif self.reduction == 'sum':
            return focal_loss.sum()
        return focal_loss

# Binary Focal Loss (for BCEWithLogits)
class BinaryFocalLoss(nn.Module):
    def __init__(self, alpha=0.25, gamma=2.0):
        super().__init__()
        self.alpha = alpha
        self.gamma = gamma
    
    def forward(self, inputs, targets):
        # inputs: logits, targets: 0/1
        p = torch.sigmoid(inputs)
        ce = F.binary_cross_entropy_with_logits(inputs, targets, reduction='none')
        p_t = p * targets + (1 - p) * (1 - targets)
        alpha_t = self.alpha * targets + (1 - self.alpha) * (1 - targets)
        loss = alpha_t * (1 - p_t) ** self.gamma * ce
        return loss.mean()
```

### 12. **Design**: "How to handle very long sequences in Transformers?"
```
O(n²) PROBLEM: Self-attention is quadratic in sequence length.

SOLUTIONS:

1. SPARSE ATTENTION:
   - Longformer: Local + global tokens
   - BigBird: Random + local + global
   - Sparse Transformer: Fixed patterns

2. LINEAR ATTENTION:
   - Performer: FAVOR+ (kernel approximation)
   - Linear Transformer: φ(Q)φ(K)^T V
   - O(n) complexity

3. MEMORY-EFFICIENT EXACT:
   - FlashAttention: IO-aware, kernel fusion
   - xFormers: Memory-efficient attention
   - Block-sparse

4. RECURRENCE:
   - Transformer-XL: Segment-level recurrence
   - Compressive Transformer: Compressed memory
   - RWKV: RNN-style linear attention

5. HIERARCHICAL:
   - Swin Transformer: Local windows + shifted
   - HAT: Hierarchical attention

6. PRACTICAL CHOICES:
   - < 2048: Standard attention + FlashAttention
   - 2048-16K: Longformer / BigBird
   - > 16K: Performer / Linear Transformer / RWKV
   - Need exact: FlashAttention + gradient checkpointing
```

### 13. **Code**: "Implement knowledge distillation loss."
```python
def distillation_loss(student_logits, teacher_logits, labels, 
                      temperature=4.0, alpha=0.7):
    """
    Hinton et al. 2015: Distilling the Knowledge in a Neural Network
    L = α * T² * KL(softmax(teacher/T) || softmax(student/T)) 
        + (1-α) * CE(student, labels)
    """
    # Soft targets
    teacher_probs = F.softmax(teacher_logits / temperature, dim=1)
    student_log_probs = F.log_softmax(student_logits / temperature, dim=1)
    
    kl_loss = F.kl_div(student_log_probs, teacher_probs, reduction='batchmean')
    kl_loss *= temperature ** 2  # Scale by T²
    
    # Hard targets
    ce_loss = F.cross_entropy(student_logits, labels)
    
    return alpha * kl_loss + (1 - alpha) * ce_loss

# Training loop
for x, y in loader:
    with torch.no_grad():
        teacher_logits = teacher(x)
    student_logits = student(x)
    loss = distillation_loss(student_logits, teacher_logits, y)
    loss.backward()
    optimizer.step()
```

### 14. **Conceptual**: "How do you debug a neural network that isn't learning?"

```
DEBUGGING CHECKLIST:

1. OVERFIT SINGLE BATCH:
   - Train on 1 batch, loss should → 0
   - If not: model bug, loss bug, data bug

2. CHECK GRADIENTS:
   - NaN gradients? → loss explosion, bad init, numerical issues
   - Zero gradients? → dead ReLU, frozen layers, wrong params
   - Exploding? → clip_grad_norm, lower LR, better init

3. VISUALIZE:
   - Input data (augmentation correct?)
   - Model predictions (random?)
   - Loss curves (train/val)
   - Gradient norms per layer
   - Weight distributions

4. COMMON BUGS:
   - Forgot model.train() / model.eval()
   - Wrong loss for task (BCE vs CE)
   - Labels wrong format (one-hot vs indices)
   - Data leakage in validation
   - BatchNorm running stats not updated
   - Dropout not disabled at eval
   - Gradient accumulation not zeroing

5. LEARNING RATE:
   - Too high: loss diverges, NaN
   - Too low: no progress
   - LR finder: exponentially increasing LR, plot loss

6. TOOLS:
   - TensorBoard / WandB
   - torch.autograd.detect_anomaly()
   - torchviz for graph visualization
   - Hook for gradient monitoring
```

### 15. **Code**: "Learning rate finder implementation."
```python
def lr_finder(model, loader, criterion, optimizer, 
              start_lr=1e-7, end_lr=10, num_iter=100, device='cuda'):
    """Leslie Smith's LR Range Test"""
    model.train()
    
    # Exponential LR schedule
    lrs = np.logspace(np.log10(start_lr), np.log10(end_lr), num_iter)
    losses = []
    smooth_losses = []
    beta = 0.98
    avg_loss = 0
    
    for i, (x, y) in enumerate(loader):
        if i >= num_iter:
            break
        
        x, y = x.to(device), y.to(device)
        
        # Set LR
        lr = lrs[i]
        for param_group in optimizer.param_groups:
            param_group['lr'] = lr
        
        optimizer.zero_grad()
        output = model(x)
        loss = criterion(output, y)
        
        # Smooth loss
        avg_loss = beta * avg_loss + (1 - beta) * loss.item()
        smoothed = avg_loss / (1 - beta ** (i + 1))
        
        losses.append(loss.item())
        smooth_losses.append(smoothed)
        
        # Stop if loss explodes
        if i > 10 and smoothed > 4 * min(smooth_losses):
            break
        
        loss.backward()
        optimizer.step()
    
    # Plot
    import matplotlib.pyplot as plt
    plt.plot(lrs[:len(smooth_losses)], smooth_losses)
    plt.xscale('log')
    plt.xlabel('Learning Rate')
    plt.ylabel('Loss')
    plt.title('LR Range Test')
    plt.show()
    
    # Find best LR (steepest descent)
    grads = np.gradient(smooth_losses)
    best_idx = np.argmin(grads)
    best_lr = lrs[best_idx]
    
    print(f"Suggested LR: {best_lr:.2e}")
    return best_lr
```

---

## QUICK REFERENCE: DEEP LEARNING CHEAT SHEET

### Architecture Selection
| Task | Architecture |
|------|-------------|
| Image classification | ResNet, EfficientNet, ConvNeXt, ViT |
| Object detection | YOLO, Faster R-CNN, DETR |
| Segmentation | U-Net, DeepLab, Mask R-CNN, SegFormer |
| NLP classification | BERT, RoBERTa, DeBERTa |
| Generation | GPT, LLaMA, T5 |
| Sequence labeling | BiLSTM+CRF, BERT+CRF |
| Time series | LSTM, TCN, Transformer, Informer |
| Recommendation | Two-tower, SASRec, BERT4Rec |

### Key Hyperparameters
| Component | Typical Values |
|-----------|----------------|
| Learning rate | 1e-4 to 3e-4 (transformer), 1e-3 (CNN) |
| Batch size | 32-512 (scale LR with √batch) |
| Weight decay | 0.01-0.1 (AdamW) |
| Dropout | 0.1-0.3 |
| Label smoothing | 0.1 |
| Gradient clip | 1.0 |
| Warmup steps | 500-10000 (1-5% of total) |

### Common Issues & Fixes
| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Loss NaN | Exploding grads | Clip grad, lower LR, check init |
| Loss stuck | Dead ReLU, LR too low | LeakyReLU, LR finder |
| Train >> Val | Overfitting | Dropout, weight decay, data aug, smaller model |
| Val >> Train | Underfitting | Larger model, more epochs, less reg |
| Unstable loss | Bad init, high LR | Xavier/Kaiming init, warmup |
| Slow training | No AMP, small batch | Mixed precision, larger batch |
| OOM | Model/batch too large | Gradient accumulation, gradient checkpointing, FSDP |

### PyTorch Debugging Commands
```python
# Check gradients
for name, param in model.named_parameters():
    if param.grad is not None:
        print(name, param.grad.abs().mean().item())

# Anomaly detection
torch.autograd.set_detect_anomaly(True)

# Model summary
from torchinfo import summary
summary(model, input_size=(1, 3, 224, 224))

# Graph visualization
from torchviz import make_dot
make_dot(output, params=dict(model.named_parameters())).render()
```