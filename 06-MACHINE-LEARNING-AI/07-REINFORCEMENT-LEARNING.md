# REINFORCEMENT LEARNING
## Exam-Style Study Notes

**Roadmap Section:** Reinforcement Learning
**Prerequisites:** Probability, calculus, linear algebra, Python, deep learning basics

---

## 1. RL FUNDAMENTALS

### WHY? (Problem Statement)
**Sequential decision making.** Learn optimal policy through interaction with environment:
- Agent observes state, takes action, receives reward
- Goal: Maximize cumulative reward (return)

### HOW? (Core Concepts)

#### MDP (Markov Decision Process)
```math
\mathcal{M} = (\mathcal{S}, \mathcal{A}, P, R, \gamma)

- \mathcal{S}: State space
- \mathcal{A}: Action space
- P(s'|s,a): Transition probability
- R(s,a,s'): Reward function
- \gamma \in [0,1]: Discount factor

MARKOV PROPERTY:
P(s_{t+1}|s_t, a_t, s_{t-1}, ...) = P(s_{t+1}|s_t, a_t)
```

#### Key Definitions
| Term | Definition |
|------|------------|
| **Policy** π(a|s) | Probability of taking action a in state s |
| **Value Function** V^π(s) | Expected return from state s under policy π |
| **Q-Function** Q^π(s,a) | Expected return from (s,a) under policy π |
| **Return** G_t | Σ_{k=0}^∞ γ^k R_{t+k+1} |
| **Optimal Value** V*(s) = max_π V^π(s) | Best achievable value |
| **Optimal Q** Q*(s,a) = max_π Q^π(s,a) | Best achievable Q-value |

#### Bellman Equations
```math
# Value function
V^π(s) = Σ_a π(a|s) Σ_{s'} P(s'|s,a) [R(s,a,s') + γ V^π(s')]

# Q-function
Q^π(s,a) = Σ_{s'} P(s'|s,a) [R(s,a,s') + γ Σ_{a'} π(a'|s') Q^π(s',a')]

# Optimal Bellman (Bellman Optimality)
V*(s) = max_a Σ_{s'} P(s'|s,a) [R(s,a,s') + γ V*(s')]
Q*(s,a) = Σ_{s'} P(s'|s,a) [R(s,a,s') + γ max_{a'} Q*(s',a')]
```

---

## 2. VALUE-BASED METHODS

### WHY? (Problem Statement)
**Learn value function, derive policy.** Estimate Q-values, act greedily.

### HOW? (Algorithms)

#### Q-Learning (Off-policy TD Control)
```math
Q(s,a) \leftarrow Q(s,a) + \alpha [r + \gamma \max_{a'} Q(s',a') - Q(s,a)]

- Off-policy: learns optimal policy while following behavior policy
- Converges to Q* with sufficient exploration
```

```python
import numpy as np

class QLearning:
    def __init__(self, n_states, n_actions, alpha=0.1, gamma=0.99, epsilon=0.1):
        self.Q = np.zeros((n_states, n_actions))
        self.alpha = alpha      # Learning rate
        self.gamma = gamma      # Discount factor
        self.epsilon = epsilon  # Exploration rate
    
    def act(self, state, train=True):
        if train and np.random.random() < self.epsilon:
            return np.random.randint(self.Q.shape[1])  # Explore
        return np.argmax(self.Q[state])  # Exploit
    
    def update(self, state, action, reward, next_state, done):
        if done:
            target = reward
        else:
            target = reward + self.gamma * np.max(self.Q[next_state])
        
        self.Q[state, action] += self.alpha * (target - self.Q[state, action])
    
    def decay_epsilon(self, decay=0.995, min_epsilon=0.01):
        self.epsilon = max(self.epsilon * decay, min_epsilon)
```

#### SARSA (On-policy TD Control)
```math
Q(s,a) \leftarrow Q(s,a) + \alpha [r + \gamma Q(s',a') - Q(s,a)]

- On-policy: learns value of current policy (including exploration)
- a' is the action actually taken in s' (not max)
- More conservative, safer in cliff-walking scenarios
```

```python
class SARSA:
    def update(self, state, action, reward, next_state, next_action, done):
        if done:
            target = reward
        else:
            target = reward + self.gamma * self.Q[next_state, next_action]
        
        self.Q[state, action] += self.alpha * (target - self.Q[state, action])
```

---

## 3. DEEP Q-NETWORKS (DQN)

### WHY? (Problem Statement)
**Q-learning with function approximation.** Handle large/continuous state spaces with neural networks.

### HOW? (DQN Architecture)

