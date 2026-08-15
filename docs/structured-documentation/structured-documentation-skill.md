# structured-documentation skill

| | |
|---|---|
| **Date** | 2026-08-15 |
| **Status** | Resolved |
| **Confidentiality** | Internal |

## Summary
Built a skill for writing short, structured Findings/Feedback/Conclusion documentation on a workspace feature, with tracking for open items and a rule against committing secrets to git.

## Findings

### 1. Initial version was flat prose
The first draft used one paragraph per section (Findings, Feedback, Conclusion), saved directly to docs/*.md. It covered the ask but wasn't scannable: there was no way to jump to a specific finding without reading the whole file.

### 2. Restructured for outline navigation
I rebuilt the template around a metadata table (Date, Status), a one-line Summary, and numbered per-finding subsections (### 1., ### 2., ...). These show up in a markdown outline, like VS Code's Outline panel, and let Feedback and Conclusion point at a specific finding by number instead of vaguely.

### 3. Output moved into a namespaced subfolder
Docs now save to docs/structured-documentation/ instead of loose files in docs/. A general workspace's docs/ folder usually holds other project documentation, and flat files there would mix into it.

### 4. Added Status and an Outstanding checklist
Status (Open, Pending, Needs decision, Resolved) tracks the doc's overall state. A separate Outstanding section holds unresolved items as checkboxes, referencing the finding they relate to, and gets deleted once everything on it is checked off.

### 5. Added a Confidentiality field and redaction rules
A committed doc stays in git history for good. The skill now requires a Confidentiality field (Public, Internal, Confidential) and forbids pasting actual secrets, credentials, or PII into a doc; it describes where a secret exists instead of writing the value.

### 6. Correlation moved to a README.md, not an index.md
One file, docs/structured-documentation/README.md, lists every doc with its Status and Outstanding count. It's named README.md so a new session opens it by default on landing in the folder and picks up the folder's conventions without being told.

### 7. Skill renamed from findings-report to structured-documentation
"Findings" originally implied something discovered by search. The skill also covers work that was built or decided, not just investigated, so I renamed it and updated the folder path to match.

### 8. Explicit Check step added to the workflow
The workflow now opens with a Check step: read docs/structured-documentation/README.md first, every run, before writing anything. A fresh session has no memory of past docs, so without this step it could create a duplicate.

## Feedback
- **On #1:** The flat version technically answered the request but missed the real goal, letting a reader find something quickly, so it needed a rewrite rather than a tweak.
- **On #2:** The most useful change of the session. Numbered subsections turned the doc from something read top to bottom into something you navigate.
- **On #3:** The right call. This skill doesn't own a general workspace's docs/ folder, so it shouldn't drop loose files into it.
- **On #4 and #5:** Both came from explicit asks. The no-secrets rule matters more than it might look: docs/ files are permanent once committed, so it isn't optional polish.
- **On #6:** A small change with a real payoff. A README nobody opens is dead weight.
- **On #7:** Took a few rounds to land on, since the early feedback was hard to parse, but the final name is more accurate than the original.
- **On #8:** A necessary follow-up to #6. An instruction-less README just sits there unread.

## Conclusion
- **Decision:** Finalized as structured-documentation, committed to the sharpps-work plugin (commit 80e99b2).
- **Next steps:** none. This doc is the skill's first real output, and doubles as its own test.
