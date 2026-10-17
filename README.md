# server-onboarding

An agent skill that adds a server to a fleet harness repository: it reaches the box over
ssh, discovers it read-only, places it in a namespace, writes its host card and README,
mirrors its configs, draws it into the namespace Topology and regenerates the index.
When the GPU stack of the box is incomplete it hands the box to `gpu-server-setup`, and
only after the user agreed.

It is one of four skills of a universal devops harness:

| Skill | Does |
|---|---|
| [server-discovery](https://github.com/EvilFreelancer/server-discovery) | read a box, refresh its card, validate the fleet, regenerate the index and the map |
| **server-onboarding** | add a new box: placement, card, README, mirrored configs, Topology, GPU hand-off |
| [server-healthcheck](https://github.com/EvilFreelancer/server-healthcheck) | is a box or a namespace actually healthy, read-only |
| [gpu-server-setup](https://github.com/EvilFreelancer/gpu-server-setup) | NVIDIA or Intel Arc driver, Docker GPU access, serving stacks, power caps |

**It needs `server-discovery` installed next to it**: the probes, the card writer and the
validator live there, and so do the conventions it fills in (host card, namespace card,
Topology notation, `fleet.yaml`).

## What it does

1. **Reachability** - batch-mode ssh; an unknown host key is confirmed with the user by
   its fingerprint, a changed one is a stop; a password is used only when the user
   authorized it for this task, never on a command line or in a file.
2. **Discovery** - one capture of the box, parsed into the card facts, plus the network
   probe for the narrative and the Topology.
3. **Placement** - the namespace whose `cidr` holds the primary address, or a question.
4. **Card and README** - every required key present, the facts written by the script,
   the password policy of `fleet.yaml` respected.
5. **Mirror** - compose projects and system configs, in the layout `fleet.yaml` names
   (`box-path` or `containers`), secrets as `*.example`.
6. **Topology** - through the gateway or the jump host, only edges the probe proves.
7. **Index and verification** - `render_index.py --check`, the hostname on the box, the
   card stays `pending` while a required key is `null`.

## Install

Inside a harness repository, as git submodules next to each other:

```bash
git submodule add https://github.com/EvilFreelancer/server-discovery.git .agents/skills/server-discovery
git submodule add https://github.com/EvilFreelancer/server-onboarding.git .agents/skills/server-onboarding
ln -s ../../.agents/skills/server-onboarding .claude/skills/server-onboarding   # Claude Code
```

As plain skill folders, put both repositories side by side in one skill root
(`~/.claude/skills/`, `~/.cursor/skills/`, `~/.codex/skills/`, `~/.coddy/skills/`); the
directory names must match the `name` fields.

## Usage

- "Onboard ops@203.0.113.10, it is the new embeddings box."
- "Add node7 to the lab namespace and prepare its GPUs."
- "Finish the pending card of node3."

## License

MIT © Pavel Rykov.
