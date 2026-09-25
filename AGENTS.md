# Yxy specification — agent and contributor instructions

This file is the single source of instructions for this repository; `CLAUDE.md`
only imports it. If the parent directory has its own `AGENTS.md` (a maintainer
workspace), read it too.

- This repository says how the language **should** behave. Implementation
  status lives in the compiler repository (`../yxy`,
  `docs/implementation/STATUS.md`). Never edit the specification to match the
  compiler without a recorded decision.
- Changing a rule means: a new or updated file in `decisions/` (context,
  decision, alternatives, what could change it), then the text in `syntax.md`
  or `semantics.md`. The **state** of every decision (author, experimental,
  awaiting the author) is controlled by the register in the control repository
  `yxy-develop/plans` (`decisions/README.md`); add or update its row.
  Decision 0001 records the author's own decisions; do not reopen them unless
  the author asks.
- Keep stable rule identifiers (`[CELL-4]`); never reuse an identifier for a
  different rule.
- Mark proposals as experimental. Do not present a proposal as accepted, or a
  written rule as proven.
- English for the text; the README also has a Portuguese section. Commits in
  English, authored by the maintainer identity configured in Git, with no AI
  attribution. No push to public remotes or visibility change without the
  maintainer's authorization.
