---
name: deep-code-review
description: Multi-lens adversarial code review protocol. Enforces zero-duplication & component reuse (DRY), security (IDOR/OWASP), concurrency & database ACID, Python/FastAPI/PEP 8 standards with Google-style docstrings, and React performance. Produces strict P0/P1/P2 triage with actionable code diffs.
when_to_use: "When reviewing code for quality, architecture, security, performance, or DRY compliance. When triggered by /review, PR reviews, or when evaluating any code changes before merging."
allowed-tools: Read, Grep, Glob, Bash
version: 2.0.0
---

# Deep Code Review — Multi-Lens Adversarial Audit

> **Philosophy:** A good reviewer is an adversarial yet constructive guardian of production. 
> Do not provide polite rubber stamps or vague suggestions. Approach every review asking:
> **"How will this code break in production under high load, edge cases, concurrent requests, or malicious input?"**

---

## 🎯 Core Review Standard: P0 / P1 / P2 Severity Grading

Every finding **MUST** be classified into one of three strict tiers. 
Vague advice like *"Consider adding error handling"* is **STRICTLY FORBIDDEN**. Every P0 and P1 finding **MUST** include an exact `diff` showing the fix.

| Tier | Severity | Definition | Policy |
|---|---|---|---|
| 🔴 **P0** | **Blocker** | Security vulnerability (IDOR, injection), data loss/corruption, runtime crash, broken business contract, unhandled async event loop freeze, **duplicate component created instead of reusing design system**. | **Blocks merge/deployment.** Must be fixed immediately. |
| 🟡 **P1** | **Major** | Performance bottleneck (N+1 query, missing index), unhandled network/API errors, missing `Raises:` documentation in public APIs, subtle race conditions, missing empty/loading states. | **Strongly recommended fix** before release. |
| 🟢 **P2** | **Polish** | Naming improvements, minor PEP 8 formatting, dead code cleanup, non-blocking refactoring. | **Optional / Advisory.** Does not block merge. |

---

## 🔬 The 6 Specialized Review Lenses

Scan the diff/code systematically through these 6 lenses:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       6-LENS SYSTEMATIC CODE SCAN                           │
├─────────────────────────────────────────────────────────────────────────────┤
│ 🔍 LENS 1: DRY & Component Integrity (Zero-Duplication Protocol)           │
│ 🛡️ LENS 2: Security & Tenant Isolation (IDOR, OWASP, Boundary Parsing)      │
│ ⚡ LENS 3: Concurrency, Reliability & Database ACID                         │
│ 🐍 LENS 4: Python / FastAPI / PEP 8 / Google Docstrings                    │
│ ⚛️ LENS 5: Frontend & React Architecture (Hydration, Re-renders, States)    │
│ 🏛️ LENS 6: Architectural Integrity & Anti-Shortcut (No Toy/MVP Hacks)      │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 🔍 LENS 1: DRY & Component Integrity (Zero-Duplication)

**Goal:** Prevent codebase bloat, fragmented UI, and reinvention of existing wheels.

#### 1.1 UI Component Re-use Audit
* 🔴 **P0:** Creating a raw HTML element (`<button>`, `<input>`, `<select>`, `<dialog>`, or custom `<div>` modal) when a design system component (`@/components/ui/button`, `@/components/ui/dialog`, `@/components/ui/input`) exists in the project.
* 🔴 **P0:** Embedding an inline `<svg>` instead of importing from the project's standard icon library (e.g. `lucide-react`, `lucide-vue-next`).
* 🟡 **P1:** Copy-pasting an existing component into a new file to change colors or padding.
  * *Required Action:* Use variants (e.g. CVA / `buttonVariants`) or composition (`<Button variant="secondary" />`).

#### 1.2 Utilities and Shared Logic Audit
* 🟡 **P1:** Re-implementing date formatting, currency masking, debounce, class merging (`cn(...)`), or API wrappers that already exist in `lib/` or `utils/`.
* 🟡 **P1:** Writing duplicate SQL/ORM queries in route handlers instead of reusing domain repositories/services.

---

### 🛡️ LENS 2: Security & Tenant Isolation

**Goal:** Zero-trust architecture, multi-tenancy protection, and input boundary security.

#### 2.1 Insecure Direct Object References (IDOR)
* 🔴 **P0:** Querying, mutating, or deleting a resource solely by `id` without verifying tenant ownership (`user_id` or `workspace_id`):
```python
# ❌ P0 IDOR Vulnerability:
query = select(Invoice).where(Invoice.id == invoice_id)

# ✅ Fixed:
query = select(Invoice).where(
    Invoice.id == invoice_id, 
    Invoice.organization_id == current_user.org_id
)
```

#### 2.2 Boundary Validation & Mass Assignment
* 🔴 **P0:** Passing unsanitized input directly into database models or ORM `update()` without a validated schema (Zod in TS, Pydantic v2 in Python).
* 🔴 **P0:** Exposing internal fields (password hashes, secret keys, internal IDs) in API responses.