#### Key Innovations
| Innovation | Purpose |
|------------|---------|
| **Experience Replay** | Break correlation, reuse data |
| **Target Network** | Stable targets (slowly updated) |
| **Double DQN** | Reduce overestimation bias |
| **Dueling DQN** | Separate value and advantage |
| **Prioritized Replay** | Sample important transitions more |

```python
import torch
import torch.nn as nn
import torch.optim as optim
import random
from collections import deque

class DQN(nn.Module):
    def __init__(self, state_dim, action_dim, hidden_dim=128):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(state_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, action_dim)
        )
    
    def forward(self, x):
        return self.net(x)

# Dueling DQN
class DuelingDQN(nn.Module):
    def __init__(self, state_dim, action_dim, hidden_dim=128):
        super().__init__()
        self.feature = nn.Sequential(
            nn.Linear(state_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU()
        )
        self.value = nn.Linear(hidden_dim, 1)
        self.advantage = nn.Linear(hidden_dim, action_dim)
    
    def forward(self, x):
        f = self.feature(x)
        v = self.value(f)
        a = self.advantage(f)
        # Q = V + (A - mean(A))
        return v + a - a.mean(dim=1, keepdim=True)

class DQNAgent:
    def __init__(self, state_dim, action_dim, lr=1e-3, gamma=0.99, 
                 epsilon=1.0, epsilon_decay=0.995, epsilon_min=0.01,
                 buffer_size=100000, batch_size=64, target_update=1000):
        
        self.action_dim = action_dim
        self.gamma = gamma
        self.epsilon = epsilon
        self.epsilon_decay = epsilon_decay
        self.epsilon_min = epsilon_min
        self.batch_size = batch_size
        self.target_update = target_update
        self.step_count = 0
        
        # Networks
        self.q_net = DQN(state_dim, action_dim)
        self.target_net = DQN(state_dim, action_dim)
        self.target_net.load_state_dict(self.q_net.state_dict())
        
        self.optimizer = optim.Adam(self.q_net.parameters(), lr=lr)
        self.memory = deque(maxlen=buffer_size)
        self.loss_fn = nn.MSELoss()
    
    def act(self, state, train=True):
        if train and random.random() < self.epsilon:
            return random.randint(0, self.action_dim - 1)
        
        with torch.no_grad():
            state = torch.FloatTensor(state).unsqueeze(0)
            q_values = self.q_net(state)
            return q_values.argmax().item()
    
    def remember(self, state, action, reward, next_state, done):
        self.memory.append((state, action, reward, next_state, done))
    
    def train_step(self):
        if len(self.memory) < self.batch_size:
            return
        
        batch = random.sample(self.memory, self.batch_size)
        states, actions, rewards, next_states, dones = zip(*batch)
        
        states = torch.FloatTensor(np.array(states))
        actions = torch.LongTensor(actions).unsqueeze(1)
        rewards = torch.FloatTensor(rewards).unsqueeze(1)
        next_states = torch.FloatTensor(np.array(next_states))
        dones = torch.FloatTensor(dones).unsqueeze(1)
        
        # Current Q values
        current_q = self.q_net(states).gather(1, actions)
        
        # Target Q values (Double DQN)
        with torch.no_grad():
            # Action selection from online network
            next_actions = self.q_net(next_states).argmax(1, keepdim=True)
            # Value evaluation from target network
            next_q = self.target_net(next_states).gather(1, next_actions)
            target_q = rewards + self.gamma * next_q * (1 - dones)
        
        loss = self.loss_fn(current_q, target_q)
        
        self.optimizer.zero_grad()
        loss.backward()
        self.optimizer.step()
        
        # Update target network
        self.step_count += 1
        if self.step_count % self.target_update == 0:
            self.target_net.load_state_dict(self.q_net.state_dict())
        
        # Decay epsilon
        self.epsilon = max(self.epsilon * self.epsilon_decay, self.epsilon_min)
        
        return loss.item()
```

---

## 4. POLICY GRADIENT METHODS

### WHY? (Problem Statement)
**Directly optimize policy.** Learn parameterized policy π_θ(a|s) via gradient ascent on expected return.

### HOW? (REINFORCE & Actor-Critic)

#### REINFORCE (Monte Carlo Policy Gradient)
```math
\nabla_θ J(θ) = E[ Σ_t ∇_θ log π_θ(a_t|s_t) G_t ]

# With baseline (variance reduction)
\nabla_θ J(θ) = E[ Σ_t ∇_θ log π_θ(a_t|s_t) (G_t - b(s_t)) ]
```

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.distributions import Categorical

