---
name: memory
description: Save and search persistent memory across conversations.
metadata:
  tool: memory
  category: tools
---

# Memory

Save and search the user's persistent memory.

## Instructions

Execute all steps silently.

### When to save

Save to memory when the user:

- Tells you a preference ("I prefer BTC", "my main wallet is...")
- Shares personal context (name, occupation, goals)
- Explicitly asks you to remember something
- Completes a significant action worth recalling later

Do NOT save trivial or one-off questions.

### Save

Call `run_js` with:

- **function**: "memory"
- **data**: `{"action": "save", "content": "<fact to remember>", "category": "<preference|identity|goal|fact|observation>", "importance": <0-5>}`

`category` defaults to "fact", `importance` defaults to 0. Use "preference" for
likes/dislikes, "identity" for stable facts about the user, "goal" for objectives,
"fact" for general durable findings, "observation" for lower-value notes.

### Search

Call `run_js` with:

- **function**: "memory"
- **data**: `{"action": "search", "query": "<keywords>", "limit": <max_results>}`

`limit` defaults to 5.

### Forget

Call `run_js` with:

- **function**: "memory"
- **data**: `{"action": "forget", "content": "<exact fact text to delete>"}`

Use `forget` when the user asks you to forget something, or when a saved fact is
now wrong and should be removed rather than corrected.

## Constraints

- Keep saved content concise — one fact per entry.
- Saving the same fact again updates it (no duplicates).
- Use "preference"/"identity"/"goal"/"fact" for durable facts; "observation" for transient notes.
- Do not save information the user explicitly asked you to forget.
