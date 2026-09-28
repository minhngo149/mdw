# MDW

**Minh Daily Works**: an engineering skill library for [Claude Code](https://claude.com/claude-code).

MDW skills read a real repository and turn what the code actually does into engineering
knowledge that people and agents can reuse:

```text
Source code  ->  Engineering knowledge  ->  Reusable text artifacts (.md, .html)
```

## Skills

| Skill | Use it to |
|---|---|
| [`distributed-flow`](skills/distributed-flow/SKILL.md) | Reconstruct how one feature, endpoint, job, event, or business flow really executes: entry point, sync and async hops, service boundaries, persistence, transactions, state changes, and failure paths. Every claim is labeled `CONFIRMED`, `INFERRED`, or `UNKNOWN` and backed by file and symbol evidence. |

## Principles

```text
Source Code    >  Documentation  >  Assumption
Accuracy       >  Completeness
Evidence       >  Guess
Readable       >  Exhaustive
Reusable Text  >  Proprietary Diagram Format
```

- **Evidence labels.** `CONFIRMED` means read in source code. `INFERRED` means strongly implied,
  with the reasoning shown. `UNKNOWN` means the repository cannot tell. An inference is never
  presented as a fact.
- **Code over docs.** READMEs and comments are leads to verify. When they disagree with the code,
  the code wins and the disagreement is reported.

## Output

| Format | Role |
|---|---|
| `.md` | Source of truth. Readable on its own, diffable, and consumable by other agents and MDW skills. |
| `.html` | Presentation of the same knowledge. One self-contained file with inline CSS, no framework, no build step, no network requests. It opens directly in a browser and works with JavaScript disabled. |

MDW does **not** depend on Mermaid, draw.io, SVG, image formats, or any proprietary diagram
format. Flows are drawn as plain text in Markdown and with HTML/CSS in HTML.

## Install

**As a Claude Code plugin** (recommended). This repository is its own plugin marketplace:

```bash
claude plugin marketplace add minhngo149/mdw
claude plugin install mdw@mdw
```

Or inside Claude Code: `/plugin marketplace add minhngo149/mdw`, then `/plugin install mdw@mdw`.
Restart Claude Code afterwards. The skill is then available as `mdw:distributed-flow`.

**As a plain skill.** Claude Code also loads skills from `~/.claude/skills/<name>/` (all
projects) or `<project>/.claude/skills/<name>/` (one project):

```bash
git clone https://github.com/minhngo149/mdw.git ~/mdw
mkdir -p ~/.claude/skills
ln -s ~/mdw/skills/distributed-flow ~/.claude/skills/distributed-flow
```

## Use

Open Claude Code in the repository you want to analyze and ask about one flow:

```text
/distributed-flow POST /orders
/distributed-flow the order.created consumer
How does the nightly invoice job work end to end?
```

Claude also picks up the skill on its own when you ask how a specific endpoint, job, consumer,
or business operation works. Artifacts are written to `mdw/distributed-flow/<flow>.md` and
`mdw/distributed-flow/<flow>.html` in the analyzed repository, unless you name another location.

## Layout

```text
.
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── .claude-plugin/
│   ├── marketplace.json                lets `claude plugin marketplace add` find the plugin
│   └── plugin.json                     plugin manifest (name, version, license)
└── skills/
    └── distributed-flow/
        ├── SKILL.md                    workflow, evidence rules, output contract
        └── references/                 loaded on demand, one topic each
            ├── repository-analysis.md
            ├── service-boundary.md
            ├── sync-flow.md
            ├── async-flow.md
            ├── failure-flow.md
            ├── transaction-flow.md
            └── text-flow-format.md
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[MIT](LICENSE)
