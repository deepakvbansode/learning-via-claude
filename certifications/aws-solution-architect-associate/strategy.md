# AWS Solutions Architect Associate (SAA-C03) — Study Strategy

> Target: Pass in first attempt within 2 months
> Daily commitment: 2 hours weekday, 4 hours weekend

---

## 1. Exam Overview

| Detail | Info |
|---|---|
| Exam code | SAA-C03 |
| Duration | 130 minutes |
| Questions | 65 (multiple choice + multiple response) |
| Passing score | 720/1000 |
| Format | Design decisions, scenario-based — not memorization |

### Four Exam Domains

| Domain | Topic | Weight |
|---|---|---|
| 1 | Design Secure Architectures | 30% |
| 2 | Design Resilient Architectures | 26% |
| 3 | Design High-Performing Architectures | 24% |
| 4 | Design Cost-Optimized Architectures | 20% |

---

## 2. Know Your Starting Point — Priority Matrix

Before studying, map all topics on this matrix. Do this once after reading the official exam guide.

|  | **High Priority** (heavily tested) | **Low Priority** (lightly tested) |
|---|---|---|
| **Known** | Quick review + practice Qs only | Skip or skim |
| **Unknown** | Deep study — full 3-pass method | Light coverage |

**Your Known advantages (leverage these):**
- VPC/networking → CDN/Fastly/Cloudflare background
- EKS/ECS → K8s expertise
- CodePipeline/CodeBuild → GitHub CI/CD background
- EventBridge/Step Functions → Temporal workflow experience
- CloudFront → Fastly CDN mental model
- PrivateLink/VPC endpoints → SASE zero-trust background

**Your Unknown high-priority areas (invest most time):**
- IAM (policies, SCP, permission boundaries, cross-account)
- S3 (storage classes, lifecycle, replication, encryption)
- RDS/Aurora/DynamoDB (when to use which)
- DR strategies (RTO/RPO tradeoffs)
- SQS/SNS/EventBridge (decoupling patterns)
- Cost optimization (pricing models, reserved vs spot vs savings plans)

---

## 3. Resources

| Resource | Purpose | Priority |
|---|---|---|
| Stephane Maarek — SAA-C03 (Udemy) | Primary video course | Must |
| TutorialsDojo (Jon Bonso) | Best practice exams | Must |
| AWS Skill Builder | Hands-on labs (free) | High |
| AWS Well-Architected Framework | 6 pillars — skim + know | High |
| Official AWS Exam Guide | Topic list + domain weights | Start here |

> Watch Stephane's course at 1.5x speed. Use it to structure topics, not as primary learning method.

---

## 4. Learning System (Tools)

| Tool | Role | Do NOT use for |
|---|---|---|
| **Notion** | Roadmap, topic tracker, progress %, mock exam score log | Long-form notes |
| **GitHub repo** | Deep structured notes per topic (markdown) | Progress tracking |
| **Anki** | Flashcard drilling — 15 min/day | Notes or planning |
| **Claude /learn** | Socratic drilling per topic (45 min sessions) | Note storage |

