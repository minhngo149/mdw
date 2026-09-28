# Contributing to MDW

MDW is a small library of Claude Code skills that turn source code into reusable engineering
knowledge. Keep it small: a new skill has to earn its place.

## Every skill must

1. **Solve one clear engineering problem.** State it in the skill's `description`.
2. **Prefer repository evidence.** Source code over documentation over assumption.
3. **Distinguish `CONFIRMED` / `INFERRED` / `UNKNOWN`.** Use the definitions in
   [`skills/distributed-flow/SKILL.md`](skills/distributed-flow/SKILL.md#evidence-labels);
   do not redefine them.
4. **Produce reusable text.** Output other people, agents, and MDW skills can read without
   special tools.
5. **Avoid unnecessary tooling.** No CLIs, packages, build steps, renderers, or scripts unless
   the skill cannot work without them.
6. **Keep Markdown as the source of truth.**
7. **Provide HTML only as presentation.** It shows the same knowledge as the Markdown, in one
   self-contained file.

Output is limited to `.md` and `.html`. Do not produce Mermaid, `.mmd`, `.drawio`, `.png`,
`.jpg`, `.svg`, or any other diagram format.

## Skill layout

```text
skills/<skill-name>/
├── SKILL.md          frontmatter + workflow + rules
└── references/       optional; one topic per file, loaded when a step needs it
```

- `name`: lowercase letters, numbers, and hyphens.
- `description`: starts with "Use when ...", written in the third person, and lists the
  triggering situations only. Do not summarize the workflow; agents may follow the summary
  instead of reading the skill.
- Link every reference directly from `SKILL.md`, one level deep, and say when to read it.
- References extend `SKILL.md`. They must not restate it.

## Test before you merge

A skill is untested until you have watched it fail without the change and pass with it.

1. Pick a repository where you know the true answer, or build a small fixture that contains
   traps: documentation that contradicts the code, code that exists but is never wired in, a
   package named like a service, an error that is logged and swallowed.
2. Run the task in Claude Code **without** the skill (or without your change). Note what goes
   wrong.
3. Run it **with** the skill. Check every trap, and check that every `CONFIRMED` claim points
   at code that really says it.
4. Describe the scenario and the before/after results in the pull request.

## Pull requests

- One skill, or one focused change, per pull request.
- Say which failure the change fixes.
- Keep examples in skills generic. Do not copy a real company's code or names.
