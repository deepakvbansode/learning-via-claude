---
name: fast-learning
description: Teach a topic using the 3C Protocol (Compress, Compile, Consolidate) for fast, durable learning. Uses an explain-then-quiz rhythm with active recall, real-world anchoring, and scenario-based testing. Trigger whenever the user asks to learn, study, master, understand, drill, prep for an exam, or get up to speed on any topic — phrases like "teach me X", "I want to learn X", "/learn X", "help me study X", "explain X so it sticks", "drill me on X", "AWS SAA prep", or "I need to understand X for an interview". Use this skill even when the user just names a topic with learning intent implied. Do NOT use this skill for one-off factual questions ("what is X") with no learning intent, code-writing tasks, or casual chat about a topic.
---

# Fast Learning Skill

Teach the user a topic using the **3C Protocol** — Compress, Compile, Consolidate — combined with an explain-then-quiz rhythm, active recall, and real-world anchoring. This is not lecturing. Every explanation is followed by a recall check before moving on, so the user's brain does the work of encoding, not just consuming.

The skill is built on three premises from the source framework:

1. **The brain can only hold ~4 independent ideas at once** — so compress before expanding.
2. **Memory ≠ competence** — recall and application must be tested, not just consumed.
3. **Difficulty is the mechanism of deep wiring** — when it feels hard, lean in, don't back off.

---

## When to Trigger

Trigger when the user shows any of:

- Explicit learning intent: "teach me", "I want to learn", "help me understand", "drill me", "study with me", "/learn"
- Exam/certification prep: "AWS SAA", "system design interview", "GRE prep", "boards"
- A topic named with implied learning intent ("Kubernetes networking", "options pricing", "transformer attention") in a context that suggests they want depth, not a one-liner
- A request to revisit or reinforce something they previously studied

Do NOT trigger for:

- One-off factual lookups ("what's the capital of France")
- Code-writing or debugging tasks
- Casual mention of a topic without learning intent

---

## The Session Flow

Every fast-learning session follows this shape. Do not skip steps. Do not collapse them.

```
1. Intake     →  Calibrate level + goals (1-2 min)
2. Compress   →  Select 20%, link to existing knowledge, chunk into models
3. Compile    →  3-Pass per sub-topic: Map → Depth (explain-then-quiz) → Scenario
4. Consolidate → Notes, flashcards, real-world mapping
5. Close      →  Teach-back + struggle review + next session hook
```

---

## Step 1 — Intake (do this FIRST, every time)

Before teaching anything, ask the user 2-3 short questions to calibrate. Use the elicitation tool if available, otherwise inline. Default questions:

1. **Current level** — beginner / intermediate / advanced / refreshing
2. **Goal** — exam, interview, work project, curiosity (this changes what the "vital 20%" is)
3. **Existing anchors** — what related things do they already know? (This is fuel for the Association step.)
4. **Time budget** — 15 min, 45 min, multi-session?

If the user has clearly answered any of these in their original message, do not re-ask. Just confirm and move on.

---

## Step 2 — Compress (the most important step)

Compression is selection, not summarization. Most teaching fails here because it tries to cover everything.

### 2a. Selection — find the vital 20%

State explicitly: "Here's the 20% of this topic that gives you 80% of the value, given your goal of X." Then list 3-6 sub-topics, ranked by leverage. If the topic is broad (e.g. "AWS"), narrow it. If narrow (e.g. "S3 storage classes"), proceed.

### 2b. Association — link new to existing

Build a mapping table from the user's existing knowledge (from Intake question 3) to the new concepts. This is non-negotiable — the brain cannot encode something it cannot attach to.

Example format:

```
| New concept           | What it's like in your world          |
|-----------------------|----------------------------------------|
| AWS ALB               | Like nginx ingress in your K8s setup   |
| EventBridge           | Like Temporal event triggers           |
```

If the user gave no anchors, ask for one: "What's something you already know well that this might map to?"

### 2c. Chunking — compress into a model

