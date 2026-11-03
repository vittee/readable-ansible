---
name: readable-ansible
description: "Use when starting a new Ansible playbook, role or collection project, or when writing or reviewing Ansible YAML. Sets a strict, readable style: modules not shell, tasks grouped in blocks, roles that behave like typed functions, a naming scheme for variables that passes ansible-lint's production profile, readable Jinja, and no skipped-task noise. Also the lint and runtime gotchas that cost time."
user-invocable: true
argument-hint: "[new project | review | role | playbook]"
---

# Readable Ansible

A style for Ansible code that a reviewer can read line by line.
It was built up in review on a real project, so each rule has a reason, and the reasons are given.

**Existing code comes first.**
In a project that already has its own style, follow that and mention the difference.
These rules are for new work and for reviews. **The lint is the judge.**
Aim for `ansible-lint` at the `production` profile with an **empty** `skip_list`.
Never skip a rule to make a warning go away, fix the cause.

## Companion skills

Load these skills when they are available:

- **`clause-wrap`** — for wrapping prose in comments, docs, and Markdown at clause ends.
  https://github.com/vittee/clause-wrap
- **`small-commits`** — for splitting uncommitted work into a clean, reviewable commit history.
  https://github.com/vittee/small-commits

If either skill is not installed,
follow the rules in this file (the Comments and docs section covers the wrapping basics).

## Start a new project

```
ansible.cfg
.ansible-lint                       # profile: production, skip_list: []
playbook.yml                        # plays only: checks, choose targets, one play per job
inventory.example.yml
collections/ansible_collections/<namespace>/<name>/     # next to the playbook, so AWX finds it
  galaxy.yml  CHANGELOG.md  meta/runtime.yml
  roles/<role>/tasks/main.yml
  plugins/{modules,filter,module_utils}/                # only when no builtin module fits
tests/unit/  tests/playbooks/       # pytest: plugins as plain Python, the playbook against fake hosts
```

- `ansible.cfg` sets `collections_path = ./collections:...` and `retry_files_enabled = False`.
  Run from the project root, since Ansible reads `./ansible.cfg` only from the current directory.
- Keep code in an in-repo collection (`galaxy.yml` stays, ansible-lint needs it,
  add a CHANGELOG entry for each version bump).
  Use FQCNs everywhere: `ansible.builtin.*`, `<namespace>.<name>.*`.
  The lint also wants a `tags:`
  list in `galaxy.yml` (one of its allowed tags, such as `tools`)
  and a supported `requires_ansible` in `meta/runtime.yml` (for example `">=2.19.0"`).
- **Targets need nothing beyond the Python Ansible already needs.**
  Roles gather raw output and parsing happens on the controller, in a plugin that has unit tests.
- Say which ansible-core version you target.
  From 2.19 conditionals must be real booleans (`x is not none`, `flag | bool`),
  never a bare string or number.

## Tasks

- **Modules, not shell.** No `sh -c` snippets.
  Use builtin modules (`stat`, `slurp`, `setup` with `gather_subset`, `find`, `systemd_service`).
  When none fits, write a small stdlib-only module in the collection with unit tests.
  `command` is only for running a real program with a plain `argv`.
- **Group related tasks in a `block`.**
  For per-item work use `include_tasks` plus one `block` per item,
  not index-aligned loops over parallel lists.
- **Lists in block style** (`- item`), including `argv` and `gather_subset`.
  Only an empty list is `[]`.
- **`vars` on the task that uses them**, never at play level.
  A play-level `vars:` overrides inventory values and hides where a value is used.
- **Name loop variables.** Don't leave `item` when a name tells the reader more.
  Use `loop_control: {loop_var: <meaningful>, label: "{{ <short> }}"}`.
  A loop var handed to an included file takes the role prefix (`deploy_host`).
- **Read-only tasks** (`stat`, a `command` that only reads) carry `changed_when: false` and,
  when they must run in check mode, `check_mode: false`.
