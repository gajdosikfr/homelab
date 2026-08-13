# Jellyfin

## Overview

I wanted a simple home media server. I first tried a basic Samba setup with VLC and Windows sharing to my Android TV, but it was a total pain. Then I found Jellyfin. It's open-source, looks great, and just works.

The setup is managed with Docker Compose. I mounted the media and config folders from the host, so I don't lose any data when updating or deleting the container. I also passed through my Intel iGPU to handle hardware transcoding easily.

A must-have feature for me is the 5.1 to stereo audio downmixing. It finally fixes that annoying issue where sound effects blow your ears out but you can't hear the voices.


## Architecture

Jellyfin runs as a Docker container on my Ubuntu Server. The deployment is managed with Docker Compose.

- **Networking:** Host network mode
- **Media storage:** `/mnt/data` mounted to `/media`
- **Configuration:** `/home/gg/jellyfin/config` mounted to `/config`
- **Cache:** `/home/gg/jellyfin/cache` mounted to `/cache`
- **Hardware acceleration:** Intel HD Graphics 630 via `/dev/dri`
- **Container user:** UID 1000 / GID 1000
- **GPU access:** `render` group (GID 991)
- **Restart policy:** `unless-stopped`

## Hardware Transcoding

The server has an Intel HD Graphics 630 iGPU. The `/dev/dri` device is passed through to the Jellyfin container and Intel Quick Sync (QSV) is enabled in Jellyfin.

I normally use Direct Play to keep the original video quality, but hardware transcoding is available when a client requires a different codec, resolution or bitrate.


## Troubleshooting

### Hardware transcoding failed

After enabling hardware transcoding, the video failed to play and the transcoding process crashed.

I checked the GPU device permissions and found that `/dev/dri/renderD128` belongs to the `render` group:

```
crw-rw---- root render /dev/dri/renderD128
```

The `render` group uses GID `991`, while the container was originally configured with GID `109`. I changed the supplementary group in `compose.yaml`:

```
group_add:
  - "991"
```

After recreating the container, Jellyfin could finally access the render device. I tested it by forcing a video to transcode, and hardware transcoding worked correctly with Intel Quick Sync.