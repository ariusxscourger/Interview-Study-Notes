# MOCK INTERVIEW PRACTICE
## Exam-Style Study Notes

---

## TOPIC: Mock Interview Practice Framework

### WHY? (Problem Statement)

**Practice is the only way to bridge knowledge → performance:**
- Knowledge ≠ Performance under pressure
- Real interviews have time constraints, nerves, follow-ups
- You discover gaps only when simulating
- Feedback loops are essential for improvement

**What Mock Practice Solves:**
| Problem | Solution |
|---------|----------|
| Blank screen syndrome | Rehearsed templates + muscle memory |
| Time management | Timed practice sessions |
| Communication | Articulate thinking out loud |
| Nerves | Exposure therapy |
| Gap identification | Targeted review |

---

### HOW? (Mock Interview Framework)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MOCK INTERVIEW FRAMEWORK (Weekly)                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  MIX: 2 Coding + 1 System Design + 1 Behavioral + 1 Negotiation/week      │
│                                                                             │
│  SESSION STRUCTURE (60-90 min):                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  1. SETUP (5 min)                                                   │   │
│  │     • Choose problem (LeetCode tagged, or System Design topic)     │   │
│  │     • Set timer (45 min coding, 45 min system design)              │   │
│  │     • Open blank editor/whiteboard                                 │   │
│  │     • Recording ON (audio + screen)                                │   │
│  │                                                                     │   │
│  │  2. EXECUTION (45-60 min)                                          │   │
│  │     • NO HINTS, NO LOOKUP                                          │   │
│  │     • Think OUT LOUD                                               │   │
│  │     • Code/design as in real interview                             │   │
│  │     • Handle "stuck" moments gracefully                            │   │
│  │                                                                     │   │
│  │  3. SELF-REVIEW (15 min)                                           │   │
│  │     • Watch recording at 1.5x                                      │   │
│  │     • Score using rubric                                           │   │
│  │     • Note: stuck points, bugs, communication gaps                 │   │
│  │                                                                     │   │
│  │  4. PEER REVIEW (Optional, 30 min)                                 │   │
│  │     • Swap recordings with study buddy                             │   │
│  │     • Give structured feedback                                     │   │
│  │     • Discuss alternative approaches                               │   │
│  │                                                                     │   │
│  │  5. TARGETED REVIEW (30 min)                                       │   │
│  │     • Review weak areas identified                                 │   │
│  │     • Update notes/anki                                            │   │
│  │     • Add to "stuck patterns" doc                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### WHAT? (Rubrics & Templates)

#### **1. CODING RUBRIC (45 min)**

| Dimension | Weight | Excellent (4) | Good (3) | Needs Work (2) | Poor (1) |
|-----------|--------|---------------|----------|----------------|----------|
| **Problem Understanding** | 15% | Clarifies all constraints, edge cases | Asks key questions | Misses some constraints | Jumps in without clarifying |
| **Approach & Algorithm** | 25% | Optimal O(n) / O(n log n), explains trade-offs | Works but suboptimal | Brute force only | No clear approach |
| **Code Quality** | 20% | Clean, modular, handles edge cases | Functional, minor issues | Bugs, messy, hard to read | Doesn't compile/runs |
| **Complexity Analysis** | 15% | Accurate T/S, explains why | Mostly accurate | Vague or wrong | Missing |
| **Testing & Debugging** | 15% | Tests edge cases, walks through | Basic testing | Minimal testing | No testing |
| **Communication** | 10% | Clear narration, thinks aloud | Mostly clear | Quiet, hard to follow | Silent |

**PASS THRESHOLD: ≥ 3.0 average**

#### **2. SYSTEM DESIGN RUBRIC (45 min)**

