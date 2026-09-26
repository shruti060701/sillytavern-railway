# Deploy and Host SillyTavern on Railway

[![Deploy on Railway](https://railway.app/button.svg)](https://railway.app/new/template/TEMPLATE_CODE)

SillyTavern is an open-source, locally-installed UI for interacting with text-generation LLMs, image models, and TTS voices — deep customization, world info, group chats, and a scripting engine, built by and for AI hobbyists. This template deploys the official image directly, verified live, and fixes the one real security gap every other SillyTavern-on-Railway template ships with unfixed: **the default account has no password until you set one yourself.**

## What's Different About This Template

SillyTavern's own multi-user system creates a `default-user` account with an empty password on first boot. Every reference template documents this as a manual step — "change the password as soon as you can log in" — because the tooling exists (`recover.js`, shipped in the official image) but nobody wires it into the deploy.

This template does. On first boot, a real password is generated (`SILLYTAVERN_ADMIN_PASSWORD`) and applied to the `default-user` account automatically, before the server ever starts accepting connections. There's no window where the instance sits open with no password — verified live by actually logging in with the generated value.

## What's Included

| Component | Detail |
|---|---|
| **Image** | `ghcr.io/sillytavern/sillytavern:latest` — official image, no fork, no third-party wrapper |
| **License path** | SillyTavern itself is AGPL-3.0. This template's own start command is original — it does not reuse code from any unlicensed community wrapper |
| **Persistence** | One volume, relocated via symlinks to cover config, character/chat data, and installed plugins |
| **Config** | Driven entirely by SillyTavern's own environment-variable override mechanism — no templating, no `envsubst`, no custom config files |

## Getting In

1. Open the **SillyTavern** service's deploy logs after first boot.
2. Visit your public domain — you'll land on the login page.
3. Log in as `default-user` with the value of the `SILLYTAVERN_ADMIN_PASSWORD` variable (Railway → Variables tab).
4. Change it from inside SillyTavern's own UI once you're in, if you want a password only you've typed.

## How Persistence Works

The official image expects three separate directories (`config`, `data`, `plugins`) plus `backups`, each independently writable. Railway services get one volume. On first boot, the start command relocates those directories onto the volume (mounted at `/home/node/app/persist`) and replaces them with symlinks — so character cards, chat logs, installed extensions, and any config changes made through the UI all survive a redeploy.

## Configuration

Every setting in SillyTavern's `config.yaml` can be overridden by an environment variable of the form `SILLYTAVERN_<KEY>` (SillyTavern's own built-in mechanism — see `src/util.js`'s `keyToEnv`). This template uses that directly instead of maintaining a parallel templating layer:

- `SILLYTAVERN_LISTEN=true` — bind to all interfaces (required for Railway's proxy to reach it)
- `SILLYTAVERN_WHITELISTMODE=false` — Railway's edge isn't in any IP whitelist, so this must be off for the app to be reachable at all
- `SILLYTAVERN_ENABLEUSERACCOUNTS=true` — turns on the login system (and makes the password fix meaningful)
- `SILLYTAVERN_ENABLEDISCREETLOGIN=false` — toggle to hide the account picker if you don't want your username visible on the login screen

See `TEMPLATE_COMPOSER_CHECKLIST.md` for the full variable reference.

## Healthcheck

Set the **Healthcheck Path** to `/login` — not `/`. `GET /` 302-redirects to `/login` whenever accounts are enabled, and Railway's healthcheck doesn't follow redirects, so `/` will fail every deploy. `/login` returns 200 directly without needing auth, so it's the correct target and passes reliably.

## Verified

Deployed fresh — new volume, no cache. `/login` returns 200, `/` redirects (302) as expected, and an actual login with the auto-generated password succeeds. Persistence confirmed across a redeploy (character/data directories stayed intact via the volume symlinks).

## Dependencies for SillyTavern Hosting

- Official image: [ghcr.io/sillytavern/sillytavern](https://github.com/SillyTavern/SillyTavern/pkgs/container/sillytavern) (AGPL-3.0)
- Source: https://github.com/SillyTavern/SillyTavern
- Documentation: https://docs.sillytavern.app/

## Why Deploy SillyTavern on Railway?

Running SillyTavern anywhere public means dealing with the same three things every time: exposing it safely, persisting your characters and chats across restarts, and not leaving the front door unlocked. This template handles all three — network config via Railway's own proxy, persistence via one relocated volume, and the password gap closed automatically instead of documented as a follow-up chore.

Source for this template's docs: https://github.com/shruti060701/sillytavern-railway
