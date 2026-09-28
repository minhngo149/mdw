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
| [`sequence-flow`](skills/sequence-flow/SKILL.md) | Reconstruct the actual execution sequence of one operation at function level: entry point, call chain, branches, loops, error paths, goroutines and other async boundaries, transactions, and the return path, in the order the code runs them. Uses the same evidence labels. |
| [`security`](skills/security/SKILL.md) | Review a repository for security risks supported by its code, config, schema, and infrastructure: authentication, authorization (IDOR/BOLA, privilege escalation, tenant isolation), injection (SQL, command, SSRF, path traversal), file uploads, sessions and tokens, secrets, cryptography, data exposure, dependencies, configuration, and business-logic and race-condition flaws. Every finding carries evidence, a safe verification, and a `CONFIRMED` / `INFERRED` / `UNKNOWN` label; secrets are redacted and CVEs are never invented. |

The skills answer different questions and compose:

```text
distributed-flow   "Where does the system communicate?"   services, brokers, stores, delivery, consistency
sequence-flow      "What executes, in what order?"         functions, branches, loops, returns, concurrency
security           "Where can trust or authorization fail?" identity, authz, injection, secrets, data boundaries
```

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
Restart Claude Code afterwards. The skills are then available as `mdw:distributed-flow` and
`mdw:sequence-flow`.

**As a plain skill.** Claude Code also loads skills from `~/.claude/skills/<name>/` (all
projects) or `<project>/.claude/skills/<name>/` (one project):

```bash
git clone https://github.com/minhngo149/mdw.git ~/mdw
mkdir -p ~/.claude/skills
ln -s ~/mdw/skills/distributed-flow ~/.claude/skills/distributed-flow
ln -s ~/mdw/skills/sequence-flow ~/.claude/skills/sequence-flow
```

`sequence-flow` points to some `distributed-flow` references for cross-process depth, so install
both.

## Use

Open Claude Code in the repository you want to analyze and ask about one flow:

```text
/distributed-flow POST /orders
/distributed-flow the order.created consumer
How does the nightly invoice job work end to end?

/sequence-flow POST /orders
/sequence-flow trace CreateOrder()
Show me what executes, in order, when payment succeeds.
```

Claude also picks up the skills on their own when you ask how a specific endpoint, job,
consumer, or business operation works (`distributed-flow`), or for its call chain or execution
order (`sequence-flow`). Artifacts are written to the analyzed repository, unless you name
another location:

| Skill | Artifacts |
|---|---|
| `distributed-flow` | `mdw/distributed-flow/<flow>.md`, `mdw/distributed-flow/<flow>.html` |
| `sequence-flow` | `mdw/sequence-flow/<operation>-sequence.md`, `mdw/sequence-flow/<operation>-sequence.html` |

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
    ├── distributed-flow/               "Where does the system communicate?"
    │   ├── SKILL.md                    workflow, evidence rules, output contract
    │   └── references/                 loaded on demand, one topic each
    │       ├── repository-analysis.md
    │       ├── service-boundary.md
    │       ├── sync-flow.md
    │       ├── async-flow.md
    │       ├── failure-flow.md
    │       ├── transaction-flow.md
    │       └── text-flow-format.md
    └── sequence-flow/                  "What executes, in what order?"
        ├── SKILL.md                    workflow, evidence rules, output contract
        └── references/                 loaded on demand, one topic each
            ├── execution-tracing.md
            ├── call-chain.md
            ├── control-flow.md
            ├── branching.md
            ├── loops.md
            ├── async-boundary.md
            └── sequence-format.md
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[MIT](LICENSE)
