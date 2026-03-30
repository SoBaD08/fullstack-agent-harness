# Fullstack Coding Agent Harness - Architecture Design

> Based on Anthropic's blog post "Harness Design for Long-Running Apps"

---

## 1. Core Design Principles

### Separation of Generation & Evaluation
When agents evaluate their own work, they tend to "confidently praise it — even when the quality is obviously mediocre."
An **independent Evaluator Agent** verifies code from a separate context.

### Decomposition Over Monolithic Approaches
Implementing large features in one shot causes agents to "go off the rails."
Break work into **Sprint-sized chunks** for incremental implementation + verification.

### Context Management
As sessions grow long, agents develop "context anxiety" (premature task completion as perceived context limits approach).
Use **structured handoff files** to pass state between agents, and reset context when needed.

### Grading Criteria as Steering Mechanisms
Instead of vague "make it good," provide **concrete, measurable criteria** so models can optimize reliably.
The prompting associated with criteria directly shapes the character of the output.

---

## 2. 3-Agent Architecture

```
┌─────────────┐    spec.md     ┌─────────────┐   contract.md   ┌─────────────┐
│   PLANNER   │ ─────────────► │  GENERATOR   │ ◄──────────────► │  EVALUATOR  │
│             │                │  (Builder)   │                 │             │
│ - Analyze   │                │ - Sprint impl│    report.md    │ - Run tests │
│   requirements│              │ - Self-check │ ◄────────────── │ - Code review│
│ - Write spec│                │ - Git commit │                 │ - UI/API QA │
└─────────────┘                └──────┬───────┘                 └─────────────┘
                                      │
                                      │ PASS/FAIL
                                      ▼
                               ┌──────────────┐
                               │ Next Sprint  │
                               │ or Complete  │
                               └──────────────┘
```

### Agent 1: Planner
- **Role**: Expands short requirements (1-4 sentences) into comprehensive product specs
- **Input**: User's feature request
- **Output**: `.harness/spec.md` (detailed spec document)
- **Focus**: What to build, not how to build it

### Agent 2: Generator (Builder)
- **Role**: Incrementally implements features in Sprint units
- **Input**: `spec.md` + `contract.md` (evaluation contract)
- **Output**: Actual code + Git commits + `build-report.md`
- **Flow**: Self-validates after each Sprint, then hands off to Evaluator

### Agent 3: Evaluator
- **Role**: Independently verifies implementation quality
- **Input**: Current codebase + `contract.md`
- **Output**: `evaluation-report.md` (PASS/FAIL + detailed feedback)
- **Methods**: Playwright MCP for live UI testing, direct API endpoint validation

---

## 3. Communication Pattern: File-Based Handoff

Agents communicate through **structured files** rather than direct messaging:

```
.harness/
├── spec.md              # Planner → Generator (product spec)
├── contract.md          # Generator ↔ Evaluator (Sprint contract)
├── build-report.md      # Generator → Evaluator (implementation report)
├── evaluation-report.md # Evaluator → Generator (evaluation results)
├── context/
│   ├── decisions.md     # Architecture decision log
│   └── blockers.md      # Known issues / blockers
└── sprints/
    ├── sprint-1/
    │   ├── goals.md
    │   ├── completed.md
    │   └── review.md
    └── sprint-2/
        └── ...
```

---

## 4. Sprint Workflow

```
1. Planner: Generate spec.md
   ↓
2. Generator + Evaluator: Negotiate contract.md
   (Agree on Sprint success criteria)
   ↓
3. Generator: Implement Sprint → build-report.md
   ↓
4. Evaluator: Verify code + UI/API testing → evaluation-report.md
   ↓
5a. PASS → Next Sprint (back to step 2)
5b. FAIL → Generator fixes based on feedback (back to step 3)
   ↓
6. All Sprints complete → Final report
```

---

## 5. Evaluation Criteria Framework

Per the blog's key insight: agents already score well on craft and functionality by default.
**Design Quality and Originality are weighted heavily** to push beyond generic "AI slop."

### Design Quality - 20% (high weight)
Does the UI feel like a **coherent whole** rather than a collection of parts?
- Colors, typography, layout, imagery combine to create a distinct mood and identity
- Consistent visual language across all screens
- Clear visual hierarchy guiding the user's eye

### Originality - 15% (high weight)
Is there evidence of **deliberate creative choices**, or is this template defaults?
- Penalizes: purple gradients over white cards, generic hero sections, unmodified stock components
- Rewards: custom hover states, branded empty states, thoughtful micro-animations
- A human designer should recognize intentional decisions

### Functionality - 20%
- API endpoints behave according to spec
- Users can complete tasks without guessing — primary actions are obvious
- Error handling is appropriate

### Code Quality - 15%
- Follows project conventions
- No duplicated code
- Appropriate level of abstraction

### Craft - 10%
Technical execution of visual design:
- Typography hierarchy, spacing consistency, color harmony
- Contrast ratios ≥ 4.5:1 for text
- Most reasonable implementations pass here — failing means broken fundamentals

### Testing - 10%
- Unit tests exist and are meaningful
- Edge cases are covered
- All tests pass

### Security - 10%
- SQL injection prevention
- Authentication/authorization handling
- No sensitive data exposure

**PASS threshold**: Overall weighted score ≥ 70%, each individual category ≥ 50%

### Strategic Feedback Loop
After each evaluation, the Evaluator recommends one of:
- **Refine**: Design scores trending well → polish the current direction
- **Pivot**: Approach isn't working → start fresh with a different aesthetic