| Dimension | Weight | Excellent (4) | Good (3) | Needs Work (2) | Poor (1) |
|-----------|--------|---------------|----------|----------------|----------|
| **Requirements Clarification** | 15% | Explicit functional/non-functional, scale | Covers basics | Misses key NFRs | Jumps to solution |
| **API Design** | 10% | RESTful, versioned, paginated, errors | Basic CRUD | Missing pagination/errors | No API design |
| **Data Model** | 15% | ER diagram, SQL/NoSQL choice, indexes | Basic schema | Missing relationships | No data model |
| **Architecture** | 20% | Services, data flow, protocols | Basic components | Missing key components | No architecture |
| **Scaling & Deep Dives** | 25% | Caching, sharding, caching, queues | Basic scaling | Misses key bottlenecks | No scaling |
| **Trade-offs** | 10% | Explicit CAP/consistency/latency | Mentions some | Few trade-offs | None |
| **Reliability** | 10% | Retries, circuit breaker, observability | Basic retries | Minimal | None |

**PASS THRESHOLD: ≥ 3.0 average**

#### **3. BEHAVIORAL RUBRIC (STAR)**

| Element | Excellent (4) | Good (3) | Needs Work (2) | Poor (1) |
|---------|---------------|----------|----------------|----------|
| **Situation** | Rich context, relevant | Clear context | Vague context | Missing |
| **Task** | Clear ownership | Clear role | Vague role | Missing |
| **Action** | 4-6 specific "I" actions | 2-3 actions | Vague "we" | Generic |
| **Result** | 2-3 quantified metrics | 1 metric | Qualitative only | Missing |
| **Learning** | Insightful, applied | Clear learning | Generic | None |

---

### PRACTICE SCHEDULE (8-Week Plan)

```
WEEK 1-2: FOUNDATIONS
├── Day 1-3: 3× Coding (Easy/Medium) - Sliding Window, Two Pointers, Hash Map
├── Day 4: 1× System Design - URL Shortener
├── Day 5: 1× Behavioral - Conflict/Leadership
├── Day 6: 1× Coding - Trees/Graphs (BFS/DFS)
├── Day 7: Review weak areas, Anki

WEEK 3-4: CORE PATTERNS
├── 2× Coding/week - Medium (DP, Binary Search, Heaps)
├── 1× System Design/week - Rate Limiter, Notification System
├── 1× Behavioral/week - Mistake, Disagreement, Learning
├── Weekend: Full mock (Coding + System Design)

WEEK 5-6: ADVANCED
├── 2× Coding/week - Hard (DP optimization, Graph algorithms)
├── 1× System Design/week - Twitter, WhatsApp, Uber
├── 1× Negotiation practice/week
├── Weekend: Full loop mock (Coding + System + Behavioral)

WEEK 7-8: POLISH
├── Company-specific mocks (target company style)
├── Timed full loops (2hr blocks)
├── Weak area blitz (identified from reviews)
├── Final prep: rest, mental prep, logistics
```

---

### TOOLS & SETUP

```
RECOMMENDED SETUP:
─────────────────────────────────────────────────────────────────────────────
□ Timer: Visible countdown (phone/website)
□ Editor: VS Code (no Copilot), or LeetCode/CodeSignal
□ Whiteboard: Excalidraw, Miro, or physical
□ Recording: OBS (screen + mic) or Loom
□ Rubric: Printed or digital checklist
□ Problems: LeetCode tagged, System Design Primer topics
□ Partner: Study buddy / Discord / Pramp / Interviewing.io

ENVIRONMENT:
□ Quiet room, good lighting
□ Camera at eye level
□ Water, notepad, pen
□ Phone on DND
```

---

### SELF-REVIEW TEMPLATE

```
POST-MOCK REVIEW (Fill immediately after)

PROBLEM: ________________________________________________
TIME TAKEN: ______ min | TARGET: ______ min

SCORES (1-4):
□ Problem Understanding: ___
□ Approach/Algorithm: ___
□ Code Quality: ___
□ Complexity Analysis: ___
□ Testing/Debugging: ___
□ Communication: ___
AVERAGE: ____

WHAT WENT WELL:
1. _______________________________________________________
2. _______________________________________________________

STUCK POINTS (Where I got stuck, time lost):
1. _______________________________________________________
   Resolution: ____________________________________________
2. _______________________________________________________
   Resolution: ____________________________________________

BUGS INTRODUCED:
1. _______________________________________________________
   Caught by: ☐ Self-test ☐ Walkthrough ☐ Missed entirely

COMMUNICATION GAPS:
□ Silent too long (>30s)    □ Didn't explain approach first
□ Didn't clarify constraints □ Jumped to code
□ Didn't explain trade-offs □ Monotone/quiet

ACTION ITEMS FOR NEXT SESSION:
1. _______________________________________________________
2. _______________________________________________________
3. _______________________________________________________

PATTERN TO REVIEW: ________________________________________
ANKI CARDS TO ADD: ________________________________________
```

