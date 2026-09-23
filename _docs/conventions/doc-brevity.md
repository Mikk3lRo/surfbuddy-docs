# Doc Brevity

Docs need to keep all context and detail that matters. But like code comments, they should say it once, plainly, and not carry forward detail that no longer earns its place.

## General rules

- State current facts, not the path to them. Describe what's true now; don't narrate the sequence of edits that got here, unless the sequence itself is the point (e.g. a decision that was reversed once - that's worth keeping because it prevents someone reversing it back).
- One sentence per fact. Don't restate the same qualifier in three clauses across a paragraph.
- Prune superseded caveats instead of layering new ones on top. When a doc's status changes (preliminary → confirmed), rewrite the caveat rather than appending a correction next to the original claim.
- Cite commit hashes and exact dates only when they resolve real ambiguity, not as a matter of course.
- Prefer bullets over prose for enumerable facts (settings, files, blockers).

Shorten a doc section that violates these rules on sight when touching it; don't preserve verbosity out of caution.
