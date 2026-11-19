# Operating a self-hosted relay

How to run a self-hosted relay for an organization. Set it up with [README.md](./README.md) first.

## What you are operating

The relay is workstation-control infrastructure. It signs the proofs that linked machines accept
before granting access, and with relay channels it carries their traffic. Whoever controls the
relay host, its cloud-mint private key or its HTTPS endpoint can act on every linked machine.
Protect it like a CI runner that holds deploy keys.

## The relay host

- Run it on a dedicated host or VM. Limit shell and Docker access to the people who operate it;
  Docker access is root on the host.
- Keep the files in `secrets/` in a secret manager. Never put the cloud-mint private key in an
  image, a repository or a ticket.
- Allow outbound traffic only to your identity provider's discovery and key endpoints, the relay's
  own public address (relay channels check carried environments through it), your certificate
  authority for Caddy, and your package mirror.
- Do not capture its traffic or log request bodies, `Authorization` headers or query strings.
  Channel traffic includes prompts, code and session tokens.
- Leave `RELAY_MANUAL_ENDPOINTS=off` unless machines must be reached at their own address. Use
  relay channels for machines without a public address.
- Optionally restrict who can reach the relay with the `remote_ip` example in `Caddyfile`.
- A restart drops every relay channel until machines reconnect.

## Identity provider

Without these, every account the identity provider issues a token to can sign in to the relay and
link machines. The settings below are named for Microsoft Entra ID; other providers have
equivalents.

- **Assignment required.** In **Enterprise applications**, set **Assignment required?** to **Yes**
  and assign the group of T3 Code users.
- **Conditional Access.** Require MFA and a compliant device for the app, and bound sessions with
  **Sign-in frequency**.
- **Device code flow.** Only headless CLI sign-in needs it, and it is a common phishing route.
  Limit it to a small group, or set **Allow public client flows** to **No** when nobody needs it.
- **No API scope.** Do not add scopes under **Expose an API**. The relay accepts ID tokens whose
  audience is the client id and refuses access tokens.
- **Token lifetime.** A stolen ID token works at the relay until it expires, an hour by default.
  A shorter ID token lifetime narrows that window.
- **Issuer.** Use the tenant-specific v2.0 issuer, never `/common` or `/organizations`.

Pin your client builds to this relay with `T3CODE_RELAY_PINNED` or `T3CODE_RELAY_ALLOWLIST`
([connect setup](../../../docs/operations/connect-setup.md#restricting-relays)), so users cannot
be pointed at a look-alike relay.

## Linked machines

- Turn on **Require approval for new devices** in **Settings → Sharing & access** on each host.
  Without it, anyone who signs in as the user, or who holds the relay's key, can connect a new
  device without a prompt.
- Agent activity publishing sends project and thread titles to the relay. Decide whether hosts
  should publish it.

## Offboarding

Disabling a user in the identity provider stops new sign-ins, but machines they linked and devices
that already connected keep working until revoked. In order:

1. **Identity provider.** Disable the account or remove it from the app's group, then revoke its
   sessions.
2. **Relay.** Run
   `docker compose exec relay node src/node.mjs offboard --subject <sub> [--issuer <iss>]`
   with the user's `sub` claim. In one transaction it revokes their environment links and
   credentials and deletes their device and push registrations. Their relay channels close within
   seconds. `--issuer` defaults to `OIDC_ISSUER`.
3. **Their machines.** On each machine the user linked, revoke their devices with
   `t3 auth connect-device list` and `t3 auth connect-device revoke <code>`, and their sessions
   with `t3 auth session list` and `t3 auth session revoke <id>`.
4. **Verify.** Their devices no longer appear in either list, and a sign-in from their account
   fails at the identity provider.

## Known limitations

- **No end-to-end encryption.** The relay operator can read and change relay channel traffic, and
  can mint access to linked machines with the cloud-mint key.
- **Custom-scheme mobile redirects.** Another Android app can register the same scheme and receive
  the sign-in redirect. PKCE stops it from redeeming the code alone.
- **Single replica.** Relay channels, queues and rate limits live in one process.
