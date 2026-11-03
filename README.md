# readable-ansible

A Claude Code skill that sets a strict, readable style for Ansible projects.

## What it covers

- Project layout with an in-repo collection and FQCNs everywhere.
- Tasks:
  modules over shell, blocks for grouping, named loop variables, read-only annotations,
  failures as data.
- Roles as typed functions with validated parameters, no `defaults/main.yml`,
  and a caller-named result fact.
- A variable naming scheme that passes `ansible-lint` at the `production` profile.
- Readable Jinja: whitespace-controlled blocks, no long ternaries, literal dicts in YAML.
- Handling hosts without Python, choosing targets with `group_by`,
  and the lint and runtime gotchas that cost time.
- Testing strategy:
  unit tests for plugins, playbook tests with fake tools, Molecule integration with mock data.

## Install

Copy or symlink this directory into `~/.claude/skills/`:

```
~/.claude/skills/readable-ansible/
├── SKILL.md
└── README.md
```

## Use

Invoke the skill in Claude Code:

```
/readable-ansible
/readable-ansible new project
/readable-ansible review
/readable-ansible role
/readable-ansible playbook
```

The skill loads the full style guide into context so Claude follows it when writing or reviewing
Ansible code.

## Companion skills

This skill loads two optional companion skills when they are installed:

- **`clause-wrap`** — wraps prose at clause boundaries (~100 columns).
- **`small-commits`** — splits uncommitted work into a reviewable commit history.

Both are optional. Without them the skill falls back to its own built-in rules.

## Lint target

All rules aim for `ansible-lint` at the `production` profile with an empty `skip_list`.
