---
name: mam-chaptarr-protonvpn
description: Deploy, duplicate, audit, and recover MyAnonaMouse book-download environments built from Chaptarr, qBittorrent, Gluetun, Mousehole, and ProtonVPN on Docker or Unraid. Use for initial setup; adding a second isolated Chaptarr environment; fixing shared-qBittorrent, host-port, network-namespace, Proton authentication, VPN port-forwarding, MAM session, dynamic-seedbox, Unrecognized host/PassKey, or download-path failures; or verifying that MAM and torrent traffic cannot bypass the VPN.
---

# Operate MAM and Chaptarr through ProtonVPN

Build one complete VPN namespace per environment:

```text
Internet
   ^
   | ProtonVPN tunnel
[ Gluetun gateway ]  owns namespace, firewall, kill switch, and host ports
   +-- qBittorrent
   +-- Chaptarr
   +-- qBittorrent port-forward helper
   +-- Mousehole MAM host updater, when this is the authorized tracker egress
   +-- Chrome/Selkies MAM browser, started when session access is needed
```

Do not place two independent Chaptarr environments behind one qBittorrent or add a second qBittorrent with conflicting listeners to the same namespace. Duplicate the whole gateway stack instead. Reusing the same Proton OpenVPN credentials is acceptable; each gateway establishes its own VPN session.

Treat VPN isolation and MAM tracker authorization as separate contracts. A new gateway normally has a new public IP. Chaptarr can authenticate and fetch a `.torrent` while qBittorrent still receives `Unrecognized host/PassKey` because MAM does not recognize the torrent client's exit host. Before scheduling updaters on two different exits for one MAM account, read [references/mam-tracker-host-authorization.md](references/mam-tracker-host-authorization.md).

## Protect secrets and live state

- Keep Proton credentials, Chaptarr `mam_id` values, Mousehole updater cookies, qBittorrent credentials, proxy passwords, Unraid API keys, and browser sessions out of files, chat, command output, and shell history.
- Use `<PLACEHOLDER>` in artifacts. Store live values only in the owning service configuration or system keychain.
- Inspect before mutating. Record the exact container, gateway, host ports, appdata path, and application instance identity for the target environment.
- Never dump all container environment variables merely to find one setting; that can expose credentials. Inspect named non-secret fields or use the Unraid template UI.
- Treat IPs, endpoint hostnames, container IDs, and forwarded ports as rotating evidence. Reverify them live.
- Create and manage these VPN-bound MAM sessions only through the VPN-only browser for the target gateway. Follow MAM's current account and security rules for any other site access.
- For a reusable two-stack template and generalized recovery lessons, read [references/two-stack-deployment.md](references/two-stack-deployment.md).

## Establish the target and mode

Classify the request before changing anything:

1. **New stack:** allocate a gateway, qBittorrent, Chaptarr, port helper, Mousehole, a VPN-only browser, independent appdata paths, and unused host ports.
2. **Duplicate environment:** clone all six containers when the browser is persistent, not just Chaptarr. Offset every host port. Change an internal app listener only when it would collide with another process inside that same gateway namespace.
3. **Recovery:** prove the tunnel, namespace bindings, forwarded-port propagation, and app reachability in that order.
4. **MAM indexer authentication:** create a distinct, non-updater session for the exact Chaptarr instance.
5. **MAM tracker host authorization:** identify qBittorrent's exact public exit, then authorize and maintain that host with a separate updater session.

For a live host, capture a compact inventory without exposing secrets:

```sh
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
docker inspect --format='{{.Name}} network={{.HostConfig.NetworkMode}} restart={{.RestartCount}}' \
  <GATEWAY> <QBITTORRENT> <CHAPTARR> <PORT_HELPER> <MOUSEHOLE> <VPN_BROWSER>
docker inspect --format='{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println}}{{end}}' \
  <QBITTORRENT> <CHAPTARR> <PORT_HELPER> <MOUSEHOLE> <VPN_BROWSER>
```

Resolve collisions before deployment. Docker host ports must be unique across gateways, and listener ports must be unique among processes inside one shared namespace.

