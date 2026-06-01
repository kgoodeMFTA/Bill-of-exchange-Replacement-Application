# Contributing

Thanks for your interest in contributing.

## Quick start

1. Fork the repo and create a feature branch off `main`:
   ```bash
   git checkout -b feat/short-description
   ```
2. Make your change. Keep commits small and focused.
3. Run tests locally before pushing.
4. Open a pull request against `main` with a clear description, screenshots if UI, and a link to the issue it addresses.

## Commit message style

We follow [Conventional Commits](https://www.conventionalcommits.org/):

- `feat: add policy-explainer caching`
- `fix: correct LAE per-claim formula in dashboard`
- `docs: clarify Fabric network bring-up steps`
- `chore: bump fastapi to 0.115`

## Code style

- **Python**: ruff + black; type hints required on public functions
- **TypeScript**: eslint + prettier; strict mode on
- **Go**: gofmt + golangci-lint
- **Markdown**: 100-char soft wrap, ATX headers

## Pull request checklist

- [ ] Tests pass locally
- [ ] New code has unit tests (target ≥ 80% coverage on touched files)
- [ ] No secrets, API keys, PII, or production data committed
- [ ] README / docs updated if behavior changed
- [ ] Single, focused change — split unrelated changes into separate PRs

## Security

Do not file public issues for security vulnerabilities. See [SECURITY.md](./SECURITY.md).

## Reporting bugs

Use the bug-report issue template. Include: environment, steps to reproduce, expected vs. actual behavior, and logs if available.

## Questions

Open a Discussion or email goodeplays@gmail.com.