---

### PEER FEEDBACK TEMPLATE

```
PEER REVIEW FOR: _________________________________________
REVIEWER: ________________________________________________

CODING / SYSTEM DESIGN (Circle one)

STRENGTHS:
1. _______________________________________________________
2. _______________________________________________________
3. _______________________________________________________

AREAS FOR IMPROVEMENT:
1. _______________________________________________________
   Specific example: ______________________________________
2. _______________________________________________________
   Specific example: ______________________________________

COMMUNICATION:
□ Clear narration    □ Good pace    □ Explains trade-offs
□ Asks clarifying    □ Tests aloud   □ Handles stuck gracefully

SPECIFIC FEEDBACK:
_____________________________________________________________________________
_____________________________________________________________________________

RATING: ____/4 overall
WOULD HIRE: ☐ Yes  ☐ Maybe  ☐ No

ADDITIONAL NOTES:
_____________________________________________________________________________
```

---

### PROBLEM SETS BY WEEK

#### **WEEK 1-2: FUNDAMENTALS (Easy/Medium)**
```
CODING:
□ Two Sum (Hash Map)
□ Best Time to Buy/Sell Stock (Sliding Window)
□ Valid Parentheses (Stack)
□ Merge Two Sorted Lists (Two Pointers)
□ Binary Tree Level Order (BFS)
□ Valid Anagram (Hash Map)
□ Maximum Subarray (Kadane)
□ Linked List Cycle (Fast/Slow)

SYSTEM DESIGN:
□ URL Shortener (bit.ly)
□ Pastebin

BEHAVIORAL:
□ Tell me about yourself
□ Why this company?
□ Conflict with teammate
□ Mistake you made
```

#### **WEEK 3-4: CORE PATTERNS (Medium)**
```
CODING:
□ Longest Substring Without Repeating (Sliding Window)
□ 3Sum (Two Pointers + Sort)
□ Container With Most Water (Two Pointers)
□ Top K Frequent Elements (Heap/Bucket)
□ Course Schedule (Topological Sort)
□ Number of Islands (BFS/DFS + Union Find)
□ LRU Cache (HashMap + Doubly Linked List)
□ Serialize/Deserialize Binary Tree

SYSTEM DESIGN:
□ Rate Limiter (Token Bucket, Sliding Window)
□ Notification Service (Email/SMS/Push)
□ Distributed Cache (Redis Cluster)

BEHAVIORAL:
□ Disagreement with manager/peer
□ Project failed / missed deadline
□ Mentored junior engineer
□ Learned new tech quickly
```

#### **WEEK 5-6: ADVANCED (Hard + System Design)**
```
CODING:
□ Median of Two Sorted Arrays (Binary Search)
□ Trapping Rain Water (Two Pointers/Stack)
□ Minimum Window Substring (Sliding Window)
□ Word Ladder (BFS + Graph)
□ Alien Dictionary (Topological Sort)
□ Burst Balloons (DP Interval)
□ Regular Expression Matching (DP)
□ Split Array Largest Sum (DP + Binary Search)

SYSTEM DESIGN:
□ Twitter Timeline (Fan-out, Push vs Pull)
□ Instagram Feed (Ranking, pagination)
□ WhatsApp/Chat (Real-time, delivery guarantees)
□ YouTube/Netflix (Video encoding, CDN, adaptive bitrate)
□ Uber/Lyft Matching (Geo-indexing, real-time)
□ Dropbox/Google Drive (Sync, conflict resolution)

BEHAVIORAL:
□ Technical leadership / architecture decision
□ Influencing without authority
□ Growing team / hiring
□ Technical debt paydown
```

