---
name: server-onboarding
description: >
  Onboard a server into a fleet harness repository. Connects over ssh, discovers OS,
  hardware, GPUs, network, Docker stacks and access facts read-only, places the host
  in a namespace, writes <namespace>/<host>/README.md with the frontmatter card,
  mirrors its configs, updates the namespace Topology and regenerates the index,
  then hands a box whose GPU stack is incomplete to gpu-server-setup once the user
  agrees. Use when the user asks to onboard, add, register, document or analyse a
  server, to finish a pending host, or to "add it to the index". Re-scanning a host
  that already has a card is the server-discovery skill.
metadata:
  version: 0.2.0
  author: Pavel Rykov <paul@drteam.rocks>
  homepage: https://github.com/EvilFreelancer/server-onboarding
  triggers: >
    onboard server, add server, register host, document server, new host card,
    add to the index, pending host, namespace topology, mirror compose stacks,
    добавить сервер, онбординг сервера, завести карточку хоста
---

# Server onboarding

This skill needs the `server-discovery` skill next to it: its scripts read the box and
write the card, and its `references/` hold the conventions this skill fills in
([host-card.md](../server-discovery/references/host-card.md),
[namespace-card.md](../server-discovery/references/namespace-card.md),
[harness.md](../server-discovery/references/harness.md)). Read the harness's own
instructions (`AGENTS.md` and the rules it lists) and its `fleet.yaml` first; they
decide the mirror layout and whether a password may sit in the card.

The commands spell the skills as `.agents/skills/<skill>/`, the project-level install;
use the real prefix when they live elsewhere. Steps 1 to 7 do not change the target
host. The GPU hand-off at the end does, and only after the user said yes.

## Inputs

- an ssh target, written below as `<target>`: what follows `ssh` in the command that
  reaches the box (`node1`, `root@203.0.113.10`, `-p 2222 ops@203.0.113.10`); the card's
  `ssh` field becomes `ssh <target>`;
- optionally the namespace; otherwise it is derived from the address;
- optionally what the box is for (feeds `description` and `tags`) and whether its GPUs
  should be prepared.

## Procedure

Each step is a gate: do not go on while it fails.

### 1. Reachability

```bash
ssh -o BatchMode=yes -o ConnectTimeout=15 <target> 'echo ok'
```

Prints `ok`: continue. A box this machine has never seen stops at `Host key verification
failed`: show the user the fingerprint from
`ssh-keyscan -T 10 <address> 2>/dev/null | ssh-keygen -lf -` and, once they confirm it,
retry with `-o StrictHostKeyChecking=accept-new` right after `ssh`. Never
`StrictHostKeyChecking=no`, and after `REMOTE HOST IDENTIFICATION HAS CHANGED` stop and
ask instead of editing `known_hosts`.

If key authentication fails and the user explicitly supplied or authorized a password in
the current task, use it through an interactive prompt or protected input and continue.
Never put passwords on a command line, in files or logs. Without explicit authorization,
stop and ask the user to run `ssh-copy-id <target>` in their own terminal, then retry.

### 2. Discovery (read-only)

Read the box once into a file and parse it; step 4 applies the same capture:

```bash
timeout 180 ssh -o BatchMode=yes -o ConnectTimeout=15 <target> sh -s < .agents/skills/server-discovery/scripts/discover.sh > /tmp/<host>.discover.txt
grep -qx '### end' /tmp/<host>.discover.txt && python3 .agents/skills/server-discovery/scripts/discover.py --input /tmp/<host>.discover.txt
```

The JSON gives the card facts (`hostname`, `user`, `auth`, `sudo`, `os`, `kernel`,
`docker`, `driver`, `cpu`, `ram`, `gpus`, `disks`) and three probe keys that stay out of
the card: `nvidia_pci_gpus`, `docker_nvidia_runtime` and `notes`. Keep the notes for the
GPU hand-off. `gpus: []` next to `nvidia_pci_gpus: 2` is a box whose NVIDIA driver is
missing, not a box without GPUs. Intel discrete GPUs show up in `gpus` once i915 or xe
drives them; an Arc card the probe does not list is a question for `lspci`.

For the narrative, the Topology and the mirror run the network probe, read-only:

```bash
ssh -o BatchMode=yes -o ConnectTimeout=15 <target> sh -s < .agents/skills/server-discovery/scripts/discover-net.sh
```