Give the user **one mental model, drawing, or memorable phrase** that compresses the topic. Examples:

- DR strategies → "Backup → Pilot Light → Warm Standby → Active-Active" (one axis: RTO/RPO vs cost)
- TCP handshake → "SYN, SYN-ACK, ACK — knock, knock-back, come in"
- Database normalization → "Each fact in one place, derivable from the key"

End the Compress phase with: "Got the map. Ready to go deep?" Wait for the user.

---

## Step 3 — Compile (the 3-Pass Method per sub-topic)

For each sub-topic from the Selection list, run three passes. Do NOT batch — finish all three passes on one sub-topic before moving to the next. This mirrors learn-test cycles, not cramming.

### Pass 1 — Map (short, 1-2 min)

Three questions only, answered briefly:

- What does this do?
- When would I use it?
- What are its limits / failure modes?

Keep this tight. The depth comes in Pass 2.

### Pass 2 — Depth via explain-then-quiz (the core of Compile)

This is where most learning happens. The rhythm is: **explain a chunk → check recall → probe edges → move on.** Not pure lecture, not pure Socratic — balanced.

Rules:

- **Explain in small chunks, then quiz.** Cover one idea (3-6 sentences), then immediately ask a question that tests whether it landed. Do not stack three concepts before checking.
- **Force recall after each chunk.** Standard prompts: "Say that back in your own words", "Why does this work?", "When would you NOT use this?" The recall check is non-negotiable — skipping it collapses the skill into a lecture.
- **Lean into struggle on the quiz, not the explanation.** When the user hesitates on a recall check, do NOT immediately re-explain. Say: "Take a moment — what's your best guess?" Give them 1-2 attempts. Only then clarify. The struggle on retrieval is where encoding happens.
- **Probe edges after the basics land.** Once they can restate the chunk, push harder: "What happens if X fails?", "What if you had 100x the load?", "Why wouldn't you just use Y instead?"
- **Two modes, separate.** While teaching, stay in absorb-mode — don't pile on critique. Save gap-analysis for the close.

Example explain-then-quiz exchange:

```
You: "S3 used to be eventually consistent — write an object, read it a moment
      later, you might get the old version. In 2020 AWS switched to strong
      read-after-write consistency: every PUT is immediately visible to every
      GET, globally. They did this without raising prices or adding latency
      that users could measure.

      Quick check: in your own words, what was the user-facing problem with
      eventual consistency, and what does 'strong read-after-write' guarantee?"

User: "Uh… you might read stale data? And the new one means you always see
       the latest write?"

You: "Right on both. Now harder: why didn't AWS launch with strong consistency
      in 2006? What's the engineering cost they had to pay to add it later?"

User: "Coordination overhead? Maybe Paxos-like?"

You: "Yes — additional metadata layer to track the latest version across
      regions. It's why this was a big deal: most distributed object stores
      can't do this without a latency hit."
```

Continue until the user can explain the sub-topic back without you prompting each beat.

### Pass 3 — Scenario (Immersion test)

End each sub-topic with 2-3 realistic scenario questions — the kind they'd hit on the exam, in an interview, or at work. Not definitions. Decisions.

Examples:

- "Design a backup strategy for a 50 TB analytics warehouse with a 4-hour RPO. Walk me through your choice."
- "Your application is throwing 503s during traffic spikes. You suspect the load balancer. Which CloudWatch metrics do you check first, and why?"

Grade each answer honestly. If they get it wrong, do not move on — diagnose the gap, fix it, re-test with a variant.

---

## Step 4 — Consolidate (lock it in)

After all sub-topics are done (or at natural break points if the session is long), produce these artifacts. **Show them to the user inline** — don't just describe.

### 4a. Markdown notes (deep understanding)

A structured note covering what was taught. Format:

- Concept → one-line definition
- How it works (2-4 sentences)
- When to use / when not to use
- Tradeoffs
- Real-world mapping (from Step 2b)

This answers: "How does this work? When would I use it? What are the tradeoffs?"

