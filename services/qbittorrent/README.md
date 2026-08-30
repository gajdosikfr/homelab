# qBittorrent

## Overview

Jellyfin runs in Docker because I wanted hands-on experience containerizing a service, since that's the direction I want to grow in professionally. qBittorrent runs natively on the host. It's a single, simple service with no need for isolation, GPU access, or portability, so adding a container on top of it would not have gained anything.

qBittorrent was first set up using the desktop GUI version. It was later replaced with `qbittorrent-nox`, the headless build meant for running on a server without a display.

## Installation

The `qbittorrent-nox` package on Ubuntu ships a templated systemd unit (`qbittorrent-nox@.service`), not a plain `qbittorrent-nox.service`. It needs to be enabled for a specific user instance:

```bash
sudo apt install qbittorrent-nox -y
sudo systemctl enable --now qbittorrent-nox@gg.service
```

`enable --now` both starts the service immediately and ensures it starts automatically on boot. The default unit shipped with the package is used, with no custom modifications, running as the user given after the `@`.

## Architecture

- **User:** runs as `gg` (UID 1000 / GID 1000), the same user Jellyfin's container is mapped to
- **WebUI:** default port `8080`, accessed directly over the local network without a VPN
- **Downloads:** saved directly into the target library folder under `/mnt/data` (`Film` or `Seriály`, depending on content), the same path Jellyfin mounts as `/media`

Because qBittorrent runs as the same UID/GID that Jellyfin's container is mapped to (`1000:1000`), downloaded files are immediately readable by Jellyfin without any extra `chmod`, ACLs, or `group_add` configuration. This is a direct benefit of keeping both services on the same UID/GID scheme, even though one runs in Docker and the other natively.

## Troubleshooting

### Service running outside systemd's control

While reviewing this setup, I noticed the qBittorrent process had actually been started manually from an SSH session at some point, before the systemd unit was ever enabled. Both existed at the same time: a manually started process holding port 8080, and an enabled but inactive systemd unit that had never successfully started.

I checked what was actually managing the running process using `ps` to get its PID, then inspected its cgroup:

```bash
ps aux | grep qbit
cat /proc/<pid>/cgroup
```

The process was inside `user.slice/user-1000.slice/session-*.scope`, which means it belonged to an interactive login session, not to systemd's `system.slice`. Meanwhile, `systemctl status qbittorrent-nox@gg.service` showed `inactive (dead)`. This confirmed that the running instance was not managed by systemd at all, so it would not have restarted automatically after a reboot or crash.

Fix:

```bash
kill <pid>
ss -tlnp | grep 8080      # confirm the port is free
sudo systemctl start qbittorrent-nox@gg.service
systemctl status qbittorrent-nox@gg.service
curl -I http://192.168.0.63:8080
```

The service now starts correctly under systemd, restarts automatically on failure or reboot, and continues seeding without interruption.

**Lesson learned:** a running process is not the same as a supervised service. Checking `ps` alone would have looked fine; checking the process's cgroup against `systemctl status` was what actually revealed the gap.

A full reboot would likely have resolved this too, since the manually started process would not survive it. I intentionally avoided doing that first: a reboot restarts everything at once, and if something else had also been broken, I would have had no way to tell which change fixed what. Fixing and verifying the specific problem first, then using a reboot only afterward as a final confirmation, keeps the diagnosis isolated to one variable at a time.

## Notes

Downloading directly into the final library folder (instead of a separate `downloads/` staging directory) keeps things simple for now, but does mean incomplete downloads sit inside the same folder Jellyfin scans. This has not caused issues so far, but a separate incomplete-downloads folder is worth considering if the library grows or automation (e.g. Sonarr/Radarr) gets added later.

The WebUI is accessed at `192.168.0.63:8080`.

Verified working correctly after a full server reboot: the systemd instance starts on its own, the WebUI comes up, and downloads resume without manual intervention.