#### 2.3 Secrets and Credentials
* 🔴 **P0:** Hardcoded API keys, passwords, or tokens. Using non-type-safe `process.env` / `os.getenv` without fallback validation (`t3-env` or Pydantic `BaseSettings`).

---

### ⚡ LENS 3: Concurrency, Reliability & Database ACID

**Goal:** Data integrity under concurrent load and high-traffic resilience.

#### 3.1 Multi-Record Mutations & Transactions
* 🔴 **P0:** Mutating 2 or more related database records (e.g. balance deduction + transaction record) outside an explicit database transaction block (`db.transaction()` or `async with session.begin():`).
* 🟡 **P1:** Missing optimistic locking (`version` column) or `SELECT ... FOR UPDATE` on financial/inventory balance mutations prone to race conditions.

#### 3.2 Database Hygiene & Queries
* 🔴 **P0:** Unbounded database queries (`select *` or `findMany()` without an explicit `LIMIT` / pagination).
* 🟡 **P1:** Querying on unindexed foreign keys or filter columns used frequently in `WHERE` / `JOIN` / `ORDER BY`.

#### 3.3 Network & Timeout Resilience
* 🟡 **P1:** Client-side async fetch without `AbortController` (causes memory leaks and race conditions on tab/page switch).
* 🟡 **P1:** External HTTP calls without explicit timeout settings (causes worker thread exhaustion if external service hangs).

---

### 🐍 LENS 4: Python / FastAPI / PEP 8 & Documentation Standards

**Goal:** High-performance async Python, clean architecture, and world-class documentation.

#### 4.1 Async Event Loop Protection
* 🔴 **P0:** Synchronous blocking I/O (`requests.get`, `time.sleep`, standard `open()`, synchronous DB drivers) inside `async def` endpoints:
```python
# ❌ P0: Freezes the entire FastAPI event loop for all users
@router.get("/weather")
async def get_weather():
    time.sleep(1)
    return requests.get("https://api.weather.com").json()

# ✅ Fixed:
@router.get("/weather")
async def get_weather(http_client: httpx.AsyncClient = Depends(get_http_client)):
    await asyncio.sleep(1)
    res = await http_client.get("https://api.weather.com")
    return res.json()
```

#### 4.2 SQLAlchemy 2.0 Async Pitfalls
* 🔴 **P0:** Lazy-loading relational attributes in async code triggering `MissingGreenlet` exceptions. Must use `selectinload()` or `joinedload()`.
* 🔴 **P0:** Unclosed database sessions or sessions created outside dependency injection scopes.

#### 4.3 Google-Style Docstrings & API Documentation
Every public function, service method, and API endpoint **MUST** have a Google-style docstring explaining business logic, arguments, side effects, and explicitly listing raised exceptions:

```python
async def transfer_funds(
    db: AsyncSession,
    sender_id: UUID,
    recipient_id: UUID,
    amount_cents: int,
    idempotency_key: str
) -> TransferReceipt:
    """Transfers funds between two accounts within an atomic transaction.

    Applies row-level locks on sender and recipient balances to eliminate
    concurrency race conditions.

    Args:
        db: Async database session.
        sender_id: Unique UUID of the sending user.
        recipient_id: Unique UUID of the recipient user.
        amount_cents: Transfer amount in integer cents (> 0).
        idempotency_key: Unique client request identifier for deduplication.

    Returns:
        TransferReceipt: Object containing transfer status, transaction IDs, and timestamp.

    Raises:
        InsufficientBalanceError: If sender balance is less than `amount_cents`.
        AccountNotFoundError: If sender or recipient account does not exist.
        DuplicateTransactionError: If transaction with `idempotency_key` was already executed.
        DatabaseError: If database commit fails.

    Side Effects:
        - Deducts sender balance and increments recipient balance.
        - Emits an asynchronous audit event to Redis/RabbitMQ.
    """
```
* 🟡 **P1:** Endpoint or service throws custom domain exceptions that are not documented in `Raises:`.
* 🟢 **P2:** Missing docstring on private helper function where the logic is simple.

#### 4.4 PEP 8 & Modern Python 3.10+ Typing
* **Naming:** `snake_case` for functions/variables/modules; `PascalCase` for classes/Pydantic models; `UPPER_SNAKE_CASE` for constants.
* **Modern Typing:** Use `X | None` instead of `Optional[X]`, `list[str]` instead of `List[str]`, `dict[str, Any]` instead of `Dict[str, Any]`.
* 🟢 **P2:** Mutable default argument `def func(items: list = [])` -> Change to `items: list[str] | None = None`.

---

### ⚛️ LENS 5: Frontend & React Architecture

**Goal:** Smooth UI/UX, predictable state management, and SSR safety.