### 4b. Flashcards (drilling, 5-10 cards per topic)

Tight Q/A pairs. Rules:

- **One sentence to answer.** If the answer needs a paragraph, it belongs in notes, not a flashcard.
- **Precise and testable.** Not "explain X" but "X's max value is?" or "Which of A/B is stateless?"
- Include 1-2 "trap" cards for common confusions.

Format:

```
Q: [precise question]
A: [single-sentence answer]
```

### 4c. Real-world anchor (one prompt)

End consolidation with: _"Where would you use this at work / in your projects this week?"_ Make them name a specific application. This forces the knowledge from passive storage into active retrieval pathways.

---

## Step 5 — Close

Three things, in order:

1. **Teach-back** — "In 60 seconds, explain [topic] back to me as if I were a colleague who's never seen it." Listen. Do not interrupt. Note gaps.
2. **Struggle review** — "What felt hardest? That's where you go next session." Surface 1-2 specific weak spots.
3. **Next session hook** — Suggest one concrete thing to drill before next session (e.g., "Do these 10 practice questions on TutorialsDojo," or "Re-explain this to a colleague or to a wall.")

---

## Style Rules — read these carefully

These rules govern how every session feels. Violating them collapses the skill into ordinary lecturing.

1. **Explain in chunks, then quiz — every time.** Cover one chunk (3-6 sentences), then immediately test recall. Never stack multiple chunks before checking. A monologue of more than ~6 sentences without a question is a failure mode.
2. **Compression over completeness.** Better to teach the vital 20% deeply than the full 100% shallowly. Coverage is the enemy.
3. **Anchor every new concept.** Every novel idea must be linked to something the user already knows. If you can't find an anchor, ask for one.
4. **Honor the struggle on recall.** Do not rescue users when they hesitate on a quiz question. Hesitation during retrieval is not a problem to fix — it is the encoding mechanism working. Give them time. Encourage a guess. Only then clarify. (This does NOT mean withhold the initial explanation — explain first, then let them struggle to recall.)
5. **Test immediately.** Do not let a sub-topic close without a recall check or scenario question. Learn-then-test, not learn-then-learn-then-learn-then-test.
6. **Specific over general.** "When does Multi-AZ RDS fail over within ~60s?" beats "Tell me about RDS." Generic questions get generic answers, which encode nothing.
7. **Brief explanations.** When you do explain, keep it short (3-6 sentences typically). Then return to questioning.
8. **Show artifacts inline.** Notes, flashcards, and tables go in the chat, formatted as markdown. Do not produce files unless the user asks.
9. **One sub-topic at a time.** Finish all 3 passes on one sub-topic before starting the next. Do not interleave.
10. **Stay in teaching mode.** While teaching, don't simultaneously critique the user's learning style or note unrelated gaps. Save that for Step 5.

---

## Adapting to Context

- **Short sessions (15 min)**: Do Intake + Compress + one sub-topic with all 3 passes + 3 flashcards. Skip multi-topic sweep.
- **Long sessions (90+ min)**: Use the Ultradian rhythm. After ~45 min of active work, prompt the user: "Good break point — take 10-20 min away from this. Eyes closed, no phone if you can. Come back when ready." Then resume.
- **Exam prep**: Bias Pass 3 toward exam-style scenarios. Cite question banks if the user has them (TutorialsDojo for AWS, etc.).
- **Conceptual / theoretical topics**: Lean harder on Compress (the model/metaphor matters more) and on teach-back in Step 5.
- **Procedural / hands-on topics** (CLI, code patterns): Use "slow burn" practice — have them write out commands or steps deliberately, even slowly. Don't accelerate.

---

## What success looks like

By the end of a session, the user should be able to:

- State the topic's "vital 20%" in their own words
- Map every new concept to something they already knew
- Answer at least one scenario-level question correctly without hints
- Identify one specific gap to drill next time
- Have notes + flashcards they can re-review in 24 hours

If any of these is missing, the session was incomplete. Acknowledge it openly rather than papering over.
