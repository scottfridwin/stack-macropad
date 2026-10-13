# Notes for agents and maintainers

This repo is a Komodo stack deployed by the shared scripts in
[stack-komodo](https://github.com/scottfridwin/stack-komodo) (see its README and AGENTS.md).
Its settings are the `[[stack]]` entry for `macropad` in
[komodo-resources](https://github.com/scottfridwin/komodo-resources) `fridwin/stacks.toml`.
It runs on the kiosk server `kiosk-diningroom`.

- Files in `config/`, `work/`, `data/` are copied over the live copies on every deploy and
  rendered with envsubst: any `$VAR` must be set or the deploy fails; use `$$VAR` for a literal.
- Merging a change to `compose.yaml` redeploys automatically; other changes need a manual Deploy.
- Don't commit files the app rewrites at runtime; use `name.seed` for create-if-missing files.
- Check the live state read-only (`docker inspect`, `docker logs`); never print secrets.

## This stack
- It is not on fridwin: `docker inspect` / `docker logs` there will not find `macropad-bridge`. Use Komodo
  (server `kiosk-diningroom`) or SSH to the kiosk as `scott`; host setup is in stack-komodo `nodes/kiosk/`.
- The `service_macropad` account (UID 1103) is created by `nodes/kiosk/bootstrap-kiosk.sh` from `SERVICE_USERS` in
  `nodes/kiosk/hosts/kiosk-diningroom.env`, not by `sync-service-users.sh`. A new UID must change both places.
- `GID_INPUT` in the stack environment is a literal (996) that must equal the kiosk's `input` group GID; the
  bootstrap warns on a mismatch. Keep `group_add`, the read-only `/dev/input` mount and `c 13:* r` together:
  removing any of them breaks reading the device.
- `DEVICE_NAME=diningroom_kiosk` and `EVENT_TYPE=macropad_key` are matched by the Home Assistant automation;
  changing them silently stops the knob from working.
- The bridge uses Home Assistant's REST API (`HA_URL`, token secret), not MQTT.
- Do not run a second bridge on the kiosk host (an earlier `macropad.service` unit did this): every key press would
  fire twice.
