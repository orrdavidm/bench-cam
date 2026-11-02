# bench-cam

Stream a USB document camera from an Ubuntu machine over the LAN, using
[MediaMTX](https://github.com/bluenviron/mediamtx) for serving and ffmpeg for
capture and encoding. View it in a browser (WebRTC) or from any RTSP client.

- Browser (WebRTC, lowest latency): `http://<bench-pc>.local:8889/bench`
- RTSP (VLC, ffmpeg, OpenCV, OBS): `rtsp://<bench-pc>.local:8554/bench`

`<bench-pc>` is the hostname (or IP address) of the machine running the script.

## Requirements

- Ubuntu (tested on 26.04), x86_64 or arm64, with `sudo` access
- A USB camera that offers MJPEG (most document cameras and webcams do)
- Internet access on that machine, to install packages and download MediaMTX

## Install

Copy the script to the machine and run it as root:

```bash
scp bench-cam-setup.sh <bench-pc>.local:/tmp/
ssh -t <bench-pc>.local 'sudo bash /tmp/bench-cam-setup.sh'
```

The script is idempotent. Re-run it after changing the settings or after
plugging in the camera.

It will:

1. Install `ffmpeg`, `v4l-utils`, `curl` and `tar`.
2. Find the first USB camera under `/dev/v4l/by-id/` (or use `CAMERA` if set).
3. Pick an encoder: `h264_nvenc` if an NVIDIA GPU is usable, otherwise `libx264`.
4. Download the latest MediaMTX release to `/usr/local/bin/mediamtx`.
5. Write `/etc/mediamtx/mediamtx.yml`, which has ffmpeg publish the camera to
   the `bench` path.
6. Create and start a `mediamtx` systemd service running as an unprivileged user.
7. Open the needed ports in `ufw`, if it is active (8554/tcp, 8889/tcp, 8189/udp).

### No camera yet

If no camera is attached, the script still installs and starts MediaMTX but
writes a config with no capture command. The RTSP path then returns 404 until
you plug the camera in and re-run the script.

## Settings

Edit the block at the top of `bench-cam-setup.sh`, then re-run it.

| Setting | Default | Meaning |
|---|---|---|
| `STREAM_NAME` | `bench` | Path in the URLs |
| `WIDTH` / `HEIGHT` | `1920` / `1080` | Capture size; must be an MJPEG mode the camera offers |
| `FPS` | `30` | Frame rate |
| `BITRATE` | `8M` | Encoder bitrate |
| `CAMERA` | empty | Device path; empty auto-picks the first USB camera |
| `MEDIAMTX_VERSION` | empty | Release tag such as `v1.21.3`; empty uses the latest |

List the sizes and frame rates your camera offers:

```bash
v4l2-ctl -d /dev/v4l/by-id/<camera>-video-index0 --list-formats-ext
```

Higher resolutions raise CPU load when encoding with `libx264`. Start at 1080p.

## Verify

From another computer on the LAN:

```bash
nc -z <bench-pc>.local 8554 && nc -z <bench-pc>.local 8889 && echo ports open
ffprobe -v error -rtsp_transport tcp rtsp://<bench-pc>.local:8554/bench
ffmpeg -rtsp_transport tcp -i rtsp://<bench-pc>.local:8554/bench -frames:v 1 -y bench.jpg
```

On the bench PC:

```bash
systemctl status mediamtx
journalctl -u mediamtx -f
```

## Focus

Cameras often default to continuous autofocus, which can hunt on a bench. To
lock it:

```bash
v4l2-ctl -d <camera> -l
v4l2-ctl -d <camera> -c focus_automatic_continuous=0 -c focus_absolute=400
```

Most cameras do not expose a light or LED control over USB, so any built-in
lamp is switched on the camera itself.

## Using it in OBS or Zoom

Add the stream to OBS, then use OBS's virtual camera as the camera in Zoom.

- **WebRTC (try first):** add a Browser Source with the WebRTC URL. Lowest latency.
- **RTSP (fallback):** add a Media Source, uncheck Local File, and enter the RTSP
  URL. Expect more latency. Turn off hardware decoding if you see glitches, and
  tick "Restart playback when source becomes active".

## Troubleshooting

- **RTSP returns 404:** nothing is publishing. Check that the camera is
  attached (`ls /dev/v4l/by-id`), re-run the script, then read
  `journalctl -u mediamtx`.
- **WebRTC page loads but video is black or stalls:** UDP 8189 is probably
  blocked between the two machines. Use RTSP over TCP instead.
- **High CPU:** lower `WIDTH`/`HEIGHT`/`FPS`, or use a machine with an NVIDIA GPU.
- **MoQ warning about `auto.key` in the log:** harmless. It does not affect
  RTSP or WebRTC.

## Security

The stream is served without authentication to anyone who can reach the ports.
Use it on a trusted LAN only. Do not forward these ports to the internet.
