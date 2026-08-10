## Type of change

- [ ] `feat`: new feature
- [ ] `fix`: bug fix
- [ ] `refactor`: code change with no behaviour difference
- [ ] `perf`: performance improvement
- [ ] `docs`: documentation only
- [ ] `test`: adding or correcting tests
- [ ] `build`: build system or dependency change
- [ ] `ci`: CI pipeline change
- [ ] `chore`: maintenance (version bump, cleanup)
- [ ] `security`: security fix

## Description

<!-- What does this PR do and why? -->

## Pre-merge checklist

- [ ] **Format clean.** `dotnet format --verify-no-changes --severity error` exits 0 locally.
- [ ] **Build clean.** `dotnet build -c Release` exits 0. Warnings-as-errors comes from `TreatWarningsAsErrors` in `Directory.Build.props`; do not pass `-warnaserror` by hand.
- [ ] **Tests green.** `dotnet test -c Release` passes with no failures.
- [ ] **No em dash in Markdown or docs.** The character U+2014 does not appear in any `*.md` or `docs/**` file touched by this PR. CI ("prose" job) enforces this automatically. Source code, resource files, and commit messages are reviewer-enforced, not CI-enforced.
      Check with `git diff main... | grep -P '\x{2014}'` if unsure.
- [ ] **No secrets.** API keys, tokens, and credentials are not committed.
- [ ] **README updated** (if this change affects user-visible behaviour or setup steps).
- [ ] **CHANGELOG updated** under `[Unreleased]` with the appropriate subsection
      (`Added`, `Changed`, `Fixed`, `Removed`, `Security`).