#### **WEEK 7-8: COMPANY-SPECIFIC + POLISH**
```
TARGET COMPANY PREP:
□ Google: System design heavy, distributed systems, scale
□ Meta: Product sense, move fast, coding + system design
□ Amazon: LP examples, system design (scale), bar raiser
□ Apple: Deep technical, collaboration, secrecy culture
□ Netflix: Freedom/responsibility, system design scale
□ Startups: Full stack, ownership, speed, ambiguity

FINAL POLISH:
□ 2 full 2-hour mock loops (Coding + System + Behavioral)
□ Weak area blitz (review all "stuck patterns" from log)
□ Final Anki review (high-yield cards only)
□ Logistics: outfit, location, timezone, backup internet
□ Mental prep: sleep, nutrition, visualization
```

---

### ANKI FLASHCARDS (Mock-Specific)

| Front | Back |
|-------|------|
| Mock Interview Frequency | 2-3 per week, mix types. Full loop weekly in final 2 weeks. |
| Coding Time Box | 45 min: 5min understand, 10min plan, 25min code, 5min test |
| System Design Time Box | 5min req, 5min API, 10min data, 10min arch, 15min deep, 5min tradeoffs |
| When Stuck in Coding | 1. Say "I'm stuck here" 2. Explain what you've tried 3. Ask for hint 4. Try simpler version |
| System Design Stuck | "I'm considering X vs Y for this component. Trade-off is..." |
| Behavioral Time Box | 2-3 min per answer. STAR: 15% S, 15% T, 60% A, 20% R |
| "I don't know" in Interview | "I haven't encountered X, but my approach would be..." |
| Recording Review | Watch at 1.5x. Note: silence >30s, unclear explanation, missed edge cases |
| Stuck Pattern Log | Track: pattern, time lost, trigger, resolution. Review weekly. |
| Peer Review Value | Different perspective, catches blind spots, practice giving feedback |
| Mock Frequency Ramp | Weeks 1-4: 2-3/week. Weeks 5-6: 3-4/week. Weeks 7-8: 4-5/week + full loops. |

---

### PRACTICE RESOURCES

```
CODING:
□ LeetCode (Tagged: Sliding Window, Two Pointers, etc.)
□ NeetCode 150 / Blind 75
□ CodeSignal / HackerRank (timed)
□ AlgoExpert (video explanations)

SYSTEM DESIGN:
□ System Design Primer (GitHub)
□ Designing Data-Intensive Applications (DDIA) - key chapters
□ Alex Xu System Design Interview (Vol 1, 2)
□ ByteByteGo newsletter
□ HLD/LLD practice: Excalidraw diagrams

BEHAVIORAL:
□ STAR stories written out (8-10 stories)
□ Company values mapped to stories
□ Pramp / Interviewing.io / Peer group
□ Recording practice: Loom / OBS

MOCK PLATFORMS:
□ Pramp (free, peer)
□ Interviewing.io (anonymous, senior engineers)
□ LeetCode Mock Interview
□ SystemDesign.one (system design specific)
□ Peer group (Discord/Slack study group)

TRACKING:
□ Spreadsheet: Date, Type, Problem, Score, Stuck Points, Action Items
□ Notion/Obsidian: Pattern log, stuck patterns, solutions
□ Anki: Daily review (15 min)
```

---

### FINAL WEEK CHECKLIST

```
□ 3 full mock loops completed (scored ≥ 3.0)
□ All 10 STAR stories polished, recorded, timed
□ System design: Can draw 5 architectures from memory
□ Coding: Can solve Medium in 25min, Hard in 40min
□ Negotiation: Scripts memorized, leverage built
□ Logistics: Outfit, location, timezone, backup internet, ID
□ Rest: 7-8 hrs sleep 3 nights before
□ Mental: Visualization exercise done
□ Gratitude: Thank peers/mentors who helped
```

---

> **Remember**: The interview is a collaboration, not an interrogation. They want you to succeed. You've prepared. Trust your preparation.

> **Final thought**: "I don't rise to the level of my expectations; I fall to the level of my training." — Train hard, interview easy.