- **Failures are data.**
  A task that may fail without being fatal uses `failed_when: false` and the role records the
  problem (an `errors` list on the result),
  so one bad host does not stop the play and missing data is never reported as fine.
  Use `block`/`rescue` for discovery steps.
- **No "skipped" noise.** Don't split one task into several `when:` tasks for readability.
  Put the choice in the Jinja, or in a `block` with one `when:`.
- **Call roles with `ansible.builtin.include_role`** and pass parameters through
  `apply: {vars: ...}`. Don't use a play's top-level `roles:`.

## A role is a function

- **Parameters** are `_`-prefixed (`_result_name`, `_privileged`).
- **Declare and check them** with an explicit `ansible.builtin.validate_argument_spec` task at the
  top of `tasks/main.yml`. Do not use `meta/argument_specs.yml`.
- That action only checks values and **does not apply `default:`**.
  State the default in the description and apply it inline at each use:
  `{{ _privileged | default(true) }}`.
- **No `defaults/main.yml`.** Role defaults are visible play-wide and leak into other roles.
- `validate_argument_spec` with `type: str` converts instead of rejecting
  (a list passes as a string), so it fails later where the value is used. Use the exact type.
- **The result goes into a fact whose name the caller gives**: `_result_name`.
  Reset it to `[]` at the start and append where each record is built:

  ```yaml
  - name: Start with no records
    ansible.builtin.set_fact:
      "{{ _result_name | default('_result') }}": []

  - name: Record the item
    ansible.builtin.set_fact:
      "{{ _result_name | default('_result') }}": "{{ lookup('ansible.builtin.vars', _result_name | default('_result')) + [_record] }}"
  ```

  No internal accumulator and a separate "publish" task.
  The playbook gives each role call its own name,
  so roles that run on one host don't overwrite each other.

## Naming (this is what ansible-lint enforces)

| Kind | Prefix | Example |
|---|---|---|
| Role parameter | `_` | `_result_name` |
| Variable used only in one task or block (`vars:`) | `_` | `_command`, `_record` |
| `register` or `set_fact` read only by the tasks next to it (same file, same block) | `__` | `__ps`, `__inspect` |
| Anything another file, a later block or another play reads | role prefix | `deploy_found` |

Why `__`:
`var-naming[no-role-prefix]`
wants the role prefix on every `register` and `set_fact` name inside a role,
and exempts names that start with `__`. A single `_` is not exempt, so `_ps` fails.
Don't skip the rule.
Name temporaries by what they hold (`__first_process_title`, `__package_query`),
not by the command that made them (`__out`, `__cmd1`).

## Jinja

- Prefer a `{%- if -%}...{%- elif -%}...{%- endif -%}` block with whitespace control over long
  inline ternaries or `'%s' %` formatting.
- Move literal dicts out of expressions into a task var written as YAML.
- A folded scalar `>-` keeps backslashes literally, a double-quoted one unescapes them.
  Moving a `regex_replace(..., '\\1')` between the two breaks it.
- `regex_search` with several groups fails, use `regex_findall`. `splitlines()` handles CRLF.
- A `set_fact` key can be templated: `"{{ name }}": value`.
- Don't put `lookup()` in a task `vars:` entry and then use it in an expression.
  ansible-lint renders templates with placeholder values and reports `jinja[invalid]`.
  Put the lookup inline.
- Ansible templates don't autoescape.
  In HTML or shell output, pass every value through `| e` or `| quote`,
  and treat file contents and titles as untrusted.
- `lookup('ansible.builtin.file', ...)` strips the trailing newline unless `rstrip=false`.

## Hosts without a usable Python (CentOS 7 and the like)

Prefer a Python-free path over holding back ansible-core.
`ansible.builtin.raw` runs a command over SSH with no Python.
Ping first (`ansible.builtin.ping`) and send hosts that fail to a `raw` role,
with `group_by` choosing the role per host. In `raw` roles:

