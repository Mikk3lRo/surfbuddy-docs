# Surfbuddy AI Workspace Instructions
- Be brief in all responses.
- Ask questions if a request is unclear or ambiguous. Never guess!
- Try to not ask more than 3 questions in one turn. Ask broader questions that helps narrow down details in quicker turns.
- Never start work on files before all details of the assignment are completely clear.
- When discovering unexpected, generally useful patterns in existing code or user-preferences, suggest updating relevant documentation. You decide what may be relevant. User decides if it should in fact be noted.

## Conventions - read every one of these before doing any work

These are unconditional rules that apply to every change in every repo, regardless of what the task appears to be about. The main points highlighted in the summary below are not a substitute for the full doc - always read the full doc.

- [Error handling](_docs/conventions/error-handling.md): fail loudly on unexpected errors - never swallow `Throwable`/`Error`/a broad `Exception` into just a log line; catch only expected failures the caller can handle.
- [Automated testing](_docs/conventions/automated-testing.md): no automated tests in any Surfbuddy application repository, unconditionally.
- [Code comments](_docs/conventions/code-comments.md): comments are a last resort - never comment what's obvious, never add one just because a property/method is new, only for a genuinely non-obvious *why*, always in English.
- [Doc brevity](_docs/conventions/doc-brevity.md): state current facts once, plainly - no edit-history narration, no restated qualifiers, prune superseded caveats instead of layering on top.

New unconditional rules like these should have their own file, created under `_docs/conventions/`, and added to this list.

## Conventions - read if relevant

Each is a detailed description with rules for how to do work in a specific part of the system. None exist yet.

Repeated "types" of work within a specific part of the system should be described in a file under `_docs/conventions/` and added to this list.

## Workspace purpose and boundaries

- This outer repository contains only AI instructions and documentation for the Surfbuddy system.
- The application repositories below it remain independent Git repositories with separate histories, branches, and releases. Never stage or commit changes made in those.
- Known application repositories include `surfbuddy-server` (server installation and setup), `surfbuddy-v3` (API and forecast-generation scripts), and `surfbuddy-vue` (frontend).
- Never add AI-specific files to an application repository. This includes `AGENTS.md`, `.codex`, AI notes and documentation.
- Never write temporary, generated, staging, scratch, analysis, or intermediary files anywhere inside an application repository. Use a folder called .tmp in the outer workspace or the system temporary directory instead.
- Store AI-maintained documentation in the outer `_docs` directory.

## Explicit approval is required for changes

- Never create, modify, delete, move, rename, format, generate, stage, or commit files in the application repositories without the user's explicit approval.
- Read-only inspection, analysis, discussion, and planning are allowed unless the user explicitly restricts them.
- Approval applies only to the specific repositories, change, and scope described by the user. Do not infer approval for adjacent refactoring, cleanup, dependency updates, generated files, documentation changes, or changes in another repository.
- If completing an approved task requires an additional change outside the approved scope, stop and request explicit approval before making it.

## Multi-repository Git safety

- Treat the outer AI repository and every application repository as separate Git worktrees.
- Check the relevant application repository with `git status --short` before making changes.
- A dirty working tree does not block continued work when every existing change is clearly part of the same ongoing, user-approved task or an immediately preceding approved change in the same collaboration.
- If the working tree contains unrelated, unexpected, or ambiguously owned changes, stop and ask the user before modifying files. Never overwrite or revert existing changes.
- Read-only inspection commands are allowed against application repositories: `git status --short`, `git diff`, `git log`, `git show`, `git blame`, and similar commands that only inspect history or the working tree (including with `git -C <repository>`).
- Never run any Git command that changes repository or history state (commit, push, pull, fetch, merge, rebase, reset, checkout, restore, clean, branch, tag, stash, etc.) against application repositories. **Only** the user does that.
- For the outer repository, committing requires explicit approval.

## Documentation

- Always write documentation in English.
- Store all AI-maintained project documentation under the outer `_docs` directory and maintain it there.
- Keep documentation modular so agents can read only the files relevant to the current task.
- Update relevant documentation when an approved change makes it inaccurate.
- Do not place documentation in application repositories.
- For a task scoped to one repository, start with that repository's project index below, then read only the linked area doc that matches the work - don't read the whole tree.
- Treat a preliminary/direction doc as a starting point, not an approved spec - confirm behavior in the actual code before a consequential change.

### Project indexes

- [Project landscape](_docs/projects/README.md)
- [surfbuddy-server](_docs/projects/surfbuddy-server/README.md)
- [surfbuddy-v3](_docs/projects/surfbuddy-v3/README.md)
- [surfbuddy-vue](_docs/projects/surfbuddy-vue/README.md)

### Architecture

- [Architecture overview](_docs/architecture/README.md)

### Feature documentation

- [Feature documentation](_docs/features/README.md)