class PolicyNetwork(nn.Module):
    def __init__(self, state_dim, action_dim, hidden_dim=128):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(state_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, action_dim),
            nn.Softmax(dim=-1)
        )
    
    def forward(self, x):
        return self.net(x)

class REINFORCE:
    def __init__(self, state_dim, action_dim, lr=1e-3, gamma=0.99):
        self.gamma = gamma
        self.policy = PolicyNetwork(state_dim, action_dim)
        self.optimizer = optim.Adam(self.policy.parameters(), lr=lr)
        self.log_probs = []
        self.rewards = []
    
    def act(self, state):
        state = torch.FloatTensor(state).unsqueeze(0)
        probs = self.policy(state)
        dist = Categorical(probs)
        action = dist.sample()
        self.log_probs.append(dist.log_prob(action))
        return action.item()
    
    def store_reward(self, reward):
        self.rewards.append(reward)
    
    def update(self):
        # Compute returns (discounted cumulative rewards)
        returns = []
        G = 0
        for r in reversed(self.rewards):
            G = r + self.gamma * G
            returns.insert(0, G)
        
        returns = torch.FloatTensor(returns)
        # Normalize (variance reduction)
        returns = (returns - returns.mean()) / (returns.std() + 1e-8)
        
        # Policy gradient loss
        loss = 0
        for log_prob, G in zip(self.log_probs, returns):
            loss += -log_prob * G
        
        self.optimizer.zero_grad()
        loss.backward()
        self.optimizer.step()
        
        # Clear buffers
        self.log_probs = []
        self.rewards = []
        return loss.item()
```

#### Actor-Critic (Advantage Actor-Critic - A2C)
```python
import torch
import torch.nn as nn
import torch.optim as optim

