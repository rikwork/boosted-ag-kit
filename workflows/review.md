---
name: review
description: Run deep adversarial code review on changes, files, or PRs. Evaluates DRY/components, security (IDOR), reliability, Python/FastAPI/PEP 8, and React performance.
version: 1.0.0
requires_agents: code-reviewer
requires_skills: deep-code-review, clean-code, python-patterns, nextjs-react-expert, code-review-graph
artifact_outputs: code-review-report
---

# /review — Deep Adversarial Code Review

$ARGUMENTS

---

## 🔴 CRITICAL RULES

1. **Adversarial Mindset** — Approach the code asking: *"How will this break in production?"*
2. **5-Lens Systematic Audit** — Apply all 5 lenses from `@[skills/deep-code-review]` (DRY, Security, Concurrency, Python/PEP 8, React).
3. **Strict Severity Grading** — Classify findings into 🔴 **P0 (Blocker)**, 🟡 **P1 (Major)**, 🟢 **P2 (Polish)**.
4. **Mandatory Actionable Diffs** — Every P0 and P1 issue **MUST** include an exact `diff` showing the fix.

---

## Task

Use the `code-reviewer` agent and `deep-code-review` skill to perform a thorough audit:

```
CONTEXT:
- Target to review: $ARGUMENTS
- If empty: inspect current git diff (uncommitted / staged / latest commit)

WORKFLOW:
1. DISCOVER: Identify all changed files and their dependency blast radius (via AST graph if available).
2. SCAN 5 LENSES:
   ├── Lens 1 (DRY): Verify no raw HTML elements or duplicated components/utilities.
   ├── Lens 2 (Security): Check for IDOR, tenant scoping, Zod/Pydantic validation boundaries, secrets.
   ├── Lens 3 (Reliability): Verify ACID transactions, query LIMITs, AbortControllers.
   ├── Lens 4 (Python): Check for event loop blocking in async def, SQLAlchemy N+1, Google docstrings with Raises:, PEP 8.
   └── Lens 5 (Frontend): Check for re-render traps, SSR hydration safety, Loading/Error/Empty states.
3. TRIAGE: Grade issues into P0 (Blocks merge), P1 (Major), P2 (Polish).
4. GENERATE DIFFS: Provide copy-pasteable diff blocks for every P0 and P1 finding.
5. REPORT: Output the final Code Review Report.
```

---

## Expected Output

```markdown
# 🛡️ Code Review Report

## 📊 Summary: [ 🔴 REJECTED (P0 Blockers) | 🟡 APPROVED WITH COMMENTS | 🟢 APPROVED ]
- **Audited Scope:** `<files audited>`
- **Blast Radius:** `<impacted modules>`

---

### 🔴 P0 (Blockers — Must Fix Immediately)
#### 1. [Category] Description
* **Location:** `file.ext:line`
* **Problem:** Why this breaks production or violates DRY.
* **Fix (Diff):**
```diff
- old code
+ fixed code
```

---

### 🟡 P1 (Major Issues — High Priority)
...

---

### 🟢 P2 (Polish / Advisory)
...

---

### 🎯 Next Steps
...
```
