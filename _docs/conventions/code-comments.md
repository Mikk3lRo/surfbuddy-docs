# Code Comments

Comments are a last resort. Prefer a clearer name or a smaller function over a comment that explains one.

## General rules

- Do not comment what a function or block obviously does. If a reader would need a comment to see *what* the code does, consider renaming the identifiers or restructuring the code to make it clearer instead of explaining it.
- Add a comment only when the *why* is genuinely non-obvious from the code itself: a hidden constraint, a workaround for a specific external bug, or behavior that would otherwise surprise a reader.
- A function that seems to need a comment explaining what it does, may have outgrown its purpose, and in that case should usually be split into smaller, well-named functions instead of commented.
- When a comment is warranted, keep it brief - stating one non-obvious fact usually fits on one line. That's a consequence of brevity, not a hard limit: don't force a comment onto a single line if it genuinely needs a bit more room to stay clear. Either way, do not write multi-paragraph docblocks, historical narratives, or comments that recount how or why a decision was reached.
- Do not reference `_docs` pages from a code comment.
- `@param`/`@return` type annotations are not explanatory comments - keep using them for their normal purpose, but don't pad them with prose.

Shorten or remove a comment that violates these rules on sight when touching the surrounding code; don't preserve it out of caution.
