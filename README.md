# Proxmox LXC Docker Compose Updater

Install the Bash command on the Proxmox node that hosts your LXCs. It uses the
host's `pct`, `flock`, and `timeout` commands. Docker, the Compose plugin, and
any registry credentials must already be available to root inside each guest.

## Install

If your installed copy already supports self-update, run this as root:

```sh
lxcdockercomposeupdater -getlatest
```

It resolves GitHub's `main` branch to a commit, downloads that commit's script,
and replaces the installed command if it has changed. It leaves the configuration and LXCs alone.

For a first install or to switch from the older command name, run this from a
root shell on Proxmox.
The download is staged and checked before replacing the installed command.
If upgrading, the installer copies your old configuration when the new default
file does not exist.

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

## Configure and run

Run `lxcdockercomposeupdater` as root to visit every LXC on the local node.
The updater skips templates and guests without Docker. It uses labels on
existing Docker containers (including stopped ones) to find deployed Compose
projects, their original project names, and their Compose files.

For a project without Docker containers or Compose labels, create the optional
`/etc/lxcdockercomposeupdater.conf`. Add one LXC ID and absolute guest directory
per line. Paths can contain spaces. For IDs listed in this file, the updater
uses those directories instead of automatic discovery.

```text
# LXC ID followed by the directory inside the guest
101 /opt/compose/immich
102 /srv/compose/downloads
102 /srv/compose/home monitoring
exclude 110
```

Run `lxcdockercomposeupdater -config` as root
to open the default configuration. It adds a commented guide with examples for
exclusions, manual projects, and Compose settings if the guide is missing. It uses `VISUAL` or `EDITOR` if set to an editor executable,
then falls back to `nano` or `vi`.

Add an `exclude ID` line to skip that guest during bulk updates and pruning.
You can still target it explicitly by passing its ID.

For a configured project, add optional **tab-separated** settings after its
directory. Supported settings are `env-file=PATH`, `profile=NAME`,
`project-name=NAME`, and `file=PATH`. For multiple profiles or Compose files,
repeat the setting in a new tab-separated field. Relative paths resolve from the
guest project directory.

```text
103 /opt/compose/my stack	env-file=.env.production	profile=workers	project-name=my-stack
```

Run as root:

```sh
lxcdockercomposeupdater -help
lxcdockercomposeupdater -plan
lxcdockercomposeupdater 101
lxcdockercomposeupdater
lxcdockercomposeupdater -prune 101
```

The plan lists selected guests, their power state, and any configured paths.
Automatic Compose discovery is deferred until an update run.

Pass one or more IDs to limit the run to those LXCs. Pass
`-config /path/to/file` before the IDs to use a different configuration file.

The updater starts stopped LXCs and waits up to about 60 seconds for Docker.
It runs `docker compose pull` followed by `docker compose up -d` for each
project and shows the container statuses before restoring LXC power state. If
a command fails, the updater preserves Docker’s output and reports the failed
step and exit code. An update failure skips the remaining projects in that LXC. The updater
cleans up and proceeds to the next LXC. The summary lists each guest processed
and distinguishes containers already running from those started or recreated.
It also flags container status that needs review and any failed shutdown. The command exits nonzero if anything failed.

Running LXCs stay running. LXCs started by the updater are gracefully shut down
afterward, including after a failed update. If graceful shutdown fails, the updater
reports the failure. An abrupt process kill or host failure can leave a guest
running. Run the updater separately from other tasks that change the power state
of the same LXCs. A host-wide lock prevents concurrent updater runs.

## Prune old images

`lxcdockercomposeupdater -prune` uses `docker image prune` in eligible guests to remove
dangling images. Tagged images and volumes are preserved. Pruning is a separate
run with the same exclusions and power-state restoration as Compose updates.
Pass an LXC ID to prune only that guest or add `-plan` to preview the guest list.

Double-dash flags also work.
