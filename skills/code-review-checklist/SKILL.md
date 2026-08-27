---
name: code-review-checklist
description: Code review guidelines covering code quality, security, and best practices. Redirects to deep-code-review for production-grade multi-lens audits.
when_to_use: "When reviewing code for quality, security, and best practices. When the user says 'review my code' or 'check this PR'."
allowed-tools: Read, Glob, Grep
version: 2.0.0
---

# Code Review Checklist (Quick Reference)

> 🔴 **MANDATORY:** For full-depth production reviews, use the **`@[skills/deep-code-review]`** skill.

---

## ⚡ Quick 6-Lens Triage

### 1. DRY & Component Integrity (Zero-Duplication)
- [ ] No raw HTML elements (`<button>`, `<input>`) when design system components exist in `@/components/ui/`.
- [ ] No inline `<svg>` — use standard icon library (Lucide).
- [ ] No duplicate utility functions (date formatting, API wrappers) already present in `utils/` or `lib/`.

### 2. Security & Tenant Isolation
- [ ] IDOR check: Every query by `id` includes `user_id` or `workspace_id`.
- [ ] All inputs validated via Zod (TypeScript) or Pydantic v2 (Python).
- [ ] No exposed secrets or internal sensitive fields in response schemas.

### 3. Concurrency & Database ACID
- [ ] Multi-table mutations wrapped in explicit transactions.
- [ ] All queries have explicit `LIMIT` / pagination.
- [ ] Client fetch calls include `AbortController`.

### 4. Python / FastAPI / PEP 8
- [ ] No blocking I/O (`requests`, `time.sleep`) inside `async def`.
- [ ] SQLAlchemy 2.0: No lazy loads in async (use `selectinload`).
- [ ] Google-style docstrings with `Args:`, `Returns:`, `Raises:`, `Side Effects:`.
- [ ] PEP 8 and modern Python 3.10+ typing (`X | None`, `list[str]`).

### 5. Frontend & React Performance
- [ ] No re-render loops or unstable object references in `useEffect` deps.
- [ ] SSR safe: No direct `window`/`localStorage` in initial render.
- [ ] Complete UI states: Loading skeleton, Error message, Empty state.

### 6. Architectural Integrity & Anti-Shortcut
- [ ] No pseudo-routing via `useState('tab')` for multi-screen apps (real declarative router required).
- [ ] URLs synchronized with views, filters, and top-level tab states.
- [ ] Production-first baseline: No unauthorized in-memory mock stores or missing SPA fallback configs.

---

## 🚦 Severity Grading

- 🔴 **P0 (Blocker):** Security vulnerability, data loss, crash, duplicate component created, state-based pseudo-routing instead of real router, unauthorized MVP shortcuts. **Blocks merge.**
- 🟡 **P1 (Major):** Performance issue, missing `Raises:` docs, race condition, missing empty state.
- 🟢 **P2 (Polish):** Naming, formatting, minor PEP 8 adjustments.

> 📖 **Full Protocol & Diff Examples:** See [`../deep-code-review/SKILL.md`](../deep-code-review/SKILL.md).
