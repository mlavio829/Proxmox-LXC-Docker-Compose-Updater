# Proxmox LXC Docker Compose Updater

Install the Bash command on the Proxmox node that hosts your LXCs. It uses the
host's `pct`, `flock`, and `timeout` commands. Docker, the Compose plugin, and
any registry credentials must already be available to root inside each guest.

## Install or update

Run this from a root shell on Proxmox. The same commands install a new copy or
replace an existing one with the latest version on GitHub's `main` branch.
The download is staged and checked before replacing the installed command.
Your configuration file stays in place.

```sh
(
  set -e
  tmp=$(mktemp /usr/local/bin/.lxcdockercomposeupdate.XXXXXX)
  trap 'rm -f -- "$tmp"' EXIT
  wget -O "$tmp" https://raw.githubusercontent.com/mlavio829/ProxmoxLXCUpdater/main/lxcdockercomposeupdate
  bash -n "$tmp"
  chmod 0755 "$tmp"
  mv -f -- "$tmp" /usr/local/bin/lxcdockercomposeupdate
)
```

## Configure and run

Run `lxcdockercomposeupdate` as root to visit every LXC on the local node.
The updater skips templates and guests without Docker. It uses labels on
existing Docker containers (including stopped ones) to find deployed Compose
projects, their original project names, and their Compose files.

For a project without Docker containers or Compose labels, create the optional
`/etc/lxcdockercomposeupdate.conf`. Add one LXC ID and absolute guest directory
per line. Paths can contain spaces. For IDs listed in this file, the updater
uses those directories instead of automatic discovery.

```text
# LXC ID followed by the directory inside the guest
101 /opt/compose/immich
102 /srv/compose/downloads
102 /srv/compose/home monitoring
exclude 110
```

An `exclude ID` line skips that guest during a bulk run. You can still update
it by explicitly passing its ID.

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
lxcdockercomposeupdate --plan
lxcdockercomposeupdate 101
lxcdockercomposeupdate
```

The plan lists selected guests, their power state, and any configured paths.
Automatic Compose discovery is deferred until an update run.

Pass one or more IDs to limit the run to those LXCs. Pass
`--config /path/to/file` before the IDs to use a different configuration file.

The updater starts stopped LXCs and waits up to about 60 seconds for Docker.
It runs `docker compose pull` followed by `docker compose up -d` for each
project. An update failure skips the remaining projects in that LXC. The updater
cleans up and proceeds to the next LXC. The summary lists each guest processed
and flags any failed shutdown. The command exits nonzero if anything failed.

Running LXCs stay running. LXCs started by the updater are gracefully shut down
afterward, including after a failed update. If graceful shutdown fails, the updater
reports the failure. An abrupt process kill or host failure can leave a guest
running. Run the updater separately from other tasks that change the power state
of the same LXCs. A host-wide lock prevents concurrent updater runs.
