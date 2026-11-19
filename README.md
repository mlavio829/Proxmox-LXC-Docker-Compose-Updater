# Proxmox LXC Docker Compose Updater

Update Docker Compose stacks across a Proxmox node with one Bash command. The updater discovers deployed projects, pulls their images, and verifies container status. LXCs that start the run powered off are shut down again after maintenance.

[Install](#install) · [Commands](#commands) · [Configuration](#configuration) · [Run summary](#run-summary)

```bash
lxcdockercomposeupdater -plan   # Preview the selected guests without booting them
lxcdockercomposeupdater         # Update all eligible LXCs on this node
```

## Install

Run the installer as **root on the Proxmox node**. The host needs Bash, `pct`,
`flock`, `timeout`, and `wget`. Each guest you want to update needs Docker,
the Compose plugin, and any registry credentials its images require.

<details open>
<summary>First install or migration from the old command name</summary>

The installer resolves GitHub’s `main` branch to a commit and checks the
downloaded script before installing it in `/usr/local/bin`. If you have an old
config, the installer copies it when the new config path is empty.

```bash
(
  set -e
  tmp=$(mktemp /usr/local/bin/.lxcdockercomposeupdater.XXXXXX)
  trap 'rm -f -- "$tmp"' EXIT
  repo=mlavio829/ProxmoxLXCUpdater
  revision=$(wget -qO- --no-cache --header='Accept: application/vnd.github.sha' \
    "https://api.github.com/repos/$repo/commits/main")
  [[ $revision =~ ^[0-9a-f]{40}$ ]]
  wget -O "$tmp" "https://raw.githubusercontent.com/$repo/$revision/lxcdockercomposeupdater"
  bash -n "$tmp"
  grep -Fxq '# lxcdockercomposeupdater self-update format 1' "$tmp"
  chmod 0755 "$tmp"
  if [[ ! -e /etc/lxcdockercomposeupdater.conf && -f /etc/lxcdockercomposeupdate.conf ]]; then
    cp -p --no-clobber /etc/lxcdockercomposeupdate.conf /etc/lxcdockercomposeupdater.conf
  fi
  mv -fT -- "$tmp" /usr/local/bin/lxcdockercomposeupdater
)
```

</details>

Already installed? Update the updater with:

```bash
lxcdockercomposeupdater -getlatest
```

## Commands

| Command | What it does |
| --- | --- |
| `lxcdockercomposeupdater` | Update eligible local LXCs. |
| `lxcdockercomposeupdater 105 108` | Update only LXCs 105 and 108. |
| `lxcdockercomposeupdater -plan` | List selected guests, power states, and configured paths without starting guests. |
| `lxcdockercomposeupdater -config` | Open the default config in an editor. |
| `lxcdockercomposeupdater -config /path/to/file 105` | Use an alternate config for this run. |
| `lxcdockercomposeupdater -prune 108` | Remove dangling Docker images in LXC 108. |
| `lxcdockercomposeupdater -plan -prune` | Preview which guests a bulk prune would visit. |
| `lxcdockercomposeupdater -getlatest` | Update the installed command from GitHub. |
| `lxcdockercomposeupdater -help` | List options. Double-dash flags also work. |

## Configuration

Configuration is optional. The updater reads Docker Compose labels on existing
containers, including stopped ones, to discover project names, working
directories, and Compose files.

Run `lxcdockercomposeupdater -config` to edit
`/etc/lxcdockercomposeupdater.conf`. The updater adds a commented guide if one
is missing. It uses the executable set in `VISUAL` or `EDITOR`, then tries
`nano` and `vi`.

```text
# Keep this guest out of bulk updates and pruning.
exclude 110

# Explicit Compose directories inside the guests.
101 /opt/compose/immich
102 /srv/compose/downloads
102 /srv/compose/home monitoring
```

- An explicit LXC ID on the command line overrides an exclusion for that run.
- Manual paths replace discovery for that guest. List every project you want to update, one per line. Spaces in directory names are supported.
- Add a real tab after the directory and between optional `file=PATH`, `env-file=PATH`, `profile=NAME`, or `project-name=NAME` settings. Repeat `file=` or `profile=` as needed. Relative paths resolve from the project directory inside the guest.

```text
103 /opt/compose/my stack	file=compose.yml	env-file=.env.production	profile=workers	project-name=my-stack
```

## Run summary

```text
Guest                       Result                   LXC power        Prunable
------------------------------------------------------------------------------
AdguardSecondary (LXC 100)  Skipped: no Docker       Kept on                 -
LibreSpeed (LXC 105)        Updated and running      Kept on                 2
Tautulli (LXC 110)          Running (unchanged)      Restored to off         0
```

The **Result** column describes containers at verification time. **LXC power**
shows what happened to the guest afterward. **Prunable** counts dangling images
that no existing container references, including stopped containers. A dash
means the check didn’t run. Actual space reclaimed depends on shared layers.

The live output streams pull progress, shows the Compose command used to
start each project, and lists container status. If a command fails, the updater
prints Docker’s output, the failed step, and its exit code.

## How a run works

1. Select local LXCs and skip templates and bulk-run exclusions.
2. Start guests that are off and wait up to about 60 seconds for Docker. Skip Compose updates when Docker or its Compose plugin is unavailable.
3. Run `docker compose pull`, then `docker compose up -d`, and check the resulting container status. A command failure skips the remaining projects in that guest.
4. Gracefully shut down guests started for maintenance, including after an update failure. Continue to the next guest and report failures in the final summary.

A host-wide lock prevents overlapping updater runs. Schedule the updater
separately from other jobs that change the same guests’ power state. An abrupt
process or host failure can leave a guest running.

### Pruning

Pruning runs separately with the same guest selection and power-state handling.
It removes dangling Docker images and keeps tagged images and volumes. Run
`lxcdockercomposeupdater -prune` across eligible guests or pass IDs to limit it.