## Preserve these architecture invariants

- Attach qBittorrent, Chaptarr, the helper, Mousehole, and the dedicated MAM browser with `network_mode: container:<gateway>` (Unraid Network Type `Container`).
- Publish their LAN-facing ports on Gluetun only. Docker rejects port publishing on a container that uses another container's network namespace.
- Keep Chaptarr's application proxy disabled. Its entire network path already comes from Gluetun.
- Mount completed downloads at the same container path in qBittorrent and Chaptarr.
- Give each environment independent appdata directories and an independent Chaptarr MAM session.
- Give each persistent browser its own appdata so sessions and profiles cannot cross environments.
- Never share a `mam_id` between Chaptarr and Mousehole or between two Mousehole instances. The updater cookie can rotate on every MAM API call.
- Start and autostart the gateway before its namespace consumers.
- After recreating a gateway, recreate or restart every consumer so none retains the removed gateway's old namespace.

## Plan the ports

Use explicit host-to-namespace mappings. This illustrative two-environment scheme avoids collisions when all listed host ports are free:

| Service | Stack A host → namespace | Stack B host → namespace |
| --- | ---: | ---: |
| Gluetun control API (optional) | `8000 → 8000/TCP` | `8001 → 8000/TCP` |
| qBittorrent WebUI | `8080 → 8080/TCP` | `8081 → 8080/TCP` |
| Chaptarr WebUI | `8789 → 8789/TCP` | `8790 → 8789/TCP` |
| Chrome/Selkies HTTPS | `3001 → 3001/TCP` | `3003 → 3001/TCP` |
| Chrome/Selkies HTTP (reverse-proxy use only) | `3000 → 3000/TCP` | `3002 → 3000/TCP` |
| Gluetun HTTP proxy (optional) | `8888 → 8888/TCP` | `8889 → 8888/TCP` |
| Mousehole WebUI | `5010 → 5010/TCP` | `5011 → 5010/TCP` |

The right side is a listener inside the gateway's shared namespace. Because the stacks have separate namespaces, both can normally keep the same default internal listeners while using different host ports. If a deployment deliberately changes an internal listener, make the mapping's right side and the application's listener agree. LinuxServer Chrome currently uses `3001` for its HTTPS desktop and `3000` for HTTP; prefer HTTPS and verify the image's current ports before deploying.

Do not confuse either WebUI mapping with Proton's dynamic incoming torrent port. Some templates also publish same-number UDP mappings for qBittorrent; those static Docker mappings are not the Proton peer port and do not replace the forwarded-port helper.

For additional environments, allocate a fresh set rather than assuming `+1` is free. Check current Docker listeners first.

## Deploy one Gluetun gateway

Create a bridge-mode `qmcgaw/gluetun` container with a unique name and appdata path. Use this baseline:

```text
Repository:                         qmcgaw/gluetun
Network Type:                       Bridge
Required capability:                NET_ADMIN
Required device:                    /dev/net/tun
TZ:                                 <YOUR_TIMEZONE>
VPN_SERVICE_PROVIDER:               protonvpn
VPN_TYPE:                           openvpn
OPENVPN_USER:                       <PROTON_OPENVPN_USER_WITH_PORT_FORWARDING_SUFFIX>
OPENVPN_PASSWORD:                   <PROTON_OPENVPN_PASSWORD>
SERVER_COUNTRIES:                   <VPN_SERVER_COUNTRY>
PORT_FORWARD_ONLY:                  on
SERVER_HOSTNAMES:                   <OPTIONAL_CURRENT_CAPABLE_ENDPOINT>
VPN_PORT_FORWARDING:                on
VPN_PORT_FORWARDING_STATUS_FILE:    /gluetun/forwarded_port
FIREWALL_OUTBOUND_SUBNETS:          <OPTIONAL_LAN_SUBNETS_CONSUMERS_MUST_REACH>
HTTPPROXY:                          off
HTTPPROXY_PORT:                     8888
DOT:                                on
```