- With `become`, SSH gets a terminal. stderr is mixed into stdout and every line ends in CR.
  Strip the CR (`regex_replace('\\r', '')`), read the output from both `stdout` and `stderr`,
  and remove `Shared connection to ... closed.`
- `ps` cuts lines to the terminal width: use `ps -ww`. Old `ps` wants one `-o` per column.
- A command is a string for the remote shell: build it with `| map('quote') | join(' ')`.
- `raw` is the one accepted exception to "modules, not shell". All parsing stays on the controller.

## Choosing targets

- Probe each host and put it in a group with `ansible.builtin.group_by`,
  then run one play per kind of job on that group.
  Set `host_pattern_mismatch = ignore` in `ansible.cfg` so an empty group is silent.
  A play's `hosts:` must be a static group name, not a value that may render empty.
- Default `ansible_ssh_common_args` in a **task** (`set_fact`), not a play `vars:`,
  so an inventory's own value (a bastion) is kept.
- Check required settings in the first play (`ansible.builtin.assert` on localhost),
  with no default, so what a run covers is a decision.

## Lint and runtime gotchas

- ansible-lint rejects Jinja `{% continue %}` and a play `hosts:` that renders empty.
- It guesses a file's kind from its path:
  task files outside a role must sit in a `tasks/` directory,
  and data files under a Molecule directory that look like a playbook go in `vars/`.
- It prints "Ignored exception from ArgsRule" for `docker_image` `build.args`.
  That is an internal error of the tool, not a finding.
- `community.docker.docker_container_copy_into` reads a quoted `mode` as decimal unless
  `mode_parse: octal_string_only` is set.
- Molecule expands `$VARIABLES` in `molecule.yml`,
  but the `molecule_yml` var the playbooks see is the raw file.
  A playbook imported from another directory doesn't get its own adjacent collection,
  so set `ANSIBLE_COLLECTIONS_PATH`.
- Old container image tags can sit on end-of-life distros whose package mirrors are gone,
  so pick a supported tag for test images.
- On Alpine, BusyBox `sed` lacks GNU features and fails silently.
  Edit YAML with Python (`yaml.safe_load`/`safe_dump`) and check the edit did what you meant.

## Tests

- **Plugins and modules**: pytest unit tests. Keep functions pure (return values, don't mutate).
- **The playbook**:
  pytest that runs it against a throwaway inventory with fake tools first on `PATH`
  (a fake `docker`, `systemctl`, `ps`), so no real host is needed.
  Hide the machine's own tools from any probe.
  This covers the plays and the handoff to the plugins in a couple of minutes.
- **Integration**:
  Molecule with mock data, never the project's real data,
  so the tests don't change as the data grows.
  When you add a check to `verify`, prove it can fail:
  corrupt the generated file and expect a non-zero exit.
- After every change run the unit tests and `ansible-lint`.
  Run the playbook tests when you touch the playbook or a plugin it calls.

## Comments and docs

If the `clause-wrap` skill is loaded, follow it.
Otherwise:
wrap prose in comments and docs at about 100 columns, at the end of a clause,
never inside brackets or backticks. Avoid semicolons in prose, write two sentences.
Say why, not what.
A paragraph that a template writes into a Markdown file is one line,
because some viewers show a single newline as a line break.

## Check your work

1. `ansible-lint` passes at `production` with an empty `skip_list`.
2. No `shell:`
   or `sh -c`, no `item` where a name fits, no play-level `vars:`, no top-level `roles:`.
3. Every role starts with `validate_argument_spec`, has no `defaults/main.yml`,
   and sets its result through `_result_name`.
4. Every `register` and `set_fact` in a role is either `__temp` or role-prefixed.
5. Read-only tasks have `changed_when: false`.
6. The unit tests pass, and the playbook tests when the playbook or a plugin changed.
