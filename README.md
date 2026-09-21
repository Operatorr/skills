<p>
  <img alt="AI Skills for Real Engineers" src="./assets/skills-banner.png" width="720">
</p>

# AI Skills for Real Engineers

[![skills.sh](https://skills.sh/b/Operatorr/skills)](https://skills.sh/Operatorr/skills)

A personal collection of small, composable agent skills I use for real engineering work.

## Quickstart

Install from this public GitHub repo with the Skills CLI:

```bash
npx skills@latest add Operatorr/skills
```

Then pick the skills you want to install into your coding agent.

> This does **not** require publishing this repo to npm. The `skills` npm package is the installer; `Operatorr/skills` points it at this GitHub repository.

## Structure

```text
.
├── .claude-plugin/        # optional plugin/config files if needed later
├── .out-of-scope/         # deprecated or parked skills/docs
├── docs/adr/              # architecture decision records
├── scripts/               # repo maintenance scripts
├── skills/
│   ├── engineering/       # code and engineering workflow skills
│   │   ├── visual-review/
│   │   │   └── SKILL.md
│   │   ├── code-review/
│   │   │   ├── SKILL.md
│   │   │   ├── REFERENCE.md
│   │   │   └── PROJECT_REVIEW_TEMPLATE.md
│   │   ├── deep-review/
│   │   │   ├── SKILL.md
│   │   │   ├── REFERENCE.md
│   │   │   ├── SUBAGENT_PROMPT.md
│   │   │   └── TOOL_BATTERY.md
│   │   ├── git-commit/
│   │   │   └── SKILL.md
│   │   └── git-pr/
│   │       └── SKILL.md
│   ├── productivity/      # general workflow skills
│   │   └── rewrite/
│   │       └── SKILL.md
│   ├── security/          # authorized security assessments
│   │   └── penetration-testing/
│   │       ├── SKILL.md
│   │       └── references/
│   └── misc/              # occasional-use skills
├── CLAUDE.md              # optional agent instructions for this repo
├── CONTEXT.md             # shared vocabulary / repo context
└── README.md
```

Each skill should be self-contained and live at:

```text
skills/<category>/<skill-name>/SKILL.md
```

## Skills

### Engineering

- **[visual-review](./skills/engineering/visual-review/SKILL.md)** — inspect a PR's running UI through computer use for clipping, wrapping, uneven controls, spacing, responsive layout issues, and UX improvements. Reports screenshot evidence, concrete fixes, and optional redesign recommendations. Invoke with `/visual-review <PR URL>`.
- **[code-review](./skills/engineering/code-review/SKILL.md)** — thorough single-pass reviews for GitHub PRs and local branch changes that aim to catch every Critical, High, and Medium issue while skipping Low/Nit noise. Built for when you want a careful senior-engineer review without the length of exhaustive CodeRabbit-style coverage.
- **[deep-review](./skills/engineering/deep-review/SKILL.md)** — the exhaustive counterpart to code-review: a maximally thorough, CodeRabbit-style review that fans out one sub-agent per changed file, runs every available linter/SAST/secret scanner, and reports every issue down to nitpicks. Optimizes for coverage over signal-to-noise — closes most of the gap with CodeRabbit when you want to find everything.
- **[git-commit](./skills/engineering/git-commit/SKILL.md)** — stage all changes and create a git commit with an auto-generated, convention-matched message.
- **[git-pr](./skills/engineering/git-pr/SKILL.md)** — branch off master/develop when needed, commit all changes, push, and open a PR automatically.

### Productivity

- **[rewrite](./skills/productivity/rewrite/SKILL.md)** — rewrite or generate copy in a plain, human voice that avoids AI tells: inflated significance, stock vocabulary, trailing "-ing" why-it-matters clauses, "not just X but Y" parallelism, autopilot rule-of-three, and chatbot filler.

### Security

- **[penetration-testing](./skills/security/penetration-testing/SKILL.md)** — authorized security assessments of owned infrastructure (traditional hosts and managed/serverless/BaaS), gated on standing org authorization plus an asset inventory, validated non-destructively, producing an OWASP/CWE/CVSS-mapped remediation report.

  **Standing authorization (do this once, before invoking).** Copy the templates from `skills/security/penetration-testing/references/` into the **engagement workspace** (the directory you invoke the skill from), fill them, and save as `org-authorization.md` and `asset-inventory.md`. Same filenames under `security/` or `.security/` also work. Keep filled copies out of this skills repo.

  ```bash
  cp skills/security/penetration-testing/references/org-authorization.template.md /path/to/engagements/org-authorization.md
  cp skills/security/penetration-testing/references/asset-inventory.template.md /path/to/engagements/asset-inventory.md
  ```

  When both files are present and filled, the skill skips the per-site ownership quiz and starts against inventory hosts — no DNS / Vercel / Cloudflare login required to begin. Hosts not on the inventory stay out of the run until you add them.

  Usage:

  ```text
  /penetration-testing assess checkout.example.com api.example.com
  ```

### Misc

None yet.

## Maintenance

List skills locally:

```bash
./scripts/list-skills
```

## Adding a skill

1. Create a directory at `skills/<category>/<skill-name>/`. Categories are `engineering`, `productivity`, `security`, and `misc`.
2. Add `SKILL.md` with the skill instructions.
3. Add any supporting examples, scripts, or reference docs inside the same skill directory.
4. Update the reference list in this README, the category `README.md` under `skills/<category>/`, and the path in `.claude-plugin/plugin.json`.

## Demo

![Code review output demo](./assets/code_review_comment_sample.jpeg)
