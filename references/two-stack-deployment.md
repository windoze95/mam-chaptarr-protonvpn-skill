# Generic two-stack deployment

Use this reference when creating or auditing two independent MAM/Chaptarr environments on one Docker host. Every name, port, and path below is an example. Inventory the live host and replace the placeholders before changing anything.

## Contents

- [Stack map](#stack-map)
- [Port and path example](#port-and-path-example)
- [Build and restart order](#build-and-restart-order)
- [Generalized recovery lessons](#generalized-recovery-lessons)

## Stack map

Run one complete namespace per environment:

| Purpose | Stack A | Stack B |
| --- | --- | --- |
| Gateway | `gluetun-a` | `gluetun-b` |
| qBittorrent | `qbittorrent-a` | `qbittorrent-b` |
| Chaptarr | `chaptarr-a` | `chaptarr-b` |
| Port helper | `qbit-port-helper-a` | `qbit-port-helper-b` |
| MAM host updater | `mousehole-a` | `mousehole-b` |
| VPN-only MAM browser | `chrome-mam-a` | `chrome-mam-b` |

Attach all five consumers to their gateway with `network_mode: container:<gateway>`. Give every stateful container independent appdata. Publish LAN-facing ports on the gateway only.

Do not interpret the example names as required image names. Match them to the actual containers and record the exact relationship before applying or restarting anything.

## Port and path example

| Service | Stack A host → namespace | Stack B host → namespace |
| --- | ---: | ---: |
| Gluetun control API (optional) | `8000 → 8000/TCP` | `8001 → 8000/TCP` |
| qBittorrent WebUI | `8080 → 8080/TCP` | `8081 → 8080/TCP` |
| Chaptarr WebUI | `8789 → 8789/TCP` | `8790 → 8789/TCP` |
| Chrome/Selkies HTTPS | `3001 → 3001/TCP` | `3003 → 3001/TCP` |
| Chrome/Selkies HTTP (reverse-proxy only) | `3000 → 3000/TCP` | `3002 → 3000/TCP` |
| HTTP proxy (optional) | `8888 → 8888/TCP` | `8889 → 8888/TCP` |
| Mousehole WebUI | `5010 → 5010/TCP` | `5011 → 5010/TCP` |

Example appdata layout:

```text
<APPDATA_ROOT>/gluetun-a
<APPDATA_ROOT>/qbittorrent-a
<APPDATA_ROOT>/chaptarr-a
<APPDATA_ROOT>/mousehole-a
<APPDATA_ROOT>/chrome-mam-a

<APPDATA_ROOT>/gluetun-b
<APPDATA_ROOT>/qbittorrent-b
<APPDATA_ROOT>/chaptarr-b
<APPDATA_ROOT>/mousehole-b
<APPDATA_ROOT>/chrome-mam-b
```

Use a common in-container completed-download path, such as `/media-server`, for qBittorrent and Chaptarr. Host paths may differ by deployment, but both apps must see the same completed files at the same container path.

For Stack B in this example:

- Configure qBittorrent `WEBUI_PORT=8080`; only the gateway's host side changes to `8081`.
- Configure Chaptarr's qBittorrent client as `localhost:8080`.
- Configure its helper as `QBT_ADDR=http://localhost:8080`, with the qBittorrent WebUI username and password.
- Mount `<GLUETUN_B_APPDATA>/forwarded_port` at `/config/forwarded_port.txt` in the helper.
- Set `PORT_FILE=/config/forwarded_port.txt`.

The static Docker WebUI mappings are not Proton's dynamic incoming peer port. Verify qBittorrent's live listening port against each gateway's `forwarded_port` file.

## Build and restart order

1. Allocate unique container names, appdata paths, and unused host ports.
2. Start the Gluetun gateway and require a healthy Proton tunnel plus a populated forwarded-port file.
3. Start qBittorrent and verify its listener, WebUI, media path, and live namespace binding.
4. Start the port helper and verify its file mount, `PORT_FILE`, qBittorrent address, and restart count.
5. Start Chaptarr and verify its WebUI, disabled app proxy, media path, and qBittorrent client test.
6. Start Mousehole with unique persistent state and its own Dynamic Seedbox-enabled MAM session.
7. Start the dedicated Chrome/Selkies browser in the gateway namespace, publish its authenticated LAN-only WebUI on Gluetun, verify its exit matches qBittorrent, and prove it loses internet when the gateway stops.
8. Enable autostart where desired with the gateway ordered before every consumer. The browser may remain stopped until session work is needed.

After recreating a gateway container, restart or recreate every namespace consumer so it binds the new gateway container ID. A normal tunnel reconnect can rotate the public IP without changing the container ID; Mousehole must then update MAM and both tracker paths must be reverified.

## Generalized recovery lessons

- Docker state `running` is not VPN health. Require Gluetun health, no fresh `AUTH_FAILED`, a Proton exit, and a numeric forwarded-port file.
- An Unraid clone can retain a hardcoded `--net=container:<old-gateway>` in Extra Parameters even when the dropdown shows another gateway. Verify the emitted command and live `HostConfig.NetworkMode`.
- A copied qBittorrent `WEBUI_PORT` can disagree with the gateway mapping. Match the namespace-side listener exactly; a different host-side port does not require changing the internal listener.
- Gluetun needs the `NET_ADMIN` capability and `/dev/net/tun`. Its control API and HTTP proxy are optional; authenticate and restrict either one before publishing it.
- `FIREWALL_OUTBOUND_SUBNETS` permits consumers to initiate LAN connections. It does not open inbound access to a published WebUI.
- Mousehole LAN access requires an exact `MOUSEHOLE_ALLOWED_HOSTS` entry, plus `MOUSEHOLE_ALLOWED_ORIGINS` when a reverse proxy changes the origin. Keep authentication enabled and restrict its plaintext state directory.
- Prefer direct namespace attachment for the MAM browser. An HTTP proxy is fallback-only because it does not remove a bridge-mode browser's direct route. Publish Chrome's WebUI on Gluetun, require authentication, keep it LAN-only, and give each persistent browser unique appdata.
- The helper bind source and destination must both be files. If the source did not exist when Docker created the mount, Docker may have created a directory instead.
- Chaptarr indexer authentication does not authorize qBittorrent's tracker-facing host. Use a separate Mousehole session with Dynamic Seedbox enabled for each gateway.
- A tracker panel can retain an older failure after a successful announce. Refresh the details or inspect a secret-safe Web API result; status `2` means `Working`. Never print the tracker URL because it contains the passkey.
- A working tracker does not fix a missing or unwritable download directory. Inspect the exact save path, category directory, container UID/GID, ownership, and mode before changing permissions.
- Use minimally sufficient permissions. Do not copy broad emergency permissions into a new deployment as a default.
- Keep updater cookies separate. Mousehole may rotate a cookie during an API call, so sharing it between instances creates intermittent failures.
- Treat concurrent authorization of two public exits as account-specific. Run the canary in [mam-tracker-host-authorization.md](mam-tracker-host-authorization.md) and repeat it after exit rotation.