#### 5.1 Re-render Traps & Hook Dependencies
* 🟡 **P1:** Instantiating new objects/arrays in dependency arrays causing infinite loops in `useEffect`.
* 🟡 **P1:** Missing cleanup functions in `useEffect` for subscriptions, timers, or event listeners.

#### 5.2 SSR & Hydration Safety (Next.js / Nuxt)
* 🔴 **P0:** Direct access to `window`, `document`, or `localStorage` during initial render (causes React Hydration Mismatch). Must use `useEffect` or `useSyncExternalStore`.

#### 5.3 Complete UI States (Loading, Error, Empty)
* 🟡 **P1:** Component renders data without handling:
  1. **Loading state:** Skeleton / loader indicator.
  2. **Error state:** User-friendly message with retry action.
  3. **Empty state:** Meaningful empty illustration and call-to-action when data array is `[]`.

---

### 🏛️ LENS 6: Architectural Integrity & Anti-Shortcut (Production Baseline)

**Goal:** Prevent "MVP degradation", toy hacks, pseudo-navigation, and architectural laziness disguised as "speed of prototyping".

#### 6.1 Routing & Navigation Integrity (Hard Ban on Pseudo-Routing)
* 🔴 **P0 (Blocker):** Using `useState('activeTab')` or conditional rendering to switch top-level pages/screens when the application has multiple distinct views.
  * *Policy:* If the app has >1 screen/page, it **MUST** use a declarative router (`react-router-dom`, `@tanstack/react-router`, Next.js App Router, Vue Router).
  * *Policy:* Navigation must support browser history (Back/Forward), bookmarkable URLs, and deep linking.
* 🔴 **P0 (Blocker):** Omitting build/server fallback configurations (e.g. Vite SPA fallback, Nginx `try_files $uri $uri/ /index.html`) when implementing client-side routing.
* 🟡 **P1 (Major):** Multi-view filters, sorting, tabs inside a page, or pagination not synchronized with URL query parameters (`searchParams`).

#### 6.2 Unauthorized MVP Degradation
* 🔴 **P0 (Blocker):** Downgrading architecture to in-memory mocks, fake timers, or skipping persistence/API layers under the pretext of "quick MVP", unless the user **EXPLICITLY** requested a throwaway prototype.
* 🔴 **P0 (Blocker):** Single-file "god components" (>250 lines mixing state, API calls, types, styling, and navigation) written to avoid creating proper module structures.
* 🟡 **P1 (Major):** Missing standard error boundaries or unhandled 404 / Not Found route handlers.

---

## 🤖 The Critic Delegation Contract (Subagent Verification)

When an orchestrator or parent agent invokes `code-reviewer` as an adversarial critic subagent, the prompt **MUST** follow this structured contract:

```text
GOAL: Adversarial architectural review and verification gate
USER_INTENT: [Original user request / requirement]
DIFF_SUMMARY: [Summary of changes and changed file paths]
CHANGED_FILES:
[Full diff or list of edited files with line ranges]

AUDIT_INSTRUCTION:
Review the changes against the 6 Lenses of @[skills/deep-code-review].
Specifically verify:
1. Did the developer take shortcuts (e.g., useState pseudo-routing instead of real router)?
2. Are all states (loading, error, empty) properly handled?
3. Is DRY respected and were design system components reused?
4. Is production-first baseline maintained (no unauthorized MVP hacks)?

OUTPUT: Return standard Code Review Report with [🔴 REJECTED | 🟢 APPROVED] and drop-in fix diffs for any P0/P1.
```

---

## 📋 Mandatory Output Format for Reviews

When performing a review, output the report strictly using this template:

```markdown
# 🛡️ Code Review Report

## 📊 Summary: [ 🔴 REJECTED (P0 Blockers) | 🟡 APPROVED WITH COMMENTS | 🟢 APPROVED ]
**Files Audited:** `<list of files>`  
**Blast Radius:** `<impacted modules / dependencies>`

---

### 🔴 P0 (Blockers — Must Fix Immediately)
#### 1. [Category / Rule] Brief Title
* **Location:** `path/to/file.ext:line_number`
* **Problem:** Clear explanation of why this will fail, leak data, or violate DRY.
* **Fix (Diff):**
\`\`\`diff
- old buggy code
+ new fixed code
\`\`\`

---

### 🟡 P1 (Major Issues — High Priority)
#### 1. [Category / Rule] Brief Title
* **Location:** `path/to/file.ext:line_number`
* **Problem:** Explanation of performance risk, missing `Raises:`, or race condition.
* **Fix (Diff):**
\`\`\`diff
- old code
+ fixed code
\`\`\`

---

### 🟢 P2 (Polish / Advisory — Non-Blocking)
* **`path/to/file.ext:line`**: Suggestion for cleaner naming or PEP 8 adherence.

---

### 🎯 Next Steps
[Clear actionable instruction on what needs to be edited before proceeding]
```
