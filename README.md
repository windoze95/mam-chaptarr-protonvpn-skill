# MAM + Chaptarr through ProtonVPN

An agent skill for building, duplicating, auditing, and recovering VPN-isolated book-download stacks: **Gluetun** (ProtonVPN gateway), **qBittorrent**, **Chaptarr**, a forwarded-port helper, **Mousehole**, and a VPN-only browser — on Docker or Unraid.

The hard parts it encodes are the ones that cost hours the first time:

- **One namespace per environment.** Every consumer attaches with `network_mode: container:<gateway>`; ports publish on the gateway only. Duplicating a stack means duplicating the whole gateway, not bolting a second Chaptarr onto one qBittorrent.
- **VPN isolation and tracker authorization are separate contracts.** Chaptarr can authenticate and fetch a `.torrent` while qBittorrent still gets `Unrecognized host/PassKey`, because MAM doesn't recognize the torrent client's exit host. Different problem, different fix.
- **Two session roles, never one cookie.** An ASN-locked indexer session for Chaptarr (dynamic seedbox *off*) and a separate updater session for Mousehole (dynamic seedbox *on*). Sharing one rotates the cookie out from under the other.
- **The Unraid clone trap.** A hardcoded `--net=container:<old-gateway>` in Extra Parameters silently overrides the Network Type dropdown, so a cloned container keeps routing through the stack you thought you left.
- **Verification that means something.** `running` isn't proof. The checks require a Proton exit ASN, zero fresh `AUTH_FAILED` lines, a non-empty forwarded port, live `HostConfig.NetworkMode` matching the gateway's current container ID, and a real grab that transfers.

## Layout

| File | Contents |
|---|---|
| `SKILL.md` | Entry point: architecture invariants, port planning, per-container deployment, end-to-end verification, failure recovery |
| `references/two-stack-deployment.md` | Reusable two-environment template and generalized recovery lessons |
| `references/mam-tracker-host-authorization.md` | Tracker-host authorization, the two-exit canary, updater error interpretation |
| `agents/openai.yaml` | Codex interface metadata |

## Install

Clone the repo into your agent's skills directory. `SKILL.md` uses standard skill frontmatter (`name`, `description`), so it loads in Codex and Claude Code alike; `agents/openai.yaml` supplies the Codex display name and default prompt.

## Secrets

The skill is written to keep credentials out of artifacts. Proton logins, `mam_id` values, updater cookies, qBittorrent passwords, and Unraid API keys stay in the owning service's configuration or a system keychain — never in files, chat, command output, or shell history. Every example uses `<PLACEHOLDER>`. Diagnostic commands deliberately avoid dumping whole environments or printing tracker announce URLs, which contain the passkey.

Nothing in this repo is a credential, and nothing here is specific to one deployment.

## After the stack is up

This skill's job ends when Chaptarr is running, authenticated, and grabbing. If you want a request layer on top of it, [**Cantinarr**](https://github.com/windoze95/cantinarr) connects to a Chaptarr instance and gives the rest of the household per-format (eBook / audiobook) requests, live availability, and a push notification when a book lands — without handing anyone the Chaptarr UI or an API key.

It dials instances from its own server, so a Chaptarr inside a gateway namespace works fine: point the instance URL at the gateway that publishes the port, since a namespace-sharing container has no address of its own.

## License

MIT — see [LICENSE](LICENSE).