It prints addresses, routes, listening ports, Docker publish ports, the compose projects
with the path of each compose file, wireguard and OpenVPN state, `/etc/hosts`,
`~/.ssh/config` and DNAT rules. Add
`systemctl list-units --type=service --state=running --no-pager | head -40` and
`ls /etc/ssh/sshd_config.d/` when the box is not Docker-centric.

Record the primary address (the one the ssh target resolves to) and the default route.

### 3. Placement

Compare the primary address with `cidr` of every `<namespace>/README.md`. Exactly one
match: that namespace. No match (a VPS with a public IP) or several: ask the user, offer
the closest namespaces (a provider namespace with `cidr: []` is the usual home of a VPS)
and a new one. A new namespace needs its own README (card, Topology with the gateway
rule, `## Hosts` markers) before the host is added.

### 4. Write the host README

Create `<namespace>/<host>/README.md` where `<host>` is the label the user uses (the ssh
alias when there is one). Start with the card, every required key present:

```yaml
---
hostname: <first entry is what hostname printed; add FQDN and public DNS names>
description: <one line>
tags: [<from the vocabulary>]
status: pending
ssh: ssh <target>
ip: <primary address>
jump: <namespace/host or null>
auth: null
user: null
sudo: null
os: null
kernel: null
docker: null
driver: null
cpu: null
ram: null
gpus: []
disks: []
---
```

Then fill the facts from the capture of step 2:

```bash
python3 .agents/skills/server-discovery/scripts/discover.py --input /tmp/<host>.discover.txt --apply <namespace>/<host>/README.md
```

Add `gpu` to `tags` when `gpus` or `nvidia_pci_gpus` shows GPUs, and `docker` when
`docker` is set. If `auth` came back with `password`, add the `password` key right after
`user`. With `password_in_card: false` (the default) it is `password: null`, and a
password the user gives you stays out of git; with `password_in_card: true` it holds the
value the user gave for the card, or `null`. Then the narrative, in the language of the
harness:

```markdown
# <host>

<one paragraph: what the box is, where it stands, since when>

## host
- **Main user** - ...
- **sshd** - ...
## network
- **Address** - ... (iface, DHCP or static, gateway)
## ports
| Port | Service | Exposure |
## users and access
## stacks
### <stack>
- **Folder** - <the path on the box>, mirrored in [<the mirror path>](<the mirror path>)
- run section (docker compose ps / up -d / restart / down)
```

### 5. Mirror configs

Where a file goes is `mirror` in `fleet.yaml` ([harness.md](../server-discovery/references/harness.md),
"Mirror"):

- `box-path` (the default): every file sits at its path on the box with the leading
  slash dropped. A compose project goes to the directory of its compose file as
  `docker compose ls` reports it: `/home/ops/containers/docker-vllm/docker-compose.yaml`
  becomes `<host>/home/ops/containers/docker-vllm/docker-compose.yaml`,
  `/srv/vllm/compose.yml` becomes `<host>/srv/vllm/compose.yml`. sshd drop-ins,
  systemd units you rely on, sudoers drop-ins and `/etc/docker/daemon.json` go under
  `etc/`, local scripts such as `/usr/local/sbin/nvidia-power-limit.sh` under
  `usr/local/sbin/`, files from a home directory under `home/<user>/` or `root/`. Never
  add a folder of your own at the top of the host folder (`containers/`, `configs/`):
  every directory there is a directory of the box;
- `containers`: every compose project goes to `<host>/containers/<stack>/`, system
  configs to `<host>/etc/` under their real paths.

In both layouts the Dockerfiles, the app code and the static configs a project mounts go
next to its compose file; `.env` and every config that holds a secret become `*.example`
with the values replaced by placeholders and described. Data, model weights, caches and
logs are not mirrored.

**Vendor telemetry.** While the stacks are read, check whether any of them reports to its
vendor: vLLM usage statistics, Infinity and Chroma analytics, the Gradio and OpenWebUI
calls, Ray usage stats. The switch of each component and what it stops are in
[telemetry.md](../gpu-server-setup/references/telemetry.md) of the `gpu-server-setup`
skill. Read-only, printing only the switch names, never the whole environment:

```bash
ssh -o BatchMode=yes <target> 'for c in $(docker ps --format "{{.Names}}"); do
  echo "$c $(docker inspect --format "{{.Config.Image}}" "$c")"
  docker inspect --format "{{range .Config.Env}}{{println .}}{{end}}" "$c" \
    | grep -E "^(VLLM_NO_USAGE_STATS|(VLLM_)?DO_NOT_TRACK|HF_HUB_DISABLE_TELEMETRY|INFINITY_ANONYMOUS_USAGE_STATS|SCARF_NO_ANALYTICS|ANONYMIZED_TELEMETRY|ENABLE_VERSION_UPDATE_CHECK|GRADIO_ANALYTICS_ENABLED|RAY_USAGE_STATS_ENABLED)=" | sed "s/^/  /"
  docker exec "$c" ls /root/.config/vllm/usage_stats.json 2>/dev/null | sed "s/^/  has reported: /"
done'
```

