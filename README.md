# stack-macropad

Komodo stack that runs [homeassistant-macropad-bridge](https://github.com/scottfridwin/homeassistant-macropad-bridge)
on the dining room kiosk (`kiosk-diningroom`). Key presses on the USB macropad become `macropad_key` events in Home
Assistant, which the **Macropad key event handler** automation turns into media player volume and playback controls.

## Prerequisites (on the kiosk)

- Secret `${DIR_CONFIG}/secrets/macropad_homeassistant_token`: a Home Assistant long-lived access token, owned by
  `UID_MACROPAD`, mode `0600`.
- `GID_INPUT` (Komodo stack environment) must be the kiosk's `input` group ID: `getent group input | cut -d: -f3`.

## Settings

| Setting | Where | Value |
| --- | --- | --- |
| `UID_MACROPAD` | Komodo variable | Service UID the container runs as |
| `GID_INPUT` | Stack environment | Host `input` group, so the container can read `/dev/input` |
| `DEVICE_NAME` | `compose.yaml` | `diningroom_kiosk`; the Home Assistant automation keys off this name |
| `MACROPAD_VID` / `MACROPAD_PID` | `compose.yaml` | `1189` / `8890`, from `ls /dev/input/by-id/` |

Minor and patch releases of the bridge image are merged automatically by Renovate and deployed by the Komodo webhook.