### Notion Setup
Use the [CompTIA Security+ Prep Lite template](https://www.notion.com/templates/comptia-security-v7-exam-study-plan-lite) (free) — replace CompTIA domains with the 4 AWS SAA domains above. Track:
- Topic status: Not Started / In Progress / Done / Needs Review
- Confidence: Low / Medium / High
- Practice Qs done per topic
- Mock exam scores over time

### Anki Cards — What to Card-ify
Only add facts that are: specific, easy to forget, commonly tested.

**Good card:**
```
Q: S3 max object size via single PUT?
A: 5 GB (use multipart for >100 MB)

Q: RDS Multi-AZ vs Read Replica — what does Multi-AZ solve?
A: HA/failover only — NOT read scaling
```

**Bad card** (too broad → goes in GitHub notes):
```
Q: How does VPC work?
```

Rule: 1 sentence to answer → Anki. More than 1 sentence → GitHub notes.

---

## 5. Learning Method — The 3C Protocol

### C1 — Compress (before studying a topic)
- **Select:** Identify the vital 20% of the topic that yields 80% of exam value
- **Associate:** Connect new service to something you already know (see Priority Matrix above)
- **Chunk:** Reduce to a simple model — one diagram, one metaphor, one comparison table

### C2 — Compile (during study)
- Follow the **ultradian cycle:** 90 min focused work → 20 min rest
- Use **learn-test cycles:** learn a topic → test immediately → fix gaps → repeat
- After each `/learn` session: close notes, explain the service aloud from scratch (teach-to-learn)

### C3 — Consolidate (after study)
- **Micro breaks:** 10-20 sec eyes-closed pause every 20-25 min (brain replays at 10-20x speed)
- **Meso rest:** 20 min NSDR (lie still, eyes closed, no phone) between 90-min blocks
- **Sleep:** Non-negotiable. Brain consolidates and reverses-replays learned material during sleep

---

## 6. Per-Topic Study Method — 3-Pass Method

Run this for every new topic:

**Pass 1 — Map (10 min)**
- What does this service do?
- When would I use it? When would I NOT use it?
- What are its key limits?

**Pass 2 — Depth via `/learn` (25-30 min)**
- Socratic drilling with Claude until you can explain it without notes
- Then: explain it aloud to yourself (teach-to-learn)
- Write structured notes → push to GitHub

**Pass 3 — Scenario (15 min)**
- Do 5-10 TutorialsDojo practice Qs on this specific topic
- Map every wrong answer to a specific gap
- Fix gap immediately — don't defer

---

## 7. Time Management

### Daily Schedule (Weekdays — 2 hrs)

| Time | Block | Activity |
|---|---|---|
| Early morning | 90 min | New topic: Pass 1 + Pass 2 (/learn session) + Pass 3 (practice Qs) |
| Evening | 30 min | Anki review + write/review GitHub notes |

### Daily Schedule (Weekends — 4 hrs)

| Time | Block | Activity |
|---|---|---|
| Early morning | 3 hrs | Hard/new topics — full 3-pass method, 2x 90-min blocks with 20-min NSDR break |
| Evening | 1 hr | Revision, mock exam Qs, reflect on week's weak areas |

### Daily Micro-Routine

```
Morning (90 min block):
  15 min  →  Anki review (yesterday's cards)
  45 min  →  New topic via /learn (Pass 1 + Pass 2)
  20 min  →  Practice Qs on today's topic (Pass 3)
  10 min  →  Add 5-10 Anki cards + push notes to GitHub

Evening (30 min):
  15 min  →  Anki review
  15 min  →  Skim weak topic or reread one GitHub note
```

---

## 8. 8-Week Roadmap

### Month 1 — Foundation

| Week | Topics | Your Leverage |
|---|---|---|
| 1 | IAM: policies, roles, STS, SCP, permission boundaries, cross-account | New — invest full time |
| 2 | VPC: subnets, NACLs, SGs, peering, endpoints, PrivateLink, Route53, CloudFront | CDN/networking → fast, focus depth on routing |
| 3 | EC2, ASG, ELB (ALB/NLB/GLB), placement groups | K8s ingress/load balancing mental model |
| 4 | S3: storage classes, lifecycle, replication, encryption, access control, Glacier | New — invest full time |

### Month 2 — Advanced + Exam Hardening

| Week | Topics | Your Leverage |
|---|---|---|
| 5 | RDS, Aurora, DynamoDB, ElastiCache — when to use which | New — invest full time |
| 6 | SQS, SNS, EventBridge, Step Functions, Kinesis — decoupling patterns | Temporal workflows mental model |
| 7 | ECS, EKS, Lambda, API Gateway, SAM — serverless + containers | K8s background → EKS fast |
| 8 | Full practice mocks + weak area blitz | No new topics — only drilling |

> Week 8 = exam simulation week. No new learning. Only timed mocks + weak area repair.

---

## 9. Mock Exam Strategy

- Start full mocks from **Week 6** (not earlier)
- Do 65-question timed mocks (130 min) — simulate real exam conditions
- **Review every answer** — wrong AND right — understand why each option is correct/incorrect
- Track scores in Notion score log
- Target: **80%+ consistently** before booking the real exam
- If stuck below 75%: don't book — find and fix weak domains first

### Elimination Technique for Wrong Answers
1. Eliminate answers that violate the scenario constraints (cost, availability, latency)
2. Eliminate answers with services that don't fit the scale/requirement
3. Between 2 remaining: pick the one that is simpler and more AWS-native
4. Watch for distractors: answers that are technically correct but not the *best* fit

---

## 10. Exam Pattern Recognition

The exam tests **design decisions**, not memorization. Learn to ask: *"Given these constraints, which architecture?"*

| Scenario keyword | Answer pattern |
|---|---|
| Cost optimization | Spot instances, Reserved, Savings Plans, S3 Intelligent-Tiering |
| High availability | Multi-AZ, Cross-zone LB, Route53 failover |
| Decoupling | SQS / SNS / EventBridge — never direct synchronous calls |
| Stateless app | Push session state to ElastiCache or DynamoDB |
| Security | Least privilege IAM, VPC endpoints, PrivateLink, KMS |
| Disaster recovery | Backup → Pilot Light → Warm Standby → Active-Active (RTO/RPO tradeoff) |
| Millions of requests | Lambda + API Gateway or SQS buffering |
| Relational + scale | Aurora (not RDS) |
| NoSQL + single-digit ms | DynamoDB |
| Shared file system | EFS (not EBS — EBS is single instance) |

---

## 11. Notes Strategy

### What goes where

| Content type | Location |
|---|---|
| Service overview, tradeoffs, architecture diagrams | GitHub repo (markdown per topic) |
| Specific facts, limits, comparisons | Anki cards |
| Topic status, progress %, mock scores | Notion |
| Session drilling | Claude /learn |

### GitHub Note Structure (per topic)
```
# [Service Name]

## What it does
## When to use it / When NOT to use it
## Key limits and gotchas
## Comparison with similar services
## Real-world mapping (how it relates to your existing experience)
## Common exam patterns
```

---

## 12. Revision Plan

| When | What |
|---|---|
| End of each week | Review all Anki weak cards from the week |
| Week 5 | Re-read Month 1 GitHub notes, re-test weak areas |
| Week 7 | Full pass through all Notion topics marked Low/Medium confidence |
| Week 8 | Only weak domains — no new material |
| Day before exam | Light Anki review only — no heavy studying |

---

## 13. Exam Readiness Checklist

Before booking the exam:
- [ ] 80%+ on at least 3 full TutorialsDojo mocks
- [ ] All 4 domains at Medium or High confidence in Notion
- [ ] Can explain all 6 Well-Architected pillars from memory
- [ ] Know all DR patterns cold (RTO/RPO for each)
- [ ] Know pricing models: on-demand, reserved, spot, savings plans
- [ ] Know IAM: policies, SCP, permission boundaries, cross-account roles
- [ ] Know when NOT to use each major service (equally tested)
- [ ] Anki deck: <10% cards in Hard bucket

---

## 14. Mindset

- Difficulty is not failure — it is the mechanism of deep learning (Carnegie Mellon study: harder = 2x retention)
- Don't critique while learning — keep absorb mode and review mode separate
- Your only competition is your previous self
- One bad practice score means a gap to fix, not a reason to panic
