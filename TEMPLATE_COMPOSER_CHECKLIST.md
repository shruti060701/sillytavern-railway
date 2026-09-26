# Template Composer Checklist — SillyTavern

Full variable reference, pulled from a live, tested deployment (fresh volume, no cache, verified via an actual login with the auto-generated password).

## Service: SillyTavern

Image: `ghcr.io/sillytavern/sillytavern:latest` · Volume: `/home/node/app/persist` · Healthcheck: none (see README — `/` redirects, Railway's checker doesn't follow it, and `/login` didn't resolve reliably either; container liveness is used instead)

Start command (required — relocates state onto the volume, then seeds the admin password before handoff to the official entrypoint):
```
sh -c 'PERSIST_DIR="/home/node/app/persist"; mkdir -p "$PERSIST_DIR"; for d in config data plugins; do if [ ! -L "./$d" ]; then if [ -e "$PERSIST_DIR/$d" ]; then rm -rf "./$d"; else mv "./$d" "$PERSIST_DIR/$d"; fi; ln -s "$PERSIST_DIR/$d" "./$d"; fi; done; [ -e "config/config.yaml" ] || cp default/config.yaml config/config.yaml; npm run init; if [ -n "$SILLYTAVERN_ADMIN_PASSWORD" ]; then node recover.js default-user "$SILLYTAVERN_ADMIN_PASSWORD"; fi; exec ./docker-entrypoint.sh'
```

| Variable | Value | Optional? | Description |
|---|---|---|---|
| `SILLYTAVERN_ADMIN_PASSWORD` | `${{secret(24, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")}}` | No | Auto-generated on deploy. Applied to the `default-user` account on first boot via SillyTavern's own `recover.js` script — this is the fix for the "no password by default" gap present in the reference template. Log in with this value, then change it from inside the UI if you want. |
| `SILLYTAVERN_LISTEN` | `true` | No | Binds to all network interfaces. Required for Railway's proxy to reach the container at all — without it, SillyTavern only listens on localhost. |
| `SILLYTAVERN_WHITELISTMODE` | `false` | No | Disables SillyTavern's own IP whitelist. Railway's edge network isn't a fixed IP you can whitelist, so this must be off for the app to be reachable publicly. |
| `SILLYTAVERN_ENABLEUSERACCOUNTS` | `true` | No | Turns on SillyTavern's multi-user login system. Needed for the password fix above to mean anything — without it, there's no auth at all. |
| `SILLYTAVERN_ENABLEDISCREETLOGIN` | `false` | Yes | When `true`, hides the account picker on the login screen (you type your username instead of selecting it from a list). Cosmetic/privacy toggle only. |
| `SILLYTAVERN_PORT` | `8000` | No | The port SillyTavern's server listens on inside the container. Matches the image's default — don't change unless you also update the service's public networking to match. |

**How these variables work:** every key in SillyTavern's `config.yaml` can be overridden by an environment variable named `SILLYTAVERN_<KEY_UPPERCASED>` — this is a built-in mechanism in SillyTavern itself (`src/util.js`, `keyToEnv`), not something this template adds. There's no templating layer or `envsubst` step involved.

Networking: a single service domain (`<hasDomain>`, routes to port `8000`). No TCP proxy needed — SillyTavern only needs one port.

## Composer Setup Notes — Read Before Publishing

**Do not set a healthcheck path.** `GET /` redirects (302) to `/login` whenever `SILLYTAVERN_ENABLEUSERACCOUNTS=true`, and Railway's healthcheck treats a redirect as a failure. `/login` itself returns 200 correctly when tested directly, but did not resolve the platform's deploy-success gate reliably in testing on this account — the root cause wasn't fully isolated (traced through SillyTavern's middleware chain without finding an obvious blocker) before an unrelated, service-specific control-plane issue was found and fixed by recreating the service. If you see a deploy stuck in a healthcheck-failure loop for several minutes with the container otherwise logging normally (booted, "listening on IPv4", no crashes), try removing the healthcheck path entirely before assuming the app itself is broken.

**Verified live, fresh deployment (new volume, no build cache):**
- Deploy reached `SUCCESS`.
- `GET /login` → `200`. `GET /` → `302` (redirect to `/login`, correct for an unauthenticated visitor).
- Logged in successfully as `default-user` using the exact value of `SILLYTAVERN_ADMIN_PASSWORD` from the deploy's variables — the password fix is confirmed working end-to-end, not just "the script ran."
- Persistence confirmed: character/chat data and config survive a redeploy via the volume-backed symlinks.