This is directly from the blog: the Generator was instructed to "make a strategic decision
after each evaluation: refine the current direction if scores were trending well, or pivot
to an entirely different aesthetic if the approach wasn't working."

---

## 6. Context Reset Strategy

### Context Anxiety (from blog)
Models "begin wrapping up work prematurely as they approach what they believe is their
context limit." Symptoms include: rushing to finish, skipping planned features,
producing lower quality output toward the end of a session.

### Solution: Structured Handoff
When the context window becomes saturated:

1. **Serialize current state to `.harness/` files** (spec, contract, progress, decisions)
2. **Start a new agent session** (full context reset)
3. **Restore state from `.harness/` files** in the new session

The file-based communication pattern makes this seamless — all inter-agent state
already lives in `.harness/`, so nothing is lost during a reset.

### When to Reset
- Agent starts rushing or producing noticeably lower quality output
- Agent summarizes remaining work as "minor cleanup" when it clearly isn't
- Very long sessions (multiple Sprints in one context)
- Note: Newer models (Opus 4.5+) largely handle this on their own

## 7. Evaluator Skepticism Tuning (from blog)

The blog's key finding: "Out of the box, Claude is a poor QA agent" — agents naturally
praise mediocre work when asked to evaluate it.

### How to Calibrate
1. Read evaluator logs and compare scores against your own human judgment
2. Where scores diverge, update the evaluate prompt to address the gap
3. Repeat for several evaluation cycles until alignment improves
4. "Tuning a standalone evaluator to be skeptical turns out to be far more tractable
   than making a generator critical of its own work"

### Anti-Patterns to Watch For
- Evaluator says "looks good" without running the code
- All scores clustered around 75-85 (suspiciously uniform)
- PASS verdict with unresolved test failures
- Design Quality scored high but screenshots show generic UI

---

## 8. Default Tech Stack

| Layer | Technology | Notes |
|-------|-----------|-------|
| Frontend | React + Vite (or Next.js) | SPA or SSR |
| Backend | FastAPI (Python) or Express (Node) | REST API |
| Database | SQLite (dev) / PostgreSQL (prod) | Migrations required |
| Testing | pytest / Jest + Playwright | Unit + E2E |
| VCS | Git | Commit per Sprint |

---

## 9. Cost Estimates

Based on the blog post case studies:
- **Solo agent**: ~20 min, ~$9 → incomplete functionality, broken features
- **3-Agent harness**: ~4 hours, ~$125 → feature-complete with polish

The harness is **cost-effective only for complex features**.
Simple tasks should use a single agent directly.

---

## 10. Two Operating Modes

This harness supports two fundamentally different ways to run:

### Mode A: Manual (Interactive)
**Commands**: `/plan`, `/build`, `/evaluate`, `/orchestrate`

```
User → /plan → [review spec] → /build → [review code] → /evaluate → [review report] → repeat
```

- User invokes each step manually via slash commands
- Full control: review and adjust between every phase
- Lower cost (user catches issues early)
- Best for: iterative development, learning, critical features

### Mode B: Autonomous (Hands-Off)
**Commands**: `/auto-orchestrate` or `./harness.sh`

```
User → /auto-orchestrate "build X" → [agents run for hours] → final report
```

- Agents run the full Plan → Build → Evaluate loop without stopping
- Evaluator runs as a **subagent** with fully isolated context (true separation)
- Auto-retries on FAIL (up to 3 attempts per Sprint)
- Emergency stop after 3 consecutive failures
- Best for: complete feature builds, overnight runs, well-defined requirements

### When to Use Which

| Scenario | Recommended Mode |
|----------|-----------------|
| Exploring a new feature idea | Manual |
| Well-defined spec, want it built | Autonomous |
| Debugging/fixing existing code | Manual (or skip harness) |
| Building a complete MVP overnight | Autonomous |
| Learning how the agents work | Manual |
| Production-critical code | Manual (more oversight) |

### The `harness.sh` Script (CLI Autonomous Mode)

For maximum autonomy, `harness.sh` runs the entire pipeline from the terminal:

```bash
# Basic usage
./harness.sh "Build a real-time chat app with WebSocket support"

# With options
./harness.sh "Add Stripe payment processing" \
  --max-sprints 3 \
  --max-retries 2 \
  --model opus \
  --max-budget 50

# Resume after fixing issues manually
./harness.sh --skip-plan "Continue from existing spec"
```

This script:
- Invokes `claude -p` for each agent phase (separate process = separate context)
- Parses evaluation reports to determine PASS/FAIL automatically
- Archives Sprint artifacts and manages git tags
- Logs everything to `.harness/logs/`
- Exits with a failure summary if a Sprint can't be fixed

---

## 11. Architecture Comparison: Skills vs Subagents

| Aspect | Skills (current) | Subagents |
|--------|------------------|-----------|
| Context | `context: fork` = isolated | Fully isolated process |
| User control | User invokes explicitly | Agent invokes programmatically |
| Tool access | Configurable via `allowed-tools` | Inherits or configured |
| Communication | `.harness/` files | `.harness/` files |
| Best for | Manual mode | Autonomous mode |

Both approaches use the same **file-based handoff** pattern. The difference is
who decides when to invoke the next step: the user (Skills) or the orchestrator (Subagents).

The Evaluator benefits most from isolation — running in a separate context prevents it
from being influenced by the Generator's reasoning, which is exactly what the blog post
identified as the key insight: agents praise their own work unless evaluation is truly independent.
