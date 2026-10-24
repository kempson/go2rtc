# go2rtc homelab fork

Forked from https://github.com/AlexxIT/go2rtc. Our change lives on branch `homelab-frigate`. Frigate runs this build in place of the go2rtc bundled in its image.

## Why we maintain this fork

Our Reolink cameras emit h264 at **Level 5.1** (4K). Upstream go2rtc's `pkg/h264/h264.go:GetProfileLevelID` only whitelists levels `{30, 31, 40}` and forces everything else to `41`. That means go2rtc advertises `avc1.XX0029` (Level 4.1) to MSE clients. Chrome rejects this combination for 3840x2160 content because L4.1's maximum luma sample rate can't carry 4K, so MSE stalls with `DEMUXER_UNDERFLOW` / `BUFFERING_HAVE_NOTHING` after about a second.

## Change vs upstream

- **`pkg/h264/h264.go`**: level whitelist expanded to `{30, 31, 32, 40, 41, 42, 50, 51, 52, 60, 61, 62}`.

With this, our stream gets advertised correctly as `avc1.640033` (High 5.1) and Chrome accepts it.

## Build

`.github/workflows/homelab-binary.yml` builds a static linux/amd64 binary on every push to `homelab-frigate`. It replaces the rolling `homelab-frigate` release with two assets, `go2rtc_linux_amd64` and `go2rtc_linux_amd64.sha256`. Nothing is built on a laptop.

## Deploy

homelab-apps `apps/frigate/scripts/bootstrap.sh` downloads both assets, refuses a binary that doesn't match its checksum, and pushes it to `/opt/frigate/go2rtc-patched` in LXC 111 on node4. `apps/frigate/docker-compose.yml` mounts that file read-only over `/usr/local/go2rtc/bin/go2rtc` in the Frigate container, so recreating or restarting the `frigate` container picks up a new binary. Replace the host file rather than `docker cp` into the container, because the mount is read-only and the host file wins on the next restart anyway.

## Verify

```bash
pct exec 111 -- docker logs frigate 2>&1 | grep "INF go2rtc platform"
```

The version should end in `+dev`, which marks a build from this fork. Frigate's bundled go2rtc reports a release version. In the browser, `chrome://media-internals/` should show `avc1.640033` for the 4K live view.

## Rollback

Remove the `go2rtc-patched` bind mount from `apps/frigate/docker-compose.yml` in homelab-apps and recreate the `frigate` container. Frigate then runs the go2rtc bundled in its image, and 4K live view in Chrome stalls again.
