# MAM tracker-host authorization

Read this reference when Chaptarr can search or fetch from MyAnonaMouse but qBittorrent reports `Unrecognized host/PassKey`, or before putting two MAM qBittorrent clients behind different public IPs on one account.

## Contents

- [Separate the three credentials](#separate-the-three-credentials)
- [Recognize the failure](#recognize-the-failure)
- [Authorize one tracker egress](#authorize-one-tracker-egress)
- [Test a two-exit design](#test-a-two-exit-design)
- [Fallback if concurrent authorization fails](#fallback-if-concurrent-authorization-fails)
- [Interpret updater errors](#interpret-updater-errors)

## Separate the three credentials

| Credential | Owner | Dynamic Seedbox | Reuse rule |
| --- | --- | --- | --- |
| Chaptarr `mam_id` | One Chaptarr indexer instance | **No** | Never share with another Chaptarr or updater |
| Mousehole updater cookie | One updater instance | **Yes** | Never share with Chaptarr or another updater |
| Tracker passkey | Embedded by MAM in `.torrent` announce URLs | Not applicable | Never print or edit casually |

Chaptarr authentication and tracker-host authorization are separate. A successful indexer test proves that the Chaptarr cookie works from its allowed ASN. It does not authorize qBittorrent's public IP with the tracker.

MAM can rotate the updater cookie during a Dynamic Seedbox API call. Store it only in one Mousehole instance's persistent state. Reusing that cookie in Chaptarr or a second updater creates intermittent invalid-session failures.

## Recognize the failure

Treat these signals together as a tracker-host authorization problem:

- Chaptarr's MAM test is green and it fetches `.torrent` files.
- qBittorrent reaches `t.myanonamouse.net` but reports `Unrecognized host/PassKey` and includes its public IP.
- The torrent remains stalled with zero peers.
- Gluetun has a healthy Proton tunnel and qBittorrent's listen port matches the forwarded-port file.

Do not blame the qBittorrent WebUI login. Do not reveal the announce URL while diagnosing; redact the tracker passkey between `tracker.php/` and `/announce`.

## Authorize one tracker egress

1. Pause the MAM torrent to avoid unnecessary tracker retries.
2. Confirm MAM shows the VPN provider as registered/authorized for the account. If not, open a MAM staff ticket before continuing.
3. Through the VPN-only browser sharing the exact qBittorrent gateway namespace, create a dedicated ASN-locked API session with **Allow Session to set Dynamic Seedbox = Yes**.
4. Do not reuse either Chaptarr's `mam_id` or another updater's cookie.
5. Run Mousehole in `network_mode: container:<gateway>` for the exact qBittorrent egress.
6. Persist `/var/lib/mousehole` in a unique appdata directory and protect its WebUI with `MOUSEHOLE_AUTH_PASSWORD`.
7. Set the cookie through Mousehole's UI, require a successful result (`OK` in the documentation, `Completed` after an update, or `No change` when already current), and confirm its observed IP equals qBittorrent's current public exit.
8. Reannounce the paused torrent and require the MAM tracker to become working. In qBittorrent Web API tracker results, status `2` is `Working`; inspect `status` and a redacted `msg` without printing the tracker URL or passkey.

Do not paste the updater cookie into a terminal command. If a manual API call is unavoidable, use a secret-safe mechanism that does not store the cookie in shell history or logs.

## Test a two-exit design

Do not assume an untested MAM account can retain two concurrent tracker hosts. MAM enforces a cooldown after changes, and two scheduled updaters on different public IPs can fail or trigger rate limiting if their sessions or state are shared.

Use this controlled canary before scheduling both:

1. Disable automatic updates on both prospective updater instances.
2. Update MAM from gateway A only.
3. Require Mousehole A `OK`, `Completed`, or `No change` and qBittorrent A's tracker working.
4. Record only the public IP and timestamp; never record the cookie or passkey.
5. Update MAM once from gateway B.
6. Require Mousehole B `OK`, `Completed`, or `No change` and qBittorrent B's tracker working.
7. Immediately reannounce the same authorized canary on qBittorrent A.
8. If A becomes unrecognized, stop. The second update displaced the first; do not schedule competing updaters.
9. If both remain working, observe through at least one permitted update interval and a gateway reconnect before declaring concurrent-host support.

Respect MAM's update cooldown. Do not repeatedly alternate the two IPs while testing.

Do not treat success on one account or one pair of public IPs as proof that every two-exit deployment is supported. Keep updater cookies and appdata independent, verify both trackers again after the second update, and repeat the check after a reconnect or material topology change.

## Fallback if concurrent authorization fails

If MAM accepts only one dynamic tracker host, keep only one complete MAM-enabled gateway/qBittorrent/Chaptarr stack active. Disable the MAM indexer and updater in the other environment, or use a stable tracker-host arrangement that MAM staff explicitly approves.

Do not point two otherwise independent Chaptarr environments at one shared qBittorrent merely to work around the host limit. That reintroduces category, path, credential, and lifecycle coupling and violates this skill's isolation model. A deliberate shared-client or multi-client redesign is outside this runbook and requires its own supported port-forwarding and path-isolation design.

Do not create another MAM account to work around host limits. Do not assume two Proton tunnels using the same account, city, hostname, or ASN share one public IP.

## Interpret updater errors

- `Incorrect Session Type - Not Allowed This Function`: create a new API session with Dynamic Seedbox **Yes**.
- `Invalid Session - IP Mismatch`: recreate the updater session as ASN locked.
- `Invalid Session - ASN Mismatch`: add the updater's actual exit ASN through MAM's supported session-management flow, then retry.
- `Invalid Cookie`: obtain a new dedicated updater cookie and do not share it.
- `Last Change Too Recent`: wait for MAM's cooldown; do not restart-loop or alternate gateways.
- Mousehole reports success but the tracker remains unrecognized: verify Mousehole and qBittorrent have the same current public IP, require a fresh announce, and query qBittorrent's tracker Web API to exclude stale UI state before contacting MAM support. Do not rotate secrets repeatedly.

Current operational references:

- [qBittorrent WebUI API tracker status values](https://github.com/qbittorrent/qBittorrent/wiki/WebUI-API-%28qBittorrent-4.1%29)
- [Mousehole overview and deployment](https://github.com/t-mart/mousehole)
- [Mousehole cookie/session rules](https://github.com/t-mart/mousehole/blob/master/docs/getting-your-cookie.md)
- [Mousehole MAM error handling](https://github.com/t-mart/mousehole/blob/master/docs/mam-errors.md)
