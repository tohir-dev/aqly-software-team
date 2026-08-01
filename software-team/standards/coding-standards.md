# Coding standards (language-agnostic)

The architect picks the stack; these apply whatever it is.

## Structure
- Small, single-responsibility modules. A file does one thing.
- Clear public surface; hide internals. Dependencies point inward (logic doesn't depend on framework details).
- Name by intent, not type. `chargeCard`, not `doStuff`. Match the surrounding code's conventions.

## Correctness
- Validate input at boundaries; fail fast with clear errors.
- No swallowed exceptions. Handle or propagate — never silently ignore.
- Make illegal states unrepresentable where the language allows (types, enums, constructors).
- Pure logic separated from I/O so it's testable.

## Security (baseline, always)
- No secrets in code or VCS — env/secret-store only. Provide `.env.example`.
- Parameterized queries; never string-concatenate untrusted input into SQL/shell/HTML.
- Least privilege for any token, file, or network access.
- Don't log secrets or PII.

## Style
- Use the stack's standard formatter and linter; zero lint errors before handing off.
- Comments explain *why*, not *what*. Match the codebase's existing comment density.
- Prefer clarity over cleverness. The next agent (or human) must read it cold.

## Dependencies
- Prefer the standard library; add a dependency only when it clearly pays for itself.
- Pin versions / commit a lockfile. Run a dependency vulnerability scan (security agent gate).
- Avoid abandoned or single-maintainer-critical packages where a maintained alternative exists.

## Tests
- Tests live beside the code or in a conventional test dir for the stack.
- Each test asserts one behaviour; names describe the behaviour.
- No network/time/randomness flakiness — inject or fake them.
- No hardcoded absolute date/time literals in fixtures that compute against "today"/"now" — derive
  timestamps relative to a current reference (or an injected clock) so a day-boundary rollover cannot
  turn a passing test into a spurious failure a calendar day later.