class ActorCritic(nn.Module):
    def __init__(self, state_dim, action_dim, hidden_dim=128):
        super().__init__()
        self.shared = nn.Sequential(
            nn.Linear(state_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU()
        )
        self.actor = nn.Linear(hidden_dim, action_dim)
        self.critic = nn.Linear(hidden_dim, 1)
    
    def forward(self, x):
        features = self.shared(x)
        return self.actor(features), self.critic(features)

class A2C:
    def __init__(self, state_dim, action_dim, lr=1e-3, gamma=0.99, entropy_coef=0.01):
        self.gamma = gamma
        self.entropy_coef = entropy_coef
        self.model = ActorCritic(state_dim, action_dim)
        self.optimizer = optim.Adam(self.model.parameters(), lr=lr)
    
    def act(self, state):
        state = torch.FloatTensor(state).unsqueeze(0)
        logits, value = self.model(state)
        probs = torch.softmax(logits, dim=-1)
        dist = torch.distributions.Categorical(probs)
        action = dist.sample()
        log_prob = dist.log_prob(action)
        entropy = dist.entropy()
        return action.item(), log_prob, value, entropy
    
    def update(self, log_probs, values, rewards, entropies, next_value, dones):
        # Compute returns
        returns = []
        G = next_value
        for r, done in zip(reversed(rewards), reversed(dones)):
            G = r + self.gamma * G * (1 - done)
            returns.insert(0, G)
        
        returns = torch.stack(returns)
        values = torch.stack(values)
        log_probs = torch.stack(log_probs)
        entropies = torch.stack(entropies)
        
        # Advantage
        advantages = returns - values.detach()
        
        # Actor loss
        actor_loss = -(log_probs * advantages.detach()).mean()
        
        # Critic loss
        critic_loss = advantages.pow(2).mean()
        
        # Entropy bonus
        entropy_loss = -entropies.mean()
        
        loss = actor_loss + 0.5 * critic_loss + self.entropy_coef * entropy_loss
        
        self.optimizer.zero_grad()
        loss.backward()
        self.optimizer.step()
        
        return loss.item()
```

---

## 5. ADVANCED ALGORITHMS

### PPO (Proximal Policy Optimization)
```python
# Key idea: Clip objective to prevent large policy updates
L^{CLIP}(θ) = E[ min(r_t(θ) A_t, clip(r_t(θ), 1-ε, 1+ε) A_t) ]

# Where r_t(θ) = π_θ(a_t|s_t) / π_{θ_old}(a_t|s_t)

class PPO:
    def __init__(self, state_dim, action_dim, lr=3e-4, gamma=0.99, 
                 clip_eps=0.2, epochs=10, batch_size=64):
        self.gamma = gamma
        self.clip_eps = clip_eps
        self.epochs = epochs
        self.batch_size = batch_size
        
        self.model = ActorCritic(state_dim, action_dim)
        self.optimizer = optim.Adam(self.model.parameters(), lr=lr)
    
    def compute_gae(self, rewards, values, dones, next_value, gamma=0.99, lam=0.95):
        """Generalized Advantage Estimation"""
        advantages = []
        gae = 0
        for r, v, done, next_v in zip(reversed(rewards), reversed(values), 
                                        reversed(dones), reversed([next_value] + values[:-1])):
            delta = r + gamma * next_v * (1 - done) - v
            gae = delta + gamma * lam * (1 - done) * gae
            advantages.insert(0, gae)
        return torch.stack(advantages)
    
    def update(self, states, actions, log_probs_old, rewards, values, dones):
        # Compute advantages
        with torch.no_grad():
            _, next_value = self.model(torch.FloatTensor(states[-1]).unsqueeze(0))
            advantages = self.compute_gae(rewards, values, dones, next_value.item())
            returns = advantages + values.detach()
        
        # Normalize advantages
        advantages = (advantages - advantages.mean()) / (advantages.std() + 1e-8)
        
        # PPO epochs
        for _ in range(self.epochs):
            # Sample batch
            idx = np.random.choice(len(states), self.batch_size)
            batch_states = torch.FloatTensor(np.array(states)[idx])
            batch_actions = torch.LongTensor(np.array(actions)[idx])
            batch_log_probs_old = torch.stack(log_probs_old)[idx]
            batch_advantages = advantages[idx]
            batch_returns = returns[idx]
            
            # Forward
            logits, values_new = self.model(batch_states)
            probs = torch.softmax(logits, dim=-1)
            dist = torch.distributions.Categorical(probs)
            log_probs_new = dist.log_prob(batch_actions)
            entropy = dist.entropy().mean()
            
            # Ratio
            ratio = (log_probs_new - batch_log_probs_old).exp()
            
            # Clipped objective
            surr1 = ratio * batch_advantages
            surr2 = torch.clamp(ratio, 1 - self.clip_eps, 1 + self.clip_eps) * batch_advantages
            actor_loss = -torch.min(surr1, surr2).mean()
            
            # Critic loss
            critic_loss = (values_new.squeeze() - batch_returns).pow(2).mean()
            
            loss = actor_loss + 0.5 * critic_loss - 0.01 * entropy
            
            self.optimizer.zero_grad()
            loss.backward()
            self.optimizer.step()
```

---

## 6. EXPLORATION STRATEGIES

| Strategy | Description | When |
|----------|-------------|------|
| **ε-greedy** | Random action with prob ε | Simple, discrete actions |
| **Boltzmann/Softmax** | Sample from softmax(Q/τ) | Better than ε-greedy |
| **UCB** | Optimism in face of uncertainty | Bandits, tabular |
| **Thompson Sampling** | Bayesian posterior sampling | Bandits |
| **Entropy Regularization** | Add entropy bonus to loss | Policy gradients |
| **Noisy Networks** | Parameter noise for exploration | DQN |
| **Curiosity/ICM** | Intrinsic reward for novelty | Sparse rewards |

---

## PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Target network** | Direct Q-target | Unstable, divergence |
| **Experience replay** | Online updates | Correlated samples, forgetting |
| **Double DQN** | Max overestimation | Overoptimistic values |
| **Reward shaping** | Sparse rewards only | Hard to learn |
| **Reward clipping** | Unbounded rewards | Gradient explosion |
| **GAE** | Monte Carlo returns | High variance |
| **Entropy bonus** | No exploration bonus | Premature convergence |
| **Gradient clipping** | No clipping | Gradient explosion in PPO |

---

## INTERVIEW QUESTIONS (Top 15)

### 1. **Conceptual**: "Q-Learning vs SARSA - what's the difference?"

```
Q-LEARNING (Off-policy):
- Updates: Q(s,a) ← Q(s,a) + α[r + γ max_a' Q(s',a') - Q(s,a)]
- Learns optimal policy while following behavior policy
- More aggressive, can be unstable
- Max operator → overestimation bias

SARSA (On-policy):
- Updates: Q(s,a) ← Q(s,a) + α[r + γ Q(s',a') - Q(s,a)]
- a' is actual action taken (includes exploration)
- Learns value of current policy (with exploration)
- More conservative, safer (Cliff Walking example)

CLIFF WALKING:
- Q-learning learns optimal path but falls off cliff during exploration
- SARSA learns safer path away from cliff
```

### 2. **Code**: "Implement DQN with experience replay and target network."

```python
# See full implementation above in DQNAgent class
# Key components:
# 1. Replay buffer (deque)
# 2. Target network (periodic update)
# 3. Epsilon-greedy exploration
# 4. Double DQN (action from online, value from target)
```

### 3. **Design**: "How would you train an agent for a game with sparse rewards?"

```
SPARSE REWARD SOLUTIONS:

1. REWARD SHAPING:
   - Add intermediate rewards (potential-based: F(s,a,s') = γΦ(s') - Φ(s))
   - Must preserve optimal policy (Ng et al. 1999)

2. CURIOSITY / INTRINSIC MOTIVATION (ICM):
   - Forward model: predict next state from (s,a)
   - Inverse model: predict action from (s,s')
   - Intrinsic reward = ||φ(s') - φ̂(s')||²
   - Encourages exploration of novel states

3. HIERARCHICAL RL:
   - Options / skills (temporally extended actions)
   - Learn sub-policies for subgoals
   - HRL: high-level policy selects options

4. HER (Hindsight Experience Replay):
   - For goal-based tasks
   - Relabel failed trajectories with achieved goals
   - Learn from failure

5. REWARD MODELING:
   - Learn reward function from demonstrations/preferences
   - IRL / Inverse RL / Preference learning

6. DEMONSTRATIONS / IMITATION:
   - Behavioral Cloning (BC) pretraining
   - DAgger (Dataset Aggregation)
   - GAIL (Generative Adversarial Imitation Learning)
```

### 4. **Code**: "Implement REINFORCE with baseline."

```python
class REINFORCEWithBaseline:
    def __init__(self, state_dim, action_dim, lr=1e-3, gamma=0.99):
        self.gamma = gamma
        
        # Policy network
        self.policy = nn.Sequential(
            nn.Linear(state_dim, 128),
            nn.ReLU(),
            nn.Linear(128, 128),
            nn.ReLU(),
            nn.Linear(128, action_dim)
        )
        
        # Value network (baseline)
        self.value = nn.Sequential(
            nn.Linear(state_dim, 128),
            nn.ReLU(),
            nn.Linear(128, 128),
            nn.ReLU(),
            nn.Linear(128, 1)
        )
        
        self.policy_optim = optim.Adam(self.policy.parameters(), lr=lr)
        self.value_optim = optim.Adam(self.value.parameters(), lr=lr)
        
        self.log_probs = []
        self.rewards = []
        self.states = []
    
    def act(self, state):
        state_t = torch.FloatTensor(state).unsqueeze(0)
        logits = self.policy(state_t)
        probs = torch.softmax(logits, dim=-1)
        dist = Categorical(probs)
        action = dist.sample()
        self.log_probs.append(dist.log_prob(action))
        self.states.append(state_t)
        return action.item()
    
    def store_reward(self, reward):
        self.rewards.append(reward)
    
    def update(self):
        # Compute returns
        returns = []
        G = 0
        for r in reversed(self.rewards):
            G = r + self.gamma * G
            returns.insert(0, G)
        returns = torch.FloatTensor(returns)
        
        # Value predictions
        states = torch.cat(self.states)
        values = self.value(states).squeeze()
        
        # Advantages
        advantages = returns - values.detach()
        
        # Policy loss
        log_probs = torch.stack(self.log_probs)
        policy_loss = -(log_probs * advantages).mean()
        
        # Value loss
        value_loss = F.mse_loss(values, returns)
        
        # Update
        self.policy_optim.zero_grad()
        policy_loss.backward()
        self.policy_optim.step()
        
        self.value_optim.zero_grad()
        value_loss.backward()
        self.value_optim.step()
        
        self.log_probs = []
        self.rewards = []
        self.states = []
        
        return policy_loss.item(), value_loss.item()
```

### 5. **Conceptual**: "Explain the bias-variance tradeoff in RL."

```
VALUE-BASED (Q-Learning, DQN):
- LOW VARIANCE (bootstrapping)
- HIGH BIAS (function approximation, bootstrapping error)
- Sample efficient

POLICY GRADIENT (REINFORCE):
- HIGH VARIANCE (Monte Carlo returns)
- LOW BIAS (unbiased gradient estimate)
- Sample inefficient

ACTOR-CRITIC:
- MEDIUM VARIANCE (TD errors as critic)
- MEDIUM BIAS (critic approximation)
- Best of both worlds

VARIANCE REDUCTION TECHNIQUES:
1. Baseline (value function)
2. GAE (λ-returns)
3. Normalized advantages
4. Entropy regularization
5. Mini-batch updates

BIAS SOURCES:
1. Function approximation
2. Bootstrapping
3. Off-policy corrections (importance sampling)
4. Clipping (PPO)
```

### 6. **Code**: "Implement GAE (Generalized Advantage Estimation)."

```python
def compute_gae(rewards, values, dones, next_value, gamma=0.99, lam=0.95):
    """
    GAE(λ) = Σ_{l=0}^∞ (γλ)^l δ_{t+l}
    
    Where δ_t = r_t + γ V(s_{t+1}) - V(s_t)  (TD error)
    """
    advantages = []
    gae = 0
    
    # values: [V(s_0), V(s_1), ..., V(s_{T-1})]
    # next_value: V(s_T)
    all_values = values + [next_value]
    
    for t in reversed(range(len(rewards))):
        delta = rewards[t] + gamma * all_values[t+1] * (1 - dones[t]) - all_values[t]
        gae = delta + gamma * lam * (1 - dones[t]) * gae
        advantages.insert(0, gae)
    
    return torch.stack(advantages)

# λ = 0: TD(0) advantage (low variance, high bias)
# λ = 1: Monte Carlo advantage (high variance, low bias)
# λ ∈ (0,1): Tradeoff
```

### 7. **Conceptual**: "On-policy vs Off-policy - when to use which?"

```
ON-POLICY (SARSA, REINFORCE, A2C, PPO, TRPO):
✓ Learn value of CURRENT policy
✓ Stable, monotonic improvement (with trust region)
✓ Simpler theory
✗ Lower sample efficiency (must collect new data per update)
✗ Can't reuse old data
✓ Use when: safety critical, stable learning needed, can afford samples

OFF-POLICY (Q-Learning, DQN, DDPG, TD3, SAC):
✓ Learn OPTIMAL policy from ANY behavior policy
✓ High sample efficiency (replay buffer)
✓ Can reuse old data, offline RL
✗ More unstable, divergence risk
✗ Overestimation bias
✓ Use when: sample efficiency critical, offline data available

HYBRID:
- PPO: On-policy but with multiple epochs per batch
- SAC: Off-policy with entropy regularization
- IQN/QRDQN: Off-policy distributional
```

### 8. **Code**: "Implement PPO clipped objective."

```python
def ppo_loss(log_probs_new, log_probs_old, advantages, clip_eps=0.2):
    """
    L^{CLIP} = E[min(r * A, clip(r, 1-ε, 1+ε) * A)]
    
    Where r = π_new / π_old = exp(log_probs_new - log_probs_old)
    """
    ratio = (log_probs_new - log_probs_old).exp()
    
    surr1 = ratio * advantages
    surr2 = torch.clamp(ratio, 1 - clip_eps, 1 + clip_eps) * advantages
    
    loss = -torch.min(surr1, surr2).mean()
    return loss

# In PPO update:
for _ in range(epochs):
    for batch in minibatches:
        logits, values = model(batch.states)
        probs = F.softmax(logits, dim=-1)
        dist = Categorical(probs)
        new_log_probs = dist.log_prob(batch.actions)
        entropy = dist.entropy().mean()
        
        # Advantage already computed
        actor_loss = ppo_loss(new_log_probs, batch.old_log_probs, batch.advantages)
        critic_loss = F.mse_loss(values.squeeze(), batch.returns)
        
        loss = actor_loss + 0.5 * critic_loss - 0.01 * entropy
        loss.backward()
        optimizer.step()
```

### 9. **Design**: "How would you design an RL system for recommendation?"

```
RL FOR RECOMMENDATION SYSTEM:

1. STATE REPRESENTATION:
   - User history (items, timestamps, actions)
   - User features (demographics, embeddings)
   - Context (time, device, session)
   - Item candidates (candidate pool)

2. ACTION SPACE:
   - Discrete: Select K items from candidate pool (combinatorial → slate Q-learning)
   - Continuous: Score all items, top-K (deterministic policy)
   - Slate: Choose set of items jointly

3. REWARD:
   - Immediate: Click, dwell time, conversion
   - Delayed: Long-term engagement, retention
   - Multi-objective: Click + Diversity + Revenue

4. ALGORITHM CHOICES:
   - Bandit (contextual): LinUCB, Thompson Sampling, Neural Bandits
   - RL: DQN (slate), Actor-Critic, PPO
   - Offline RL: BCQ, CQL (learn from logs)

5. CHALLENGES:
   - Large action space (millions of items) → Two-tower, ANN
   - Delayed rewards → TD(λ), GAE, reward shaping
   - Non-stationarity → Periodic retraining, online learning
   - Exploration vs exploitation → ε-greedy, Thompson, Entropy
   - Off-policy evaluation → OPE (IPS, DR, FQE)

6. ARCHITECTURE:
   - Two-tower: User tower + Item tower → dot product
   - State encoder: Transformer / GRU for history
   - Critic: Q-network or Value network
   - Actor: Policy network (categorical over candidates)
```

### 10. **Code**: "Soft Actor-Critic (SAC) implementation outline."

```python
class SAC:
    def __init__(self, state_dim, action_dim, lr=3e-4, gamma=0.99, tau=0.005, alpha=0.2):
        self.gamma = gamma
        self.tau = tau
        self.alpha = alpha  # Entropy temperature
        
        # Actor (policy)
        self.actor = GaussianPolicy(state_dim, action_dim)
        self.actor_optim = optim.Adam(self.actor.parameters(), lr=lr)
        
        # Critics (twin Q-networks)
        self.critic1 = QNetwork(state_dim, action_dim)
        self.critic2 = QNetwork(state_dim, action_dim)
        self.critic1_optim = optim.Adam(self.critic1.parameters(), lr=lr)
        self.critic2_optim = optim.Adam(self.critic2.parameters(), lr=lr)
        
        # Target critics
        self.target_critic1 = QNetwork(state_dim, action_dim)
        self.target_critic2 = QNetwork(state_dim, action_dim)
        self.target_critic1.load_state_dict(self.critic1.state_dict())
        self.target_critic2.load_state_dict(self.critic2.state_dict())
        
        # Entropy tuning (automatic)
        self.target_entropy = -action_dim
        self.log_alpha = torch.zeros(1, requires_grad=True)
        self.alpha_optim = optim.Adam([self.log_alpha], lr=lr)
    
    def update(self, batch):
        states, actions, rewards, next_states, dones = batch
        
        # === Critic Update ===
        with torch.no_grad():
            # Sample next action from policy
            next_actions, next_log_probs = self.actor.sample(next_states)
            # Target Q values
            target_q1 = self.target_critic1(next_states, next_actions)
            target_q2 = self.target_critic2(next_states, next_actions)
            target_q = torch.min(target_q1, target_q2) - self.alpha * next_log_probs
            target_q = rewards + self.gamma * (1 - dones) * target_q
        
        # Current Q values
        q1 = self.critic1(states, actions)
        q2 = self.critic2(states, actions)
        
        critic1_loss = F.mse_loss(q1, target_q)
        critic2_loss = F.mse_loss(q2, target_q)
        
        self.critic1_optim.zero_grad()
        critic1_loss.backward()
        self.critic1_optim.step()
        
        self.critic2_optim.zero_grad()
        critic2_loss.backward()
        self.critic2_optim.step()
        
        # === Actor Update ===
        actions_new, log_probs_new = self.actor.sample(states)
        q1_new = self.critic1(states, actions_new)
        q2_new = self.critic2(states, actions_new)
        q_new = torch.min(q1_new, q2_new)
        
        actor_loss = (self.alpha * log_probs_new - q_new).mean()
        
        self.actor_optim.zero_grad()
        actor_loss.backward()
        self.actor_optim.step()
        
        # === Alpha Update ===
        alpha_loss = -(self.log_alpha * (log_probs_new + self.target_entropy).detach()).mean()
        
        self.alpha_optim.zero_grad()
        alpha_loss.backward()
        self.alpha_optim.step()
        
        self.alpha = self.log_alpha.exp().item()
        
        # === Target Update (Polyak) ===
        for param, target_param in zip(self.critic1.parameters(), self.target_critic1.parameters()):
            target_param.data.copy_(self.tau * param.data + (1 - self.tau) * target_param.data)
        for param, target_param in zip(self.critic2.parameters(), self.target_critic2.parameters()):
            target_param.data.copy_(self.tau * param.data + (1 - self.tau) * target_param.data)

class GaussianPolicy(nn.Module):
    def __init__(self, state_dim, action_dim, hidden_dim=256):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(state_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU()
        )
        self.mean = nn.Linear(hidden_dim, action_dim)
        self.log_std = nn.Linear(hidden_dim, action_dim)
    
    def forward(self, x):
        features = self.net(x)
        mean = self.mean(features)
        log_std = self.log_std(features).clamp(-20, 2)
        return mean, log_std
    
    def sample(self, state):
        mean, log_std = self(state)
        std = log_std.exp()
        dist = torch.distributions.Normal(mean, std)
        # Reparameterization trick
        x_t = dist.rsample()
        action = torch.tanh(x_t)
        # Log prob correction for tanh
        log_prob = dist.log_prob(x_t) - torch.log(1 - action.pow(2) + 1e-6)
        log_prob = log_prob.sum(dim=-1, keepdim=True)
        return action, log_prob
```

### 11. **Conceptual**: "What is the credit assignment problem?"

```
CREDIT ASSIGNMENT:
- Which action caused the reward?
- Reward may come many steps later
- Need to attribute credit to correct actions

SOLUTIONS:
1. TD LEARNING (bootstrapping):
   - Propagate value estimates backward
   - Q(s,a) updated based on immediate reward + estimated future

2. MONTE CARLO (full returns):
   - Wait for episode end
   - Average returns for each (s,a) pair
   - High variance, no bias

3. λ-RETURNS / GAE:
   - Interpolate between TD and MC
   - λ controls bias-variance tradeoff

4. ELIGIBILITY TRACES:
   - e_t = γλ e_{t-1} + ∇_θ V(s_t)
   - Weight updates by trace
   - Forward view: λ-returns
   - Backward view: eligibility traces

5. HIERARCHICAL CREDIT:
   - Options framework: credit to options + intra-option
   - Feudal RL: manager sets goals, worker executes
```

### 12. **Code**: "Implement ε-greedy with decay schedule."

```python
class EpsilonGreedy:
    def __init__(self, start=1.0, end=0.01, decay=0.995, decay_type='exponential'):
        self.epsilon = start
        self.end = end
        self.decay = decay
        self.decay_type = decay_type
        self.step = 0
    
    def get_epsilon(self):
        return self.epsilon
    
    def step_decay(self):
        self.step += 1
        if self.decay_type == 'exponential':
            self.epsilon = max(self.epsilon * self.decay, self.end)
        elif self.decay_type == 'linear':
            self.epsilon = max(self.end, self.epsilon - (1.0 - self.end) / 1000000)
        elif self.decay_type == 'cosine':
            # Cosine annealing
            self.epsilon = self.end + 0.5 * (1.0 - self.end) * \
                (1 + np.cos(np.pi * self.step / 1000000))

# Adaptive ε based on uncertainty
class AdaptiveEpsilon:
    def __init__(self, q_network, threshold=0.5):
        self.q_net = q_network
        self.threshold = threshold
    
    def get_action(self, state):
        with torch.no_grad():
            q_values = self.q_net(torch.FloatTensor(state).unsqueeze(0))
            # Uncertainty: max Q - second max Q
            sorted_q = torch.sort(q_values, descending=True).values
            gap = sorted_q[0, 0] - sorted_q[0, 1]
            
            if gap < self.threshold:
                return random.randint(0, q_values.shape[1] - 1)
            return q_values.argmax().item()
```

---

## QUICK REFERENCE: REINFORCEMENT LEARNING CHEAT SHEET

### Algorithm Selection
| Scenario | Algorithm |
|----------|-----------|
| Discrete actions, tabular/small | Q-Learning, SARSA |
| Discrete actions, large state | DQN, Double DQN, Dueling DQN |
| Continuous actions | DDPG, TD3, SAC |
| High-dim continuous, stable | SAC, PPO |
| Sparse rewards | PPO + curiosity, HER, offline RL |
| Safety-critical | TRPO, PPO, Constrained RL |
| Offline data only | BCQ, CQL, IQL |
| Multi-agent | MADDPG, QMIX, MAPPO |

### Key Hyperparameters
| Algorithm | Key Hyperparameters |
|-----------|---------------------|
| DQN | lr=1e-3, γ=0.99, ε=1.0→0.01, buffer=1e5, batch=64, target_update=1000 |
| PPO | lr=3e-4, γ=0.99, clip=0.2, epochs=10, batch=64, λ=0.95 |
| SAC | lr=3e-4, γ=0.99, τ=0.005, α=auto, batch=256 |
| TD3 | lr=3e-4, γ=0.99, τ=0.005, policy_noise=0.2, noise_clip=0.5, delay=2 |

### Exploration Comparison
| Method | Pros | Cons |
|--------|------|------|
| ε-greedy | Simple | Non-adaptive, random |
| Boltzmann | Smooth, temp-based | Temp schedule needed |
| Entropy (PPO/SAC) | Built-in, adaptive | Hyperparameter sensitive |
| Noisy Nets | Parameter space | Extra params |
| Curiosity (ICM) | Works for sparse rewards | Extra computation |

### Common Pitfalls
| Pitfall | Solution |
|---------|----------|
| Diverging Q-values | Target network, gradient clipping, reward clipping |
| Overestimation | Double DQN, Clipped Double Q (TD3) |
| Premature convergence | Entropy bonus, ε-greedy, parameter noise |
| High variance | GAE, baselines, advantage normalization |
| Catastrophic forgetting | Experience replay, target networks |
| Sparse rewards | Reward shaping, curiosity, HER, demonstrations |