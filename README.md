<img src="assets/icon.svg" width="64" height="64" alt="">

# skills

Agent skills, installable with [`npx skills`](https://github.com/vercel-labs/skills).

## Install

```bash
# pick from the skills in this repo
npx skills add axelgalea/skills

# or install one directly
npx skills add axelgalea/skills --skill effect
```

## Skills

| Skill | Description |
| --- | --- |
| [`effect`](skills/effect/SKILL.md) | Coding conventions and API patterns for Effect v4, including services, schemas, layers, RPC, tests, HTTP, and React state with Effect Atom. Adapted from Kit Langton's work, see [NOTICE](skills/effect/NOTICE.md). |
| [`effect-project`](skills/effect-project/SKILL.md) | Project structure, package boundaries, feature folders, application wiring, and development workflow for CLI, server-only, and full-stack Effect projects. Invoke explicitly with `$effect-project`. Use with `effect`. |

Install both skills to use the project conventions. `effect-project` reads the
coding and API references in `effect`. Install `effect` alone if you only want
the coding guidance.

Created in [T3 Code](https://t3.codes).
