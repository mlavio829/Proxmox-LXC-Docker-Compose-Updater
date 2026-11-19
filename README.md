# Proxmox LXC Docker Compose Updater

Install the Bash command on the Proxmox node that hosts your LXCs. It uses the
host's `pct`, `flock`, and `timeout` commands. Docker, the Compose plugin, and
any registry credentials must already be available to root inside each guest.

## Install

From a root shell on Proxmox, run the following to download and install the command from GitHub.

```sh
wget -O lxcdockercomposeupdate https://raw.githubusercontent.com/mlavio829/ProxmoxLXCUpdater/main/lxcdockercomposeupdate &&
  install -m 0755 lxcdockercomposeupdate /usr/local/bin/lxcdockercomposeupdate
```

If you have a local checkout on the node, run the same `install` command from
that checkout.

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
```

Run as root:

```sh
lxcdockercomposeupdate 101
lxcdockercomposeupdate
```

Pass one or more IDs to limit the run to those LXCs. Pass
`--config /path/to/file` before the IDs to use a different configuration file.

The updater starts stopped LXCs and waits up to about 60 seconds for Docker.
It runs `docker compose pull` followed by `docker compose up -d` for each
project. An update failure skips the remaining projects in that LXC. The updater
cleans up and proceeds to the next LXC, then exits nonzero if anything failed.

Running LXCs stay running. LXCs started by the updater are gracefully shut down
afterward, including after a failed update. If graceful shutdown fails, the updater
reports the failure. An abrupt process kill or host failure can leave a guest
running. Run the updater separately from other tasks that change the power state
of the same LXCs. A host-wide lock prevents concurrent updater runs.