For Proton OpenVPN port forwarding, ensure the username presented to Gluetun ends with the current Proton port-forwarding suffix (currently `+pmp`); do not append it twice. Verify this requirement and current option names against the official [Gluetun ProtonVPN guide](https://github.com/qdm12/gluetun-wiki/blob/main/setup/providers/protonvpn.md) and [port-forwarding options](https://github.com/qdm12/gluetun-wiki/blob/main/setup/options/port-forwarding.md) when repairing an older template.

Prefer `PORT_FORWARD_ONLY=on` or a current broad P2P-capable server filter. Pin `SERVER_HOSTNAMES` only when recovering or when exact endpoint control is intentional: a hostname can disappear from Gluetun's server data.

`FIREWALL_OUTBOUND_SUBNETS` allows Gluetun and its namespace consumers to initiate connections to the listed LAN subnets. It is not the setting that opens inbound access to a host-published WebUI. Leave it unset unless a consumer needs LAN egress, and avoid broad or overlapping route ranges.

The control API and HTTP proxy are optional. Publish the control API only when needed and configure authentication using Gluetun's [control-server guide](https://github.com/qdm12/gluetun-wiki/blob/main/setup/advanced/control-server.md). Direct namespace attachment is preferred for the MAM browser. If a proxy fallback is unavoidable, enable `HTTPPROXY`, require `HTTPPROXY_USER` and `HTTPPROXY_PASSWORD`, restrict the host bind or firewall to trusted clients, and prevent the browser container from using a direct non-VPN route. Mount a unique host appdata directory at `/gluetun`, then publish only this environment's required qBittorrent, Chaptarr, browser, Mousehole, and optional proxy/control ports on the gateway.

Examples use recognizable image names. For a stable deployment, pin each image to a tested version or digest and update it deliberately; if using a mutable tag, re-run the end-to-end checks after updates.

Verify the tunnel itself; `running` is insufficient:

```sh
gateway=<GATEWAY>
gateway_data=<HOST_GLUETUN_APPDATA>

docker inspect --format='{{.State.Status}} {{if .State.Health}}{{.State.Health.Status}}{{end}} {{.RestartCount}}' "$gateway"
docker logs --since 10m "$gateway" 2>&1 | grep -c 'AUTH_FAILED'
docker exec "$gateway" wget -qO- -T 15 https://ipinfo.io/json
test -f "$gateway_data/forwarded_port"
test -s "$gateway_data/forwarded_port" && sed -n '1p' "$gateway_data/forwarded_port"
```

Require all of the following:

- Docker state `running` and health `healthy`.
- Zero fresh `AUTH_FAILED` lines.
- An exit IP and ASN belonging to Proton rather than the user's ISP.
- A non-empty numeric forwarded-port file.

For a deliberate kill-switch test, stop the gateway and confirm its consumers cannot reach the internet. Restore the gateway before continuing.

## Deploy qBittorrent in the gateway namespace

Use a unique config directory and the same media path Chaptarr will use:

```text
Network Type:       Container
Container Network:  <GATEWAY>
Config:             <UNIQUE_QBIT_APPDATA> → /config
Downloads:          <HOST_MEDIA_PATH> → /media-server
WEBUI_PORT:         <NAMESPACE_QBIT_WEBUI_PORT>
```

Publish no ports on qBittorrent. Reach its WebUI through the gateway's host mapping.

Retrieve the image's initial WebUI credentials from its first-run log without copying them into artifacts, then replace them with a unique credential stored in the owning service.

Treat `WEBUI_PORT` as an internal listener setting, not the host port. In the example scheme both qBittorrent instances listen on `8080`; Stack B is reached through gateway mapping `8081 → 8080`. If a cloned gateway instead maps `8081 → 8081`, then that qBittorrent instance must listen on `8081`.

### Defuse the Unraid clone trap

Inspect Extra Parameters whenever cloning a namespaced container. A hardcoded value such as:

```text
--restart=always --net=container:<OLD_GATEWAY>
```

can silently override the Network Type dropdown. Change or remove the stale `--net=` argument, apply the template, and verify both the emitted `docker run` command and live state:

```sh
gateway=<GATEWAY>
consumer=<QBITTORRENT>
gateway_id=$(docker inspect --format='{{.Id}}' "$gateway")
consumer_mode=$(docker inspect --format='{{.HostConfig.NetworkMode}}' "$consumer")
printf 'expected=container:%s\nactual=%s\n' "$gateway_id" "$consumer_mode"
test "$consumer_mode" = "container:$gateway_id"
```

Do not trust the dropdown alone.

## Sync Proton's forwarded port into qBittorrent

Deploy one `charlocharlie/qbittorrent-port-forward-file` helper per environment in the same gateway namespace:

```text
Network Type:       Container
Container Network:  <GATEWAY>
QBT_ADDR:            http://localhost:<QBIT_WEBUI_PORT>
QBT_USERNAME:        <QBIT_WEBUI_USERNAME>
QBT_PASSWORD:        <QBIT_WEBUI_PASSWORD>
Host file:           <HOST_GLUETUN_APPDATA>/forwarded_port
Container file:      /config/forwarded_port.txt
PORT_FILE:           /config/forwarded_port.txt
```

Use the qBittorrent WebUI credentials for `QBT_USERNAME` and `QBT_PASSWORD`; do not depend on localhost authentication bypass unless it was deliberately enabled and restricted. The source and destination must both be files, and `PORT_FILE` must exactly equal the container mount target. Create or wait for Gluetun to create the host file before adding the bind mount; otherwise Docker may create a directory and the helper will restart-loop.

Verify:

```sh
test -f <HOST_GLUETUN_APPDATA>/forwarded_port
docker inspect --format='{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println}}{{end}}' <PORT_HELPER>
docker inspect --format='{{.State.Status}} {{.RestartCount}}' <PORT_HELPER>
docker exec <PORT_HELPER> printenv PORT_FILE
```

Reject `/tmp/gluetun/...` host sources in this layout and reject `/config/forwarded_port` versus `/config/forwarded_port.txt` mismatches. If Docker created a directory where the file belongs, stop and inspect it before any removal.

## Deploy Chaptarr in the gateway namespace

Use independent appdata and shared media mounts:

```text
Network Type:       Container
Container Network:  <GATEWAY>
Config:             <UNIQUE_CHAPTARR_APPDATA> → /config
Media:              <HOST_MEDIA> → /media-server
WebUI listener:     <NAMESPACE_CHAPTARR_PORT>
```

Publish no ports on Chaptarr. Publish its listener on Gluetun and confirm the app actually listens at the mapping's namespace-side port. Configure:

- Settings → General → Proxy: **Disabled**.
- Settings → Media Management / Root Folders: the intended books library.
- Completed-download path: the same container path qBittorrent uses, normally `/media-server`.

Track the exact Chaptarr container and instance identity throughout diagnosis. Identical Goodreads IDs may have independent records in different instances.

## Create separate MAM sessions through the target VPN

Prefer a dedicated [LinuxServer Chrome/Selkies](https://docs.linuxserver.io/images/docker-chrome/) container in the exact gateway namespace, following Gluetun's [container-attachment model](https://github.com/qdm12/gluetun-wiki/blob/main/setup/connect-a-container-to-gluetun.md):

```text
Image:              lscr.io/linuxserver/chrome:<TESTED_TAG_OR_DIGEST>
Network Type:       Container
Container Network:  <TARGET_GATEWAY>
Config:             <UNIQUE_BROWSER_APPDATA> → /config
Browser proxy:      Disabled
WebUI listeners:    3001/HTTPS; 3000/HTTP only when reverse-proxied
```

Publish the required Chrome/Selkies listener on Gluetun, never on Chrome. Require `CUSTOM_USER` and a strong `PASSWORD` (or the image's supported secret-file mechanism), keep the WebUI LAN-only, and prefer HTTPS. The container includes a terminal with powerful local access, so never expose it directly to the internet. One browser can share only one gateway namespace at a time: use one browser with unique appdata per environment, or deliberately stop and reattach a single browser before switching gateways.

Before logging into MAM, confirm the browser's public IP/ASN matches qBittorrent in that gateway. Stop Gluetun and confirm the browser cannot load a new site, then restore the gateway. A bridge-mode browser using Gluetun's HTTP proxy is a fallback only; an application proxy does not remove the browser container's ordinary direct route by itself.

Create two session roles; never reuse one cookie for both. The account owner performs signed-in MAM web steps and transfers credentials directly to the owning service. Do not automate the signed-in MAM website or take its browser credentials; use only supported MAM API workflows allowed by the [tracker rules](https://www.myanonamouse.net/rules.php).

### Chaptarr indexer session

1. Open an IP-check site and confirm the target Proton exit IP/ASN, not the ISP.
2. Have the account owner log in to MyAnonaMouse in that browser.
3. Have the account owner open **Preferences → Security → Create session for downloading .torrent files or accessing API end points**. The **IP** field is the live public IP of the target Chaptarr namespace, not its LAN address or the operator's browser IP. Give the owner the verified IP privately rather than asking them to infer it.
4. Choose **ASN locked**, set **Allow Session to set Dynamic Seedbox** to **No**, label the session for the exact Chaptarr instance, and click **Submit Changes!**.
5. Copy the long value labeled `mam_id` on the resulting page (Cookie Info) directly into **Settings → Indexers → MyAnonaMouse → mam_id** on that Chaptarr instance. Copy the value only, not the label, browser login cookie, or tracker passkey. Run **Test → Save** without printing or relaying the value.

Create one indexer session per Chaptarr instance even if two gateways currently exit through the same ASN. Never put the token in diagnostic output or the skill.

### Tracker-host updater session

1. Create another API session for the qBittorrent egress that MAM must recognize.
2. Choose **ASN locked**, set **Allow Session to set Dynamic Seedbox** to **Yes**, and label it for the gateway or updater.
3. Store this cookie only in that updater's persistent state. Do not paste it into Chaptarr, another updater, a shell command, or an artifact.

Some MAM session settings cannot be changed later. Prefer a new dedicated updater session over repurposing an existing Chaptarr session.

Do not add extra ASNs as a substitute for correct VPN routing. A single Gluetun tunnel presents one current exit ASN; bind the session to the ASN actually observed through the target gateway.

## Maintain MAM's recognized tracker host

Use [Mousehole](https://github.com/t-mart/mousehole) to call MAM's Dynamic Seedbox API from the same network namespace and public IP as qBittorrent:

```text
Image:               tmmrtn/mousehole:latest
Network Type:        Container
Container Network:   <QBITTORRENT_GATEWAY>
Persistent state:    <UNIQUE_MOUSEHOLE_APPDATA> → /var/lib/mousehole
Browser auth:        MOUSEHOLE_AUTH_PASSWORD=<UNIQUE_SECRET>
                     or MOUSEHOLE_AUTH_PASSWORD_FILE=<DOCKER_SECRET_PATH>
WebUI listener:      5010 inside the namespace
```

Publish Mousehole's WebUI on Gluetun only when it is needed from the LAN. Give each gateway a unique host port, such as `5010 → 5010` for Stack A and `5011 → 5010` for Stack B. Direct LAN access also requires `MOUSEHOLE_ALLOWED_HOSTS` to include the exact host name or IP used in the URL; a reverse proxy that changes the request host, scheme, or port may also require `MOUSEHOLE_ALLOWED_ORIGINS`. Follow Mousehole's [security guide](https://github.com/t-mart/mousehole/blob/master/docs/security-guide.md), and never use wildcard allowlists on an untrusted network.

Open Mousehole's WebUI, set the dedicated updater cookie, and require a successful state. Mousehole must report the same public IP that qBittorrent uses to announce. Its persistent state must be unique and access-restricted because it contains the MAM cookie in plaintext; a second instance must not overwrite or rotate that cookie.

Treat `OK` as the documented overall success condition, but recognize the current WebUI's last-run results: `Completed` means the IP update succeeded, and a later `No change` means the registered IP already matches. Reject an error state. Store the updater cookie through Mousehole's WebUI only; it does not belong in Chaptarr or qBittorrent.

Two independently authenticated Mousehole instances can work concurrently, but treat that as account- and topology-specific rather than guaranteed. MAM rate-limits changes, Proton exits rotate, and both tracker paths must be reverified after a reconnect. Run the controlled canary in [references/mam-tracker-host-authorization.md](references/mam-tracker-host-authorization.md) before scheduling both updaters.

## Wire Chaptarr to MAM and qBittorrent

In the target Chaptarr instance:

1. Add the native MyAnonaMouse indexer.
2. Paste `mam_id`, run **Test**, require green, then save.
3. Add qBittorrent with host `localhost` and the environment's qBittorrent WebUI listener port.
4. Enable it, run **Test**, require green, then save.

`localhost` is correct because Chaptarr, qBittorrent, and Gluetun share one network namespace.

A failing MAM indexer test can prevent saving and leave the old token in place. A 403 or `Authentication failed` means Chaptarr's MAM request was rejected; it does not establish whether the token expired or its IP/ASN is wrong. Check this indexer's live VPN egress and session scope before replacing only the affected session. On older builds, Chaptarr could discard refreshed session cookies and reuse the original until it expired ([upstream fix #308](https://github.com/Chaptarr/chaptarr/pull/308)). Compare the **installed build** with the fix; a merged development PR does not mean the installed image contains it.

## Verify end to end

Run checks against one exact environment at a time:

```sh
docker exec <GATEWAY> nslookup www.goodreads.com
docker exec <CHAPTARR> getent hosts www.goodreads.com
docker exec <CHAPTARR> wget -S --spider -T 20 https://www.goodreads.com/
docker inspect --format='{{.Name}} {{.State.Status}} {{if .State.Health}}{{.State.Health.Status}}{{end}} {{.RestartCount}}' \
  <GATEWAY> <QBITTORRENT> <CHAPTARR> <PORT_HELPER> <MOUSEHOLE> <VPN_BROWSER>
```

Then verify in the UI:

- Mousehole is healthy and its last run is successful (`OK`, `Completed`, or `No change`); its observed public IP matches qBittorrent's tracker-facing IP.
- While the VPN browser is running, its IP/ASN matches that same gateway and it loses internet access when the gateway is stopped.
- Chaptarr System → Status shows no MAM indexer outage. A green qBittorrent client test only verifies the WebUI connection, and Mousehole success only verifies the updater. Neither proves Chaptarr can search MAM.
- On a known title's detail page, use the **Search** tab with the intended media type and verify an actual **MyAnonaMouse** result without grabbing. A global/Add New catalog lookup tests metadata, not MAM. If MAM works and other indexers are still failing, report those separately.
- For an authorized end-to-end test, a Grab creates a Queue item with live progress, the intended file imports, and the requester can open it from their normal account when a request front end is in scope.
- qBittorrent's MAM tracker status is working, not `Unrecognized host/PassKey`, and peer traffic begins. In Web API tracker results, status `2` is `Working`; inspect only `status` and a redacted `msg`, never print the tracker `url` because it contains the passkey.
- qBittorrent's listening port equals the current Gluetun forwarded port.
- The gateway's exit IP/ASN remains Proton.

Do not declare success merely because the WebUIs load. Require a real test grab or an equivalent authorized canary.

## Recover common failures

### Host port already allocated

Find the sibling gateway or other container that owns the host port. Assign a unique host port and, when mapping to a different namespace port, ensure the application listener matches the namespace-side value.

### Consumer attached to the wrong gateway

Inspect Extra Parameters, correct stale `--net=container:...`, apply, and compare live `HostConfig.NetworkMode` with the target gateway's current container ID. Repeat for qBittorrent, Chaptarr, the helper, Mousehole, and the VPN browser.

### Gateway recreated or renamed

Verify the gateway is healthy, then restart or recreate every namespace consumer. Recheck autostart on the gateway and consumers; Unraid may reset it after recreation or rename. Order the gateway above its consumers.

### VPN browser shows the ISP or the wrong Proton exit

Do not log into MAM. Require the browser's live `HostConfig.NetworkMode` to reference the exact target gateway ID, remove any stale hardcoded `--net=container:...`, and disable browser-level proxy settings. Verify the exit again, then perform the stop-gateway kill-switch check. If one browser is reused between environments, stop and recreate or reattach it rather than assuming a running container changed namespaces.

### Chrome/Selkies WebUI is unreachable

Publish its HTTPS or reverse-proxied HTTP listener on Gluetun, not Chrome. Match the mapping's namespace-side port to Chrome's current internal listener, keep the host port unique, and verify the gateway firewall/bind permits only the intended LAN. Require Chrome WebUI authentication and do not expose it to the internet.

### Fresh Proton `AUTH_FAILED` messages

Do not hammer restart. Stop the gateway for at least two minutes so the old session can expire. Then, within the user's authorization, pull the current image, choose a current port-forward-capable endpoint or broaden the filter, and recreate the gateway. If fresh failures persist, rotate the dedicated Proton OpenVPN credentials in the template without exposing them.

### WebUI unreachable after cloning

Check four layers: unique host port, gateway mapping target, application listener/`WEBUI_PORT`, and correct live namespace. Correct copied Gluetun settings such as `FIREWALL_INPUT_PORTS`, `VPN_PORT_FORWARDING_UP_COMMAND`, `VPN_PORT_FORWARDING_DOWN_COMMAND`, or health/callback URLs that still reference the original port.

### Helper says the port file does not exist

Require `<HOST_GLUETUN_APPDATA>/forwarded_port` to be a real file, mount it at `/config/forwarded_port.txt`, and set `PORT_FILE` to that exact destination. Confirm the helper's restart count returns to zero.

### LAN cannot reach a WebUI

Confirm the host port is published on Gluetun, the application listens on the mapping's namespace-side port, and the host bind/firewall allows the intended LAN client. `FIREWALL_OUTBOUND_SUBNETS` controls consumer-initiated LAN access; it does not open inbound host-published WebUIs. For Mousehole host/origin rejections, configure its exact allowed host and, when needed, origin instead of weakening the VPN kill switch.

### Control API returns Unauthorized

Treat this as expected on Gluetun versions with control-server authentication. Use `docker exec` or configure auth deliberately. Browser requests from an HTTPS Unraid page may also fail because the gateway API is plain HTTP mixed content.

### MAM searches fail while qBittorrent and Mousehole tests pass

Test the native MyAnonaMouse indexer in the specific Chaptarr instance with **Interactive Search** enabled. If there is an authentication error, verify its current exit before asking the owner to replace the session. If the image lacks curl, on the Docker host use its running network namespace without installing anything:

```sh
chaptarr='<CHAPTARR_CONTAINER>'
pid=$(docker inspect --format='{{.State.Pid}}' "$chaptarr")
test "$pid" -gt 0 && nsenter -t "$pid" -n -- curl -4fsS --max-time 10 https://api.ipify.org
```

Replace the placeholder with the actual container name and confirm it shares the expected gateway. The probe does not exercise a configured application proxy; if one is enabled, verify the indexer path separately. Track updater health and Chaptarr indexer auth independently so a healthy updater cannot mask a search outage. Alert on a sustained failure and recovery rather than polling MAM aggressively.

### MAM tracker says `Non-Whitelisted client or version`

Check the running qBittorrent version against MAM's current [Allowed Clients](https://www.myanonamouse.net/tor/allowed_clients.php). A floating image tag can cross into an unapproved version. Pin the affected stack to a currently allowed image and preserve its appdata, queue, paths, and gateway binding. Check WebUI login and the forwarding helper after a recreation, then verify the actual tracker status. Do not rotate a MAM session to fix a rejected client version.

### MAM tracker says `Unrecognized host/PassKey`

Do not change qBittorrent's WebUI username/password; those credentials are unrelated to MAM's tracker passkey.

1. Pause the affected MAM torrent to avoid repeated tracker errors.
2. Redact the announce URL, but record the public IP included in the tracker message.
3. Confirm qBittorrent's current forwarded peer port and Proton exit. Reaching the tracker proves DNS and outbound routing work; it does not prove host authorization.
4. Confirm MAM shows the VPN provider as registered/authorized for the account.
5. Require a dedicated ASN-locked session with Dynamic Seedbox **Yes** and a Mousehole update from the exact qBittorrent namespace.
6. Reannounce only after Mousehole reports a successful update state such as `Completed`, or `No change` for an already-current IP.
7. Require qBittorrent's MAM tracker to become `Working` (Web API status `2`) with no host/passkey error.
8. If fresh torrents still fail from an authorized current IP, verify the account and tracker passkey status with MAM staff. Do not edit the passkey or announce URL.

If qBittorrent's tracker tab still shows an older error after reannounce, query its tracker Web API or refresh the torrent details before concluding the fix failed. In the current qBittorrent API, tracker status `2` with an empty message is working; peer/seed counts and active transfer traffic provide additional confirmation.

If two qBittorrent clients show different public IPs, give each gateway its own updater session and persistent Mousehole state. Follow the two-exit canary in [references/mam-tracker-host-authorization.md](references/mam-tracker-host-authorization.md) and reverify both after an exit rotation. If activating the second host breaks the first, use one MAM torrent egress or ask MAM staff to authorize the intended topology.

### Tracker works but the torrent remains in local `error`

Treat this as a separate filesystem problem. Inspect the torrent's save path, temporary/incomplete path, category subdirectory, qBittorrent process UID/GID, and exact directory ownership and mode. Create a missing category directory and grant only the access required by the container identity. Do not keep changing MAM sessions, VPN routing, or tracker settings once the tracker is working.

After correcting the exact path, resume or force-start the torrent and require data transfer or a completed/seeding state. A working tracker does not repair an unwritable or nonexistent local download directory.

### Chaptarr says Manual Import Required for a book already imported

Check the book's **History** and **Files** against the queued release first. If the matching file is already in the intended library and the torrent should keep seeding, clear only the stale queue entry with **Ignore Download**, without blocklisting. Do not choose **Remove from Download Client** or re-import a duplicate. Recheck that the library file is present, the queue warning cleared, and the original torrent is still seeding.

## Completion checklist

- [ ] One complete gateway/qBittorrent/Chaptarr/helper/Mousehole stack exists per environment, with a directly attached VPN browser available for MAM session work.
- [ ] Host ports do not collide; every app port is published on the correct gateway.
- [ ] Gateway is `running healthy` with zero fresh `AUTH_FAILED` messages.
- [ ] Exit IP/ASN is Proton and `forwarded_port` is a non-empty number.
- [ ] Each consumer's live `HostConfig.NetworkMode` references the exact current gateway ID.
- [ ] qBittorrent's listener matches the gateway mapping and WebUI is reachable.
- [ ] Helper mount and `PORT_FILE` match exactly; restart count is zero.
- [ ] Chaptarr uses the correct appdata, paths, instance identity, and disabled proxy.
- [ ] Gateway and consumers have autostart enabled in dependency order.
- [ ] VPN-only browser uses the exact gateway namespace, unique appdata, a gateway-published authenticated WebUI, no app proxy, the expected exit IP/ASN, and a proven kill switch.
- [ ] Distinct ASN-locked Chaptarr and updater sessions were created for separate roles.
- [ ] Mousehole uses its own persistent state and cookie, reports a successful state (`OK`, `Completed`, or `No change`), and observes qBittorrent's public exit.
- [ ] A two-exit MAM design has passed the controlled canary and is rechecked after reconnects; otherwise only one updater is scheduled.
- [ ] MAM indexer and qBittorrent client tests are green and saved.
- [ ] A real test grab downloads, MAM's tracker status is working, and the dynamic qBittorrent peer port matches Gluetun.
