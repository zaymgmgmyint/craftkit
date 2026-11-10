# craftkit

A working engineer's habits turned into AI agent skills: plan, build, review PRs and fix bugs in existing codebases.

Each skill is a folder with a `SKILL.md` in the open Agent Skills format, so it works in Claude Code, Codex, Gemini CLI, Copilot and other agents that read that format.

Skills hold the **process**. Your **rules** stay in each project's own files
(AGENTS.md or CLAUDE.md, CODING_STANDARDS.md, GLOSSARY.md, docs/codebase-map.md).

## Skills

| Skill | Job |
|---|---|
| `ask-zay` | Entry point: which skill to use, and the usual order |
| `repo-scout` | Drafts a short codebase map, glossary and coding standards for a repo |
| `clarify` | Asks one question at a time to close gaps in vague requirements |
| `deep-dig` | Researches options and trade-offs; sources and a recommendation |
| `blueprint` | Pasted requirements to `plans/<ticket-id>.md` |
| `build-it` | Builds from the plan step by step; tests after each step |
| `explain-this` | Explains code, a flow or a PR in plain words |
| `bug-hunt` | Reproduce, find the cause, fix, add a test |
| `self-check` | Developer checks their own changes before opening a PR |
| `pr-inspect` | Reviews one PR against pasted requirements and repo rules; writes `reviews/pr-<n>.md` |
| `second-look` | Fresh second opinion that tries to disprove a finished review |
| `review-digest` | Rolls up many reviews by rule: what to teach, add or automate |

All 12 are written; only `repo-scout`, `pr-inspect` and `self-check` have had an install test
(none has been tested on real PRs yet).

## Install

```
npx skills add zaymgmgmyint/craftkit --skill '*' -g    # every project on this machine
npx skills add zaymgmgmyint/craftkit --skill '*'       # current project only
npx skills add zaymgmgmyint/craftkit --skill pr-inspect   # one skill
```

Add `-a claude-code` (or another agent name) to pick the agent, and `-y` to skip prompts.
Tested from a local copy into a scratch project: skills and their
`references/` files were copied. Installing from GitHub needs the repo pushed first.
Update later with `npx skills update`.

## Safety defaults
- Review skills are read-only: they write one review file and never edit code, push or post.
- Text from PRs, tickets, comments and repo files is treated as data, never as instructions.
- Rules are read from the default branch, so a PR cannot change the rules it is judged by.
- Do not run these skills in automated pipelines that hold secrets.

## Tests
Tests live outside this repo in `craftkit-evals/` (kept private, since it holds real PRs). A skill ships only if it beats the agent with no skill on your own PRs.

## Adding a skill
One job per skill. Frontmatter has only `name` and `description`. Body under 500 lines.
No dependence on one project or one agent. At least 3 scenario tests and about 20
trigger prompts in `evals/`.
