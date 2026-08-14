# Implementation notes

## Scope

Implemented the five student-owned harness layers only. The frozen
`arena/` package, `parse_output`, and the agent step limit were not changed.

## Decisions and trade-offs

- The middleware installation order remains the one already used by
  `scripts/run_practice.py`: injection guard, critic, citation checker,
  budget policy, then retry. This makes citation correction run before the
  critic during the reverse `after_agent` pass, and leaves the injection
  answer sweep until last.
- Citation support is checked per document line, not against an entire
  document body. A candidate source must also have been returned in full in
  `ctx.observed_text`; a search snippet or truncated fetch is intentionally
  not considered sufficient evidence.
- Claims are never rewritten. The critic may delete an unsupported claim or
  split a fused claim into substrings already written by the model. The
  citation checker changes only `doc_id`. This preserves scorer provenance.
- Fusion detection supports the documented `" và "` join plus a few common
  Vietnamese contrast joins. It accepts a split only when both parts are
  supported by distinct, fully observed documents.
- The injection guard removes all delimited untrusted blocks and also handles
  an unterminated opening marker, which occurs on truncated tool output. The
  final safety sweep edits `answer` only; editing claim text would lose
  grounding provenance.
- Both budget policy and retry reserve one call for `submit`. Budget policy
  blocks a new tool action once the reserve is reached; retry separately
  checks the same threshold before every additional attempt because it is
  nested inside the budget wrapper.
- `retry_attempts` records the attempt count for the most recent tool action
  in `ctx.state`, which is useful for diagnostics without altering the trace.

## Validation environment

The workspace virtual environment uses Python 3.12. Its ambient pytest
plugin discovery can import incompatible system ROS plugins, so validation
uses `PYTEST_DISABLE_PLUGIN_AUTOLOAD=1` with `.venv/bin/python`.

## Validation results

- `PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 .venv/bin/python -m pytest -q`:
  **757 passed**.
- `.venv/bin/python scripts/run_practice.py`: **81.71 / 100** across the
  nine public briefs, matching the complete-stack reference score documented
  in the README. The two low grounding synthesis-style public briefs are an
  expected retrieval-depth limitation of the baseline agent, not a claim
  provenance or safety regression.
