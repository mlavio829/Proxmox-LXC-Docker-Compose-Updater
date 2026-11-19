# Proxmox LXC Docker Compose Updater

Install the Bash command on the Proxmox node that hosts your LXCs. It uses the
host's `pct`, `flock`, and `timeout` commands. Docker, the Compose plugin, and
any registry credentials must already be available to root inside each guest.

## Install

From a root shell on Proxmox, run the following after publishing this repository
to GitHub. Replace `OWNER/REPO` and `main` with your repository and branch.

```sh
wget -O lxcdockercomposeupdate https://raw.githubusercontent.com/OWNER/REPO/main/lxcdockercomposeupdate &&
  install -m 0755 lxcdockercomposeupdate /usr/local/bin/lxcdockercomposeupdate
```

If you have a local checkout on the node, run the same `install` command from
that checkout.

## Configure and run

Create `/etc/lxcdockercomposeupdate.conf` with one LXC ID and absolute guest
directory per line. Use one line per Compose project. Paths can contain spaces.

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

With no IDs, the command updates all configured LXCs on that node. Pass
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
