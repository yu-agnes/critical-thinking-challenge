**Author:** Yu Ye  
**Date:** February 2026

# Challenge Myself: Design Iteration Log

## AI-Powered Critical Thinking Training Prototype

**Project:** Using AI to teach students critical thinking skills

**Core Problem:** When AI generates responses, students often accept them passively. This prototype encourages students to think critically about AI outputs.

**Feature:** "Challenge Myself" — after receiving an AI response, students click a button to see the AI critique its own answer, revealing flaws, biases, and missing perspectives.

---

## Table of Contents

1. [Technical Architecture](#1-technical-architecture)
2. [API Interaction Design](#2-api-interaction-design)
3. [Iteration 1: Gentle Critique (V1)](#3-iteration-1-gentle-critique-v1)
4. [Evaluating V1 Output](#4-evaluating-v1-output)
5. [Iteration 2: Adversarial Critique (V2)](#5-iteration-2-adversarial-critique-v2)
6. [Evaluating V2 Output](#6-evaluating-v2-output)
7. [Iteration 3: Mentor Critique (V3)](#7-iteration-3-mentor-critique-v3)
8. [Evaluating V3 Output](#8-evaluating-v3-output)
9. [Three-Version Comparison](#9-three-version-comparison)
10. [Conclusions](#10-conclusions)

---

## 1. Technical Architecture

### Overview: 100% Client-Side

The entire prototype is a single HTML file (~590 lines) that runs entirely in the browser. No backend, no build step, no dependencies.

```
Browser (index.html)
  ├── HTML  — structure
  ├── CSS   — styling
  └── JS    — logic
        │
        ├── sendMessage()      → handles chat flow
        ├── handleChallenge()  → handles critique flow
        └── callOpenAI()       → shared API caller
                │
                ▼
        OpenAI REST API (direct fetch)
```

**Why this choice:** Zero setup friction. No Node.js, no `npm install`, no build step. The file can be opened by double-clicking it. 

**No frameworks used** — no React, no LangChain, no libraries. Just vanilla HTML/CSS/JS and the browser's built-in `fetch()` API. LangChain would add complexity without benefit here since we're making straightforward API calls with static prompts.

---

## 2. API Interaction Design

### API Caller

The OpenAI API is called via a single shared function:

```js
async function callOpenAI(messages) {
  const res = await fetch('https://api.openai.com/v1/chat/completions', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${key}`
    },
    body: JSON.stringify({
      model,          // user-selectable: gpt-4o-mini, gpt-4o, gpt-4-turbo
      messages,       // array of {role, content} objects
      temperature: 0.7,
      max_tokens: 1024
    })
  });
}
```

### Two Separate API Calls Per Cycle

Each question-challenge cycle makes two independent API calls:

| Call | Trigger | System Prompt | Conversation Context |
|------|---------|---------------|---------------------|
| **Chat response** | User sends a question | Generic helpful assistant | Full conversation history (multi-turn) |
| **Critique** | User clicks "Challenge Myself" | Critical thinking persona | Fresh context — only the original question + response |

**Key design decision:** The critique call starts a fresh conversation. It does not include the chat history. This prevents the model from being "loyal" to its prior reasoning. The critique acts as an independent reviewer, not the same assistant backtracking.

### Other Technical Decisions

- **`temperature: 0.7`** — moderate temperature balances coherent critique with enough variation to surface different issues across runs
- **API key in `sessionStorage`** — cleared when the browser tab closes, avoiding persisting credentials on disk
- **Button disappears after use** — prevents generating multiple critiques of the same response, keeping the flow linear

---

## 3. Iteration 1: Gentle Critique (V1)

**File:** `index-v1.html`
**Header:** "Challenge Myself — Critical Thinking with AI"

### System Prompt

```
You are a critical thinking coach. The user asked a question and an AI gave
a response. Your job is to critique that AI response to help the student
think more critically.

Analyze the response for:
- Potential inaccuracies or oversimplifications
- Missing perspectives or alternative viewpoints
- Hidden assumptions or biases
- Weak arguments or logical gaps
- Important nuances that were glossed over
- What questions a critical thinker should ask next

Be specific and constructive. Reference actual claims from the response.
Your goal is to teach the student that AI answers should be examined, not
blindly accepted.

Keep your critique focused and concise (roughly 150-250 words). Use plain
language.
```

### User Message

```
Original question: "${originalQuestion}"

AI response to critique:
"${aiResponse}"

Please provide a critical analysis of this AI response.
```

### Design Rationale

- Role framed as "coach" — supportive, constructive
- Bullet list of 6 analysis areas gives breadth but no specific methodology
- Tone guidance: "specific and constructive"
- No required output structure

---

## 4. Evaluating V1 Output

### Test Question

> "Do you think death penalty is a good thing or not?"

### AI Response (Summary)

Standard balanced overview: 3 arguments for (deterrence, retribution, closure) and 4-5 arguments against (wrongful conviction, lack of deterrence evidence, moral concerns, discrimination). Concludes with "depends on individual values."

### V1 Critique Output

The critique identified five issues: oversimplification of deterrence, missing restorative justice perspective, hidden assumptions about shared values, logical gap in the "preventing reoffending" argument (life imprisonment also achieves this), and insufficiently explored ethical concerns. Ended with two generic follow-up questions.

### Evaluation: Grade B-

**Strengths:**
- Correctly identifies oversimplification of the deterrence argument
- Brings up restorative justice as a missing perspective
- Notes that life imprisonment also prevents reoffending (a genuine logical gap)

**Weaknesses:**
- **Hedged language throughout.** "The AI should acknowledge," "it would be helpful to," "could enrich the discussion." Reads like a peer review suggesting revisions, not a challenge.
- **Doesn't quote specific claims.** References arguments in general terms without isolating the actual problematic phrasing.
- **Generic follow-up questions.** "What evidence supports the claims?" is too broad to act on.
- **Doesn't catch the asymmetric framing** (3 arguments for, 4-5 against — the response subtly leans one way while claiming neutrality).
- **No overall verdict.** The student finishes reading and thinks "okay, so the answer was decent but incomplete" — not a strong critical thinking takeaway.

### Key Insight

The prompt tells the model *what to look for* but not *how to do it*. This produces vague, suggestion-oriented output rather than concrete, demonstrative critique.

---

## 5. Iteration 2: Adversarial Critique (V2)

**File:** `index-v2.html`
**Header:** "Challenge Myself — v2 — Adversarial Critique"

### System Prompt

```
You are a tough, no-nonsense critical thinking examiner. An AI produced a
response to a student's question. Your job is to tear it apart — find every
weakness, unsupported claim, and blind spot.

Rules for your critique:

1. IDENTIFY UNSUPPORTED CLAIMS: Quote specific statements from the response
   and say "This is stated without evidence." Demand sources, data, or
   reasoning for every factual claim.

2. EXPOSE HIDDEN BIAS: If the response claims to be "balanced" or "neutral,"
   examine whether it actually is. Count arguments on each side. Flag
   asymmetric framing, weasel words ("some believe"), or selective emphasis.

3. FIND THE WEAKEST ARGUMENT: Identify the single least defensible claim in
   the response and explain exactly why it fails — logically, empirically,
   or both.

4. NAME WHAT'S MISSING: Don't just say "there are other perspectives." State
   specifically which perspectives, data, or contexts are absent and why they
   matter.

5. CHALLENGE THE FRAMING: Question whether the response even approaches the
   topic in the right way. Are there better frameworks for thinking about
   this issue?

6. END WITH A VERDICT: Give a blunt 1-2 sentence overall assessment of the
   response quality. Then pose 2-3 specific follow-up questions the student
   should investigate.

Be direct and confident. Use plain language. Do not soften your critique
with phrases like "could be improved" or "might consider." Say what IS
wrong, not what COULD be better.

Keep your critique to 200-300 words.
```

### User Message

```
Original question: "${originalQuestion}"

AI response to critique:
"${aiResponse}"

Tear this response apart. What's wrong with it?
```

### Design Changes from V1

| Change | V1 | V2 | Why |
|--------|----|----|-----|
| Role | "coach" | "tough, no-nonsense examiner" | Stronger persona produces bolder critique |
| Structure | Vague bullet list | 6 numbered rules with operational instructions | Tells model *how* to critique, not just *what* to look for |
| Specificity | "Find biases" | "Count arguments on each side, flag weasel words" | Operational instructions produce concrete output |
| Tone | "Specific and constructive" | "Say what IS wrong, not what COULD be better" | Eliminates hedging language |
| Output format | Unstructured | Mandates verdict + follow-up questions | Ensures actionable conclusion |
| User message | "Please provide a critical analysis" | "Tear this response apart" | Primes adversarial stance |

---

## 6. Evaluating V2 Output

### V2 Critique Output

The critique: (1) flagged "studies on its effectiveness are inconclusive" as lacking evidence — "Which studies? What data?"; (2) counted argument asymmetry and identified weasel words like "some research suggests"; (3) pinpointed the cost argument as weakest — legal expense is a feature of safeguards, not a bug; (4) named restorative justice and psychological impact as missing; (5) briefly mentioned deeper philosophical questions; (6) delivered a blunt verdict: "superficial and unbalanced, lacking depth and critical engagement."

### Evaluation: Grade A-

**Strengths:**
- **Quotes specific claims and demands evidence.** "The statement 'studies on its effectiveness are inconclusive' is presented without evidence. Which studies? What data?" — models exactly the questioning students should learn.
- **Catches hidden bias.** Counts emphasis on negative aspects, identifies weasel words, explains *how* framing creates asymmetry despite claiming balance.
- **Pinpoints weakest argument** with a specific logical problem — the cost argument's expense is a feature of safeguards, not a bug of the death penalty itself.
- **Names specific missing perspectives** rather than vague "there are other views."
- **Delivers a blunt verdict** — student clearly understands the response has real problems.
- **Specific, researchable follow-up questions.**

**Weaknesses:**
- The verdict may be slightly too harsh — "superficial and unbalanced" could feel dismissive to a student who thought the answer was reasonable. Risk of defensiveness rather than engagement.
- Point 5 (challenge the framing) gestures at "deeper philosophical questions" without actually naming them — the same vagueness it criticizes the AI response for.

### Key Insight

V2 is substantively better because of **operationalization** — the prompt tells the model *how* to perform each analytical step. But the purely adversarial tone creates a pedagogical risk: students may dismiss the critique as "too harsh" or feel attacked, reducing learning impact.

---

## 7. Iteration 3: Mentor Critique (V3)

**File:** `index.html`
**Header:** "Challenge Myself — v3 — Mentor Critique"

### System Prompt

```
You are a rigorous but encouraging critical thinking mentor. An AI produced
a response to a student's question. Your job is to first acknowledge what
the response does well, then systematically dismantle its weaknesses — so
the student learns to appreciate good reasoning AND spot bad reasoning in
the same answer.

Structure your critique in exactly two sections:

## WHAT THIS RESPONSE GETS RIGHT
In 1-3 sentences, identify what the response genuinely does well. Be
specific — name the particular strength (e.g., "It correctly identifies X"
or "The structure makes it easy to compare both sides"). This must be
honest, not a token compliment. If the response is truly poor, say what it
*attempted* to do well even if it fell short.

## HERE'S THE PROBLEM
Now apply rigorous scrutiny:

1. UNSUPPORTED CLAIMS: Quote specific statements and flag them as lacking
   evidence. Don't accept "studies show" or "some argue" — demand which
   studies, who argues this, and what data supports it.

2. HIDDEN BIAS: If the response claims neutrality, test it. Count arguments
   per side, flag asymmetric framing, and identify weasel words that subtly
   steer the reader.

3. WEAKEST POINT: Identify the single least defensible claim and explain
   exactly why it fails.

4. WHAT'S MISSING: Name specific perspectives, data, or contexts that are
   absent — not just "other viewpoints exist" but which ones and why they
   matter.

5. BETTER FRAMING: Suggest a more productive way to think about the topic.
   What question should the student ACTUALLY be asking?

## YOUR NEXT STEPS
End with 2-3 specific, researchable questions the student should investigate
to go deeper than this AI response.

Be direct and confident in your critique. Use plain language. Say what IS
wrong, not what "could be improved." But maintain the tone of a mentor who
believes the student is capable of thinking harder — you're pushing them up,
not putting them down.

Keep your critique to 250-350 words.
```

### User Message

```
Original question: "${originalQuestion}"

AI response to critique:
"${aiResponse}"

First acknowledge what this response does well, then rigorously challenge
its weaknesses.
```

### Design Changes from V2

| Change | V2 | V3 | Why |
|--------|----|----|-----|
| Role | "tough examiner" | "rigorous but encouraging mentor" | Maintains analytical rigor while creating psychological safety |
| Opening | Jumps straight to flaws | **Required "WHAT THIS RESPONSE GETS RIGHT" section** | Validates student's initial understanding before challenging it — creates buy-in |
| Positive framing | None | "Must be honest, not a token compliment" | Prevents empty praise; the acknowledgment must be earned |
| "Challenge the Framing" | "Are there better frameworks?" (vague) | **"BETTER FRAMING: What question should the student ACTUALLY be asking?"** (explicit) | Higher-order skill — teaches students to question the question itself |
| Follow-up questions | Embedded in verdict | **Dedicated "YOUR NEXT STEPS" section** | Frames investigation as student empowerment, not homework |
| Tone guidance | "Do not soften" | "Say what IS wrong... But maintain the tone of a mentor who believes the student is capable" | Keeps directness but adds respect |
| Closing philosophy | None | "You're pushing them up, not putting them down" | Anchors the model's persona in constructive intent |

---

## 8. Evaluating V3 Output

### V3 Critique Output

**WHAT THIS RESPONSE GETS RIGHT:** "The response effectively introduces the complexity of the death penalty debate by acknowledging its ethical, legal, and practical dimensions. It also provides a balanced structure, presenting arguments both for and against."

**HERE'S THE PROBLEM:** (1) Flagged deterrence claim ("fear of death will discourage people") and closure claim as lacking empirical backing; (2) caught the 5-vs-4 argument asymmetry and flagged emotionally charged language ("irreversible nature... raises serious ethical concerns") appearing asymmetrically; (3) pinpointed public safety claim as weakest — life imprisonment also removes individuals from society; (4) named missing international perspectives, psychological impact on executioners, and potential for offender reform; (5) proposed reframed question: "What are the ethical and practical implications of maintaining or abolishing the death penalty in contemporary society?"

**YOUR NEXT STEPS:** Three specific research directions — deterrence studies, cost comparisons, and international case studies on abolition and crime rates.

### Evaluation: Grade A

**Strengths:**
- The opening section earns trust with honest, specific praise — not a throwaway compliment
- Retains all of V2's analytical sharpness: quotes claims, counts asymmetry, identifies weakest point
- "Better Framing" is the standout addition — the reframed question pushes students toward higher-order thinking
- Next steps are concrete and researchable — a student could actually act on them
- Tone balances directness with respect throughout

**Weaknesses:**
- The opening praises "balanced structure" but then exposes 5-vs-4 imbalance — pedagogically effective (shows surface balance can be misleading) but accidental rather than explicitly called out
- The closure claim is flagged as unsupported but not challenged substantively — V2 would have noted that research suggests execution can prolong trauma
- Slightly long (~350 words) but the clear section structure keeps it scannable

---

## 9. Three-Version Comparison

### Prompt Design Comparison

| Element | V1 — Gentle | V2 — Adversarial | V3 — Mentor |
|---------|-------------|-------------------|-------------|
| **Role** | "critical thinking coach" | "tough, no-nonsense examiner" | "rigorous but encouraging mentor" |
| **Core directive** | "critique to help think critically" | "tear it apart" | "acknowledge what's good, then dismantle weaknesses" |
| **Structure** | Unstructured bullet list | 6 numbered rules | 3 sections: Gets Right → Problem → Next Steps |
| **Tone** | "Specific and constructive" | "Say what IS wrong" | "Say what IS wrong... pushing them up, not down" |
| **Positive acknowledgment** | None | None | Required honest praise section |
| **Bias detection** | "Hidden assumptions or biases" (vague) | "Count arguments, flag weasel words" (operational) | Same operational approach as V2 |
| **Reframing** | Not included | "Better frameworks?" (brief) | "What question should they ACTUALLY be asking?" |
| **Follow-up** | "What should a critical thinker ask?" | "2-3 follow-up questions" (in verdict) | Dedicated "YOUR NEXT STEPS" section |
| **User message tone** | "Please provide a critical analysis" | "Tear this response apart" | "Acknowledge what's good, then challenge" |
| **Word limit** | 150-250 | 200-300 | 250-350 |

### Output Quality Comparison (Death Penalty Test)

| Dimension | V1 (B-) | V2 (A-) | V3 (A) |
|-----------|---------|---------|--------|
| **Catches unsupported claims** | Notes deterrence is contested but doesn't quote or demand evidence | Quotes specific claims, demands "Which studies? What data?" | Quotes deterrence and closure claims, flags both as lacking empirical backing |
| **Detects asymmetric framing** | Misses entirely | Catches it: counts arguments, flags emotionally charged language on one side only | Catches it: notes 5-vs-4 imbalance, flags asymmetric intensity |
| **Identifies weakest argument** | Notes life imprisonment also prevents reoffending (good but soft) | Pinpoints cost argument — reframes legal expense as safeguard feature, not bug (insightful) | Pinpoints public safety claim — life imprisonment achieves the same goal (clear and direct) |
| **Names missing perspectives** | Mentions restorative justice (vague) | Names restorative justice + psychological impact | Names international comparisons, psychological impact, offender reform potential |
| **Suggests better framing** | Not attempted | Gestures at "deeper philosophical questions" without naming them | Proposes specific reframed question about ethical/practical implications |
| **Actionable next steps** | Generic: "What evidence supports claims?" | Specific: deterrence studies, cost comparisons, societal impacts | Specific and researchable: deterrence studies, cost data, international case studies |
| **Tone** | Tentative: "could enrich," "would be helpful" | Blunt: "superficial and unbalanced" | Balanced: honest praise, then equally sharp analysis |
| **Student experience** | "The answer was okay but incomplete" | "That answer was seriously flawed" — may trigger defensiveness | "I see what was good AND what was wrong" — most likely to change thinking behavior |

### Pedagogical Assessment

| Factor | V1 | V2 | V3 |
|--------|----|----|-----|
| **Models critical thinking** | Tells students to think critically | Demonstrates how to attack an argument | Demonstrates both recognizing quality and spotting flaws |
| **Student receptiveness** | High comfort, low impact | Low comfort, high impact — risks defensiveness | **High comfort AND high impact** |
| **Skill transfer** | "I should question things" (abstract) | "Here's how to attack an argument" (one-sided) | "Here's how to evaluate — what's strong, what's weak, what to do next" (complete) |
| **Scaffolding** | None — unstructured paragraphs | Structured but critique-only | **Full scaffold**: appreciate → critique → reframe → investigate |
| **Pedagogical alignment** | Minimal — generic feedback | Adversarial debate training | **Closest to Socratic method** — recognition, questioning, self-directed inquiry |
| **Risk of dismissal** | Low risk, but low impact | Students may dismiss as "too harsh" or contrarian | **Lowest risk** — hardest to dismiss because it demonstrates fairness first |

---

## 10. Conclusions

### Why V3 Is the Strongest Version

V3 is the most effective version for teaching critical thinking, for three reasons:

1. **It models the complete behavior it wants students to learn.** V1 says "question things." V2 demonstrates how to question. V3 demonstrates how to *evaluate* — recognizing what's good, identifying what's bad, and knowing what to do next. This is the full critical thinking skill set.

2. **It creates productive discomfort without triggering defensiveness.** The "yes, and here's the problem" pattern validates the student's initial understanding before challenging it. A student who feels their comprehension was acknowledged is more willing to engage with subsequent critique. This mirrors established feedback frameworks like "commend-recommend" in education research.

3. **It gives students concrete next steps.** Each critique ends with specific, researchable questions — not abstract encouragement to "think harder" but actual directions for independent investigation.

### The Iteration Story

The three versions demonstrate an iterative prompt engineering methodology:

- **V1** proved the concept works but revealed that gentle, unstructured prompts produce gentle, forgettable output
- **V2** proved that operationalized prompts (telling the model *how* to critique, not just *what* to look for) produce substantively better analysis — but at the cost of pedagogical warmth
- **V3** synthesized V2's analytical rigor with a structure grounded in how effective teaching works — acknowledge competence, then challenge, then direct next steps

Each version was informed by evaluating the previous version's output against specific criteria, which is itself a form of the critical thinking process the tool is designed to teach.

