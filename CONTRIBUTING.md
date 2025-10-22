# Contributing

Thank you for considering a contribution! This project is heavily **blueprint-driven** to enable AI-assisted development without ambiguity. Please follow these guidelines.

## Code of Conduct
- Be respectful and constructive.
- Assume good intent.
- Report security/privacy issues privately.

## Development Workflow
1. Fork and clone.
2. Create a feature branch: `feat/<short-scope>` or `fix/<short-scope>`.
3. Run `pre-commit` (optional), `ruff`, `black`, `mypy`, and `pytest` locally.
4. Commit using conventional commits (see below).
5. Open a PR with a focused scope and checklist filled.

## Conventional Commits
- `feat:` new feature
- `fix:` bug fix
- `docs:` documentation only changes
- `refactor:` code change that neither fixes a bug nor adds a feature
- `test:` add or refactor tests
- `build:` build system or external dependencies changes
- `ci:` CI pipeline changes
- `chore:` maintenance

## DCO / CLA
If required by the hosting platform, sign-off your commits (DCO) or complete the CLA.

## Issue Labels
- `good first issue`, `help wanted`, `design`, `blocked`, `needs-snapshot`, `selectors`, `flaky`, `security`.

## PR Checklist
- [ ] Scope limited to one problem.
- [ ] Docs updated (USER_GUIDE/DEVELOPER_GUIDE).
- [ ] Tests adapted or added.
- [ ] Lint and type checks pass.
- [ ] No secrets or credentials committed.

## Security & Ethics
- Never store credentials in the repo.
- Respect website ToS and legal constraints.
- Consider adding randomized delays and rate limiting.
