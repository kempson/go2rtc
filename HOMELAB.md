# go2rtc — homelab fork

Forked from https://github.com/AlexxIT/go2rtc. The patch lives on branch `homelab-frigate` (see `pkg/h264/h264.go`). CI builds the binary and publishes it as the `homelab-frigate` release asset; homelab-apps `apps/frigate/scripts/bootstrap.sh` downloads it and verifies the checksum. Nothing is vendored into homelab-apps.

## Why we maintain this fork

Our Reolink cameras emit h264 at **Level 5.1** (4K). Upstream go2rtc's `pkg/h264/h264.go:GetProfileLevelID` only whitelists levels `{30, 31, 40}` and forces everything else to `41`. That means go2rtc advertises `avc1.XX0029` (Level 4.1) to MSE clients. Chrome rejects this combination for 3840×2160 content because L4.1's maximum luma sample rate can't carry 4K, so MSE stalls with `DEMUXER_UNDERFLOW` / `BUFFERING_HAVE_NOTHING` after ~1 second.

## Change vs upstream

- **`pkg/h264/h264.go`**: level whitelist expanded to `{30, 31, 32, 40, 41, 42, 50, 51, 52, 60, 61, 62}`.

With this, our stream gets advertised correctly as `avc1.640033` (High 5.1) and Chrome accepts it.

## Build (static binary for Debian glibc-based Frigate container)

Run from the homelab-apps repo root so `$PWD/apps/frigate/vendor/go2rtc` is the vendored source path.

```bash
# One-time setup (from homelab-apps repo root)
docker run -d --platform linux/amd64 --name go2rtc-builder \
  -v "$PWD/apps/frigate/vendor/go2rtc:/src" -w /src \
  golang:alpine sleep infinity
docker exec go2rtc-builder apk add --no-cache build-base

# Every iteration — CGO_ENABLED=0 makes it static so it'll run in Frigate's Debian container
docker exec go2rtc-builder sh -c 'CGO_ENABLED=0 go build -o /src/go2rtc -ldflags "-s -w" .'
```

Binary lands at `./go2rtc` (~20 MB stripped static).

## Deploy to Frigate LXC

Frigate's in-container path `/usr/local/go2rtc/bin/go2rtc` is a read-only bind mount sourced from `/opt/frigate/go2rtc-patched` on the LXC host (see `apps/frigate/docker-compose.yml`). Install by replacing the host file, never `docker cp` into the container (the mount is `:ro` and the host file would shadow it on restart anyway).

```bash
scp go2rtc frigate:/opt/frigate/go2rtc-patched
ssh frigate 'chmod +x /opt/frigate/go2rtc-patched && docker restart frigate'
```

Frigate's s6 init will restart its go2rtc subprocess automatically, or the docker restart covers it.

## Verify the patch took effect

```bash
ssh frigate 'wget -qO- http://127.0.0.1:1984/api/streams?src=garden | grep -oE "\"level\":[0-9]+"'
# Should show "level":51 if camera is on High 5.1 (not 41)
```

In the browser: `chrome://media-internals/` should show `Browser negotiated codecs: video/mp4; codecs="avc1.640033"` for the live view.

## Rollback

Replace `/opt/frigate/go2rtc-patched` on the LXC with a stock go2rtc binary (e.g. extracted from ghcr.io/alexxit/go2rtc) and `docker restart frigate`.
