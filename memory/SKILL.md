---
name: memory
description: Maintain the agent's core memory — a single bounded, self-edited document of user preferences and durable facts.
metadata:
  tool: memory_edit
  category: tools
---

# Memory

Your memory is a single bounded text document (core memory). You read it and
rewrite it in-place — you do NOT append discrete facts. This prevents
duplication, fragmentation, and contradictions.

## Instructions

Execute all steps silently.

### When to update memory

Update memory when the user:

- Tells you a preference ("I prefer BTC", "my main wallet is...")
- Shares personal context (name, occupation, goals)
- Explicitly asks you to remember something
- Changes a previously-stated preference (rewrite, don't append)

Do NOT save trivial or one-off questions.

### The self-editing loop

1. **View** — read the current memory first:
   - **function**: "memory_edit"
   - **data**: `{"action": "view"}`

2. **Reconcile** — decide:
   - New fact → `append`
   - Changed/contradicted fact → `replace` the existing line (never append a contradiction)
   - Stale fact → `replace` to remove or correct it

3. **Rewrite** — apply the edit:
   - **function**: "memory_edit"
   - **data**: `{"action": "replace", "old_str": "<exact existing text>", "new_str": "<new text>"}`
   - **function**: "memory_edit"
   - **data**: `{"action": "append", "new_str": "<new line>"}`

`old_str` must match the document exactly once. If it appears more than once,
include more surrounding context to make it unique. If it is not found, re-read
with `view` and retry.

### Size cap and consolidation

The document is capped (~2400 chars). When it approaches the cap, consolidate:
merge related lines, drop stale/trivial entries, and keep only what matters for
future recommendations. Prefer rewriting a section over appending.

### Search history (archival)

For details from past conversations that don't belong in core memory, use:

- **function**: "search_history"
- **data**: `{"query": "<keywords>", "scope": "<messages|conversations|all>", "limit": <max_results>}`

## Constraints

- One fact per line, concise.
- Never append a fact that contradicts an existing line — replace it.
- Do not save information the user explicitly asked you to forget.
- Do not save transient data (prices, timestamps that will be stale) or secrets.