A vLLM engine without `VLLM_NO_USAGE_STATS=1` or `DO_NOT_TRACK=1` (the value `1`, `true`
does not silence vLLM), a `usage_stats.json` in its container, an Infinity without
`INFINITY_ANONYMOUS_USAGE_STATS=0` or `DO_NOT_TRACK=1`, an OpenWebUI without
`ENABLE_VERSION_UPDATE_CHECK=false`: each goes into the stack section of the README and
into the final report. On the host, `popularity-contest` with `PARTICIPATE="yes"`,
`whoopsie` and `ubuntu-report`, and on Ubuntu `ENABLED=1` in `/etc/default/motd-news` and
`apt_news` in `pro config show`, go there too. Switching any of it off changes the box:
ask, unless the rules of the harness already order it.

### 6. Topology

Add the host to `## Topology` of the namespace README: inside the right `subgraph`,
reached through its gateway or jump host, labels on DNAT ports, in the notation of
[namespace-card.md](../server-discovery/references/namespace-card.md). Add unmanaged gear
it depends on to `### Network gear`. Draw only edges the network probe or a config on the
box shows.

### 7. Index and verification

```bash
python3 .agents/skills/server-discovery/scripts/render_index.py && python3 .agents/skills/server-discovery/scripts/render_index.py --check
ssh -o BatchMode=yes <target> hostname
```

The second command must print the first entry of `hostname`. Set `status: active` when
no required key is `null` any more, otherwise leave `pending` and list the gaps in the
final report together with what was written, the notes of step 2 and the vendor telemetry
found in step 5.

## GPU hand-off

A note from step 2 means the NVIDIA stack of the box is incomplete: GPUs on the PCI bus
that `nvidia-smi` does not list (no driver, a broken one, a card not driven) or Docker
without the `nvidia` runtime. An Intel Arc card that `lspci` shows but `gpus` does not
list, or one without a `/dev/dri/renderD*` node, is the same case for Intel. Steps 1 to 7
still document the box as it is; then:

1. Say what was found and ask whether to prepare the GPUs now, unless the user already
   asked for it in this task. The `gpu-server-setup` skill installs packages and reboots
   the box; a no leaves the card `pending` and the findings in the report.
2. On a yes, make sure the skill is present: when
   `.agents/skills/gpu-server-setup/SKILL.md` is missing and the harness carries it as a
   git submodule, run `git submodule update --init .agents/skills/gpu-server-setup`. Use
   its Prepare mode for a box without a driver and Diagnose for a broken one, and follow
   its gates; the mandatory end is its stage-2 gate, a container that sees every GPU.
3. Read the box again into a fresh capture (step 2), compare it with the card and apply
   as the `server-discovery` skill says: `driver` and `gpus` fill in and the notes are
   gone.
4. Mirror what the setup put on the box (`/etc/docker/daemon.json`, the power cap unit,
   each serving stack) as step 5 says, and write the driver branch, the power cap and the
   stacks into the README body. Every stack the setup brought up carries its telemetry
   switches; check them as step 5 says.
5. Run step 7 again.

A firewall or DNAT change on a gateway is a change on the gateway host, handled through
its own README, not here.

## Refreshing a known host

A host that already has a card is re-scanned with the `server-discovery` skill, not
with this one. Come back here only to finish a `pending` host: the narrative, the
stacks, the Topology, the GPU hand-off and `status`.

## Common mistakes

- `discover.py --ssh "<target>"` without the word `ssh`: the script runs the target as a
  program. The capture of step 2 avoids it;
- reading `gpus: []` as "no GPU" while `nvidia_pci_gpus` counts cards on the bus;
- starting `gpu-server-setup` without a yes from the user;
- mirroring into a layout other than the one `fleet.yaml` names, or guessing the path of
  a compose project instead of reading `docker compose ls`;
- writing a password into the card while `password_in_card` is off;
- `StrictHostKeyChecking=no` to get past an unknown host key;
- drawing a Topology edge nobody can point to in the probe output;
- mirroring a stack without noting that it still reports to its vendor, or printing the
  whole environment of a container to find out (it carries keys).
