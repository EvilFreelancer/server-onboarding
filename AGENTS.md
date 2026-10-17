# Authoring & maintaining the `server-onboarding` skill

Agent guide for working **inside this skill repository**. Read it before editing
`SKILL.md` or any plugin manifest.

> Naming (Russian): the adjective for "agent / agentic" is **«агентный»**, never
> «агентский».

## Repository layout

```
server-onboarding/
├─ .claude-plugin/plugin.json  .cursor-plugin/plugin.json  .codex-plugin/plugin.json
├─ SKILL.md              # CANONICAL skill: YAML frontmatter + instructions
├─ README.md  AGENTS.md  LICENSE
```

The skill has no scripts of its own: it drives the scripts of `server-discovery`, and
the conventions it fills in are the `references/` of that skill.

## Editing rules

- **Steps are gates.** Every step says what passes it; a step that changes the box (only
  the GPU hand-off does) runs after an explicit yes from the user.
- **No harness wins by default.** Anything that differs between harnesses (mirror
  layout, password policy, extra tags) is read from `fleet.yaml`, never hard-coded for
  one repository.
- **Access rules stay strict**: batch-mode ssh, fingerprint confirmation for a new host
  key, a stop on a changed one, passwords only through protected input and never on a
  command line, in files or logs.
- **Hand-offs** go to `server-discovery` (refresh of a known host) and
  `gpu-server-setup` (anything that installs drivers), by skill name.

## No personal data

The skill is public and serves any fleet. Examples, fixtures and tests use neutral
values only: hosts `node1`, `node2`, a namespace `lab`, the login `ops`, private ranges
(`10.0.0.0/24`) and the documentation ranges (`192.0.2.0/24`, `198.51.100.0/24`,
`203.0.113.0/24`), `example.org`. Never copy a real hostname, login, address, domain,
password or incident date from a harness into this repository; the lessons of an
incident stay, its names go. Before every commit:

```bash
grep -rnE '[0-9]{1,3}(\.[0-9]{1,3}){3}' --include='*.md' --include='*.py' --include='*.sh' . | grep -vE '\b(10|127)\.|192\.0\.2\.|198\.51\.100\.|203\.0\.113\.|0\.0\.0\.0'
```

prints nothing but version numbers.

## Keeping metadata in sync

When you change the description or bump the version, update it in **all** of:

- `SKILL.md` frontmatter (`metadata.version`, `description`)
- `.claude-plugin/plugin.json`
- `.cursor-plugin/plugin.json`
- `.codex-plugin/plugin.json` (both top-level and the `interface` block)
- `README.md`

Keep the `name` identical everywhere: `server-onboarding`. The Claude manifest must not carry a
`category` key, the plugin schema rejects it. Keep the `SKILL.md` frontmatter strict YAML
(`description` as a folded `>` block), otherwise some agents load the skill without its
description.

## The set of fleet skills

`server-discovery`, `server-onboarding`, `server-healthcheck` and `gpu-server-setup` are
released separately but work as one set inside a harness, installed side by side in
`.agents/skills/`. The contract between them:

- `server-discovery` owns every script that reads a box into a card or validates the
  fleet, and the conventions in its `references/`; the other skills call its scripts by
  `.agents/skills/server-discovery/scripts/<script>` and link its references by
  `../server-discovery/references/<file>`;
- a renamed script, a changed command line, a new JSON key of `discover.py` or a new
  `fleet.yaml` key is a change in every skill that mentions it, released together;
- hand-offs name the other skill by its `name`, never by a path of one harness.

When the set is published in the [rpa-skills](https://github.com/EvilFreelancer/rpa-skills)
catalog, mirror every version and description change there as its `AGENTS.md` says.
