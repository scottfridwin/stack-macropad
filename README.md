# stack-macropad

Runs [homeassistant-macropad-bridge](https://github.com/scottfridwin/homeassistant-macropad-bridge) on the dining
room kiosk (`kiosk-diningroom`), not on fridwin. Key presses on the USB macropad (a knob, USB ID `1189:8890`) become
`macropad_key` events in Home Assistant, which the **Macropad key event handler** automation turns into volume and
playback controls for `media_player.first_floor`.

## Services

| Container | Image | Role | Runs as | Networks | Reached at |
|---|---|---|---|---|---|
| `macropad-bridge` (service `macropad`) | `ghcr.io/scottfridwin/homeassistant-macropad-bridge` | Reads key presses from `/dev/input` and fires Home Assistant events | `UID_MACROPAD` (1103) : `GID_SERVICES` (1001), plus group `GID_INPUT` | `public` (egress) | no UI or port; outbound only |

- The bridge posts events to the Home Assistant REST API at `http://${LOCALIP_HOME_ASSISTANT}:8123` (plain HTTP on
  the LAN) with a long-lived access token. It does not use MQTT.
- Event settings in `compose.yaml`: `EVENT_TYPE=macropad_key`, `DEVICE_NAME=diningroom_kiosk` (the Home Assistant
  automation maps this name to `media_player.first_floor`), `MACROPAD_VID=1189`, `MACROPAD_PID=8890`.
- The bridge listens on every keyboard interface of the device and picks it up again after it is unplugged.
- Logs use the kiosk's Docker default (`local` driver, capped), not fridwin's syslog.

## Deploy settings

Deployed by the shared scripts, see [stack-komodo](https://github.com/scottfridwin/stack-komodo#stack-deploy-scripts);
on the kiosk they come from the `deploy-scripts-kiosk-diningroom` clone of stack-komodo.
Komodo `environment` (in `komodo-resources/fridwin/stacks.toml`, `server = "kiosk-diningroom"`):

| Setting | Value / effect |
|---|---|
| `SERVICE_UID` / `SERVICE_GID` / `SERVICE_NAME` | `UID_MACROPAD` / `GID_ADMINS` / `macropad` |
| `SHARED_NETWORKS` | `public` (created on the kiosk if missing) |
| `GID_INPUT` | `996`, the kiosk's host `input` group (Raspberry Pi OS trixie). A literal value, not a Komodo variable. |
| `LOCALIP_HOME_ASSISTANT` | Home Assistant's LAN IP, used for `HA_URL` |

No Komodo tags. `auto_update = true`, but the image tag is pinned, so new versions arrive as Renovate PRs that are
merged by hand (no automerge).

## Kiosk prerequisites

Set up by the kiosk bootstrap in [stack-komodo `nodes/kiosk/`](https://github.com/scottfridwin/stack-komodo/blob/main/nodes/kiosk/README.md),
from `nodes/kiosk/hosts/kiosk-diningroom.env`:

- **Service user:** `SERVICE_USERS` contains `macropad:1103`, so the bootstrap creates `service_macropad` (UID 1103,
  primary group `fridwin_services`). There is no other account script for kiosk stacks.
- **Input group:** the host file's `GID_INPUT` is the expected GID of the `input` group. At the end, the bootstrap
  warns if the real GID differs; then set `GID_INPUT` in this stack's environment to
  `getent group input | cut -d: -f3`.
- **Macropad plugged in** at the kiosk's USB port; check with `ls /dev/input/by-id/` (`usb-1189_8890-...-event-kbd`).

## Storage

The stack mounts no `/config`, `/work` or `/data` folders (the deploy scripts still create empty
`/config/macropad`, `/work/macropad`, `/data/macropad`). Nothing needs backing up.

| Path | Contents |
|---|---|
| `/dev/input` (host) | Mounted read-only at `/dev/input`. |
| `/tmp` (tmpfs) | Heartbeat file for the container health check. |

## Secrets

| File in `/config/secrets` (on the kiosk) | Use |
|---|---|
| `macropad_homeassistant_token` | Home Assistant long-lived access token (Compose secret, read via `HA_TOKEN_FILE`). Owned by `service_macropad`, mode `0600`. Restored by hand when the kiosk is rebuilt (see the kiosk README). |

## Security

Non-root (`UID_MACROPAD`), `cap_drop: ALL`, `no-new-privileges`, read-only root filesystem with a tmpfs `/tmp`.
Device access is limited to reading input devices: `/dev/input` is mounted read-only, the device cgroup rule
`c 13:* r` allows only reads of input character devices, and file access comes from the added `input` group.

## Operations / Notes

- **Wrong `GID_INPUT`** shows as `Cannot open ...: Permission denied` in the log. A kiosk reinstall can change the
  `input` GID; compare with `getent group input` on the kiosk.
- **`Macropad 1189:8890 not found, waiting`** means the device is unplugged or has different IDs.
- **Verify:** in Home Assistant, **Developer tools > Events**, listen to `macropad_key` and turn the knob. The
  container is healthy in Komodo while its loop keeps touching the heartbeat file.
- Key names come from evdev and can be joined (for example `KEY_MIN_INTERESTING_KEY_MUTE`); the automation should
  match with `in`, not `==`. See the [bridge README](https://github.com/scottfridwin/homeassistant-macropad-bridge#home-assistant).
