# Contributing

traymirror is a Windows 11 utility. You need a **Windows 11 machine with at least two
monitors** to build meaningfully and test. There is no cross-platform build path and no
plans for one: the project depends on DWM thumbnail APIs that exist only on Windows.

## Prerequisites

| Requirement | Detail |
|---|---|
| OS | Windows 11 (build 22000 or later) |
| Monitors | Two or more, mixed DPI scaling recommended for real testing |
| .NET SDK | 10.0.302, pinned in `global.json` (`rollForward: latestFeature`) |

Install the exact SDK version before cloning, or let Visual Studio 2022 17.12+ pick it up
from `global.json` automatically. The pin exists so a routine SDK update cannot silently
change language semantics or analyser behaviour.

## Build and test

`traymirror.sln` and the three projects are documented intent and have not landed yet, so there
is nothing to compile at this commit. These are the commands once they exist, and the CI build
and format jobs stay dormant until `traymirror.sln` appears.

```
dotnet restore
dotnet build -c Release
dotnet test -c Release
```

`dotnet build` runs with `TreatWarningsAsErrors=true` by default via `Directory.Build.props`.
A warning is a build failure. Fix it before pushing.

To check formatting without changing any files:

```
dotnet format --verify-no-changes --severity error
```

To apply formatting fixes locally:

```
dotnet format
```

## PR gates (all must pass)

All three CI jobs run on `windows-latest`.

1. **Prose.** The `prose` job greps for U+2014 in `*.md` and `docs/**`. Source code,
   resource files, and commit messages are reviewer-enforced, not CI-enforced.
2. **Format.** `dotnet format --verify-no-changes --severity error` must exit 0.
3. **Build and test.** `dotnet build -c Release` must succeed. Warnings are errors via
   `TreatWarningsAsErrors` in `Directory.Build.props`, which covers all Roslyn analyser
   diagnostics. Tests must pass with no failures or skips.

## Commit convention

This project uses [Conventional Commits](https://www.conventionalcommits.org/).

```
type(scope): lowercase imperative summary
```

- **Type** (required, lowercase): `feat`, `fix`, `docs`, `refactor`, `perf`, `test`,
  `build`, `ci`, `chore`, `security`.
- **Scope** (optional, rare): confine to one subsystem from the fixed list: `dwm`, `tray`,
  `monitors`, `dpi`, `config`, `ui`, `interop`, `ci`, `release`.
- **Summary**: lowercase, imperative, no trailing period, 72-character hard limit.
- **Body**: wrap at 72 columns. State what was tested and why the decision was made. Close
  with a `Verified:` line describing the evidence.
- **Breaking changes**: `type!: summary` plus a `BREAKING CHANGE:` footer paragraph.

Examples:

```
feat(dwm): register thumbnail per secondary monitor
fix(dpi): recompute source rect on WM_DPICHANGED
docs: update release verification steps
```

## Writing style for docs and Markdown

- Australian spelling in all prose: behaviour, colour, initialise, centre.
  Code identifiers and .NET API names stay US-spelled (`Color`, `Initialize`).
- Sentence case headings. Second person, present tense.
- **The em dash character U+2014 is forbidden in all tracked files** (house rule).
  CI enforces this in `*.md` and `docs/**` via the `prose` job. Source code and commit
  messages are reviewer-enforced. If a sentence seems to need one, restructure it with
  a colon, commas, or parentheses.
- No en dash in prose. En dash is permitted only inside numeric ranges in tables.
- No marketing superlatives. Be frank about failure modes.

## Manual verification checklist (required before release, not automated)

- [ ] Three monitors at mixed DPI (for example 100 / 125 / 150 per cent)
- [ ] Monitor hot-plug (connect and disconnect during a session)
- [ ] RDP session (expected: tray mirrors are suppressed or handle session switch cleanly)
- [ ] Lock and unlock the workstation
- [ ] App launched both elevated and non-elevated
