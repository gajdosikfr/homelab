# Homelab Portfolio

This repository documents my journey from an automotive technician to a Linux System Administrator. It contains notes, configurations, and lessons learned from my homelab, covering Linux, Docker, networking, automation, and DevOps.

The goal is not only to build working services, but to understand how they work and why they are designed that way.

I used AI mainly to help explain commands and their options, and to review and clean up the documentation. Everything documented here was actually built and tested on my own hardware. The write-ups just went through AI-assisted editing afterward.

## Server

| | |
|---|---|
| Host | HP ProDesk 600 G3 SFF |
| CPU | Intel Core i5-7500 (4C/4T) @ 3.80 GHz |
| GPU | Intel HD Graphics 630 (integrated) - used for Jellyfin hardware transcoding |
| RAM | 16 GB |
| OS | Ubuntu 25.10 |
| Storage | 8 TB Seagate IronWolf (`/mnt/data`, ext4) for media, separate system NVME disk |
| Network | Static local IP on a home LAN |

The server currently runs a full desktop environment. This was a leftover from when I started and wasn't confident setting things up headless yet. I'd approach this differently today.

## Services

| Service | Description |
|---|---|
| [Jellyfin](services/jellyfin/) | Self-hosted media server with Intel Quick Sync hardware transcoding |
| [qBittorrent](services/qbittorrent/) | Torrent client, installed via apt and run natively as a systemd service |

## Projects

| Project | Description |
|---|---|
| [Storage migration: 2 TB NTFS → 8 TB ext4](projects/storage_migration/) | Replaced the media disk over network transfer, migrated to ext4/UUID mount |