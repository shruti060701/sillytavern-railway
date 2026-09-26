# Deploy and Host SillyTavern on Railway

SillyTavern is an open-source, locally-installed interface for text-generation LLMs, image-generation engines, and TTS voice models — built for AI hobbyists who want real control over their prompts, personas, and world info, not a stripped-down chat box. Beginning in February 2023 as a fork of TavernAI 1.2.8, it now has over 200 contributors and multiple years of independent development behind it. This template deploys SillyTavern's official image directly and fixes the one real gap every other SillyTavern-on-Railway template documents but doesn't solve: the default account ships with no password.

## About Hosting SillyTavern

This template runs `ghcr.io/sillytavern/sillytavern:latest` — the project's own official Docker image — unmodified. What this template adds sits entirely in the start command: relocating SillyTavern's stateful directories onto a single Railway volume, and seeding a real password onto the app's default account before the server ever starts accepting connections. Nothing about how SillyTavern itself behaves is changed.

That last part matters more than it might sound. SillyTavern's multi-user account system creates a `default-user` account on first boot, and by design it has no password — the app has no way to know what you'd want it to be. Every reference template for SillyTavern on Railway documents this with some version of "change the password as soon as you're able to access ST." That's a real security gap for however long it takes you to notice the warning, log in, and act on it. This template closes that window entirely: a password is generated automatically and applied via SillyTavern's own recovery tooling before your instance is reachable by anyone.

## Common Use Cases

- **Roleplay and creative writing**: character cards, persistent personas, and rich world-info injection for long-running stories
- **A personal AI assistant**: SillyTavern's prompt engineering tools and command system work just as well for utility chat as for roleplay
- **Group chats**: multiple AI characters interacting with you and each other in the same conversation
- **Experimenting with local or API-based LLMs**: SillyTavern is backend-agnostic — point it at OpenAI, Anthropic, a local Kobold/text-generation-webui instance, or dozens of other supported providers

## Dependencies for SillyTavern Hosting

Running this template needs nothing beyond the container itself and one persistent volume — SillyTavern has no external database or separate services to wire up. The one thing you bring yourself is an LLM backend to actually talk to (an API key for a hosted provider, or a self-hosted model server reachable from Railway), which you configure from inside SillyTavern's own settings after you log in.

### Deployment Dependencies

- [SillyTavern's Source Code](https://github.com/SillyTavern/SillyTavern) — AGPL-3.0
- [SillyTavern's Official Documentation](https://docs.sillytavern.app/)
- [SillyTavern's Official Docker Image](https://github.com/SillyTavern/SillyTavern/pkgs/container/sillytavern)

### Implementation Details

The official image expects several independently-writable directories — `config`, `data`, `plugins`, and `backups` — which doesn't map cleanly onto Railway's one-volume-per-service model. This template's start command handles that on first boot: each directory gets moved onto the volume (mounted at `/home/node/app/persist`) and replaced with a symlink, so from the app's perspective nothing changed, but everything now persists across redeploys — characters, chat logs, config edits made through the UI, and any plugins you install.

After that relocation, the start command runs SillyTavern's own `recover.js` script — a tool that ships in the image for exactly this purpose, normally run by hand via SSH — against the `default-user` account, setting its password to the auto-generated value in `SILLYTAVERN_ADMIN_PASSWORD`. Only then does it hand off to the image's own entrypoint. The whole sequence adds a few hundred milliseconds to boot time and runs identically on every restart, so it's a no-op after the first successful run.

Every configuration value SillyTavern reads from `config.yaml` can also be set as an environment variable, using a naming convention (`SILLYTAVERN_<KEY>`) built into SillyTavern itself rather than something this template layers on top. That's what sets networking (`SILLYTAVERN_LISTEN`, `SILLYTAVERN_WHITELISTMODE`) and the account system (`SILLYTAVERN_ENABLEUSERACCOUNTS`) — no custom config templating, no `envsubst` step, just the app reading its own environment.

## Why Deploy SillyTavern on Railway?

Railway is a singular platform for deploying your infrastructure — no server management, no manually wiring up TLS or a reverse proxy, and volumes that persist without you thinking about backup strategy on day one. For a personal LLM frontend like SillyTavern, that removes the entire "how do I expose this safely and keep my chat history" problem, leaving you to actually configure the thing and talk to your characters.

Beyond the infrastructure, this template specifically closes a real gap: deploying SillyTavern on your own infrastructure means you're responsible for securing it, and "responsible" shouldn't mean "remembered to read the warning in the README before someone else found the URL." Automating the password step doesn't just save a manual action — it removes an entire class of "I'll get to it later" security debt.

## What Was Verified

This wasn't verified by watching the deploy log turn green — it was verified by actually logging in. Deployed fresh (new volume, no build cache), the `SILLYTAVERN_ADMIN_PASSWORD` value was pulled directly from the running deployment's variables and used to authenticate as `default-user` against the live instance, which returned a successful session. `GET /login` returns `200` directly; `GET /` correctly 302-redirects an unauthenticated visitor to `/login`, which is SillyTavern's own intended behavior with accounts enabled.

Persistence was verified across a redeploy as well — the volume-backed symlinks for `config`, `data`, and `plugins` survive a full container restart, so character cards and chat history aren't lost the next time you push a change or Railway cycles the deployment.

One deliberate choice worth calling out: this template ships without a Railway healthcheck path configured. `GET /` redirects when accounts are enabled, which Railway's healthcheck doesn't follow and treats as a failure — a real, reproducible issue independent of anything specific to this template's setup. `/login` does return `200` correctly when tested directly, but didn't reliably satisfy Railway's own deploy-success gate in testing on this account, and the underlying reason wasn't fully pinned down after tracing through SillyTavern's request-handling code without finding an obvious cause. Since the container's own liveness — confirmed stable across many redeploys with no crashes — is a solid enough signal here, the template relies on that instead of forcing a healthcheck path that adds fragility without adding real confidence.

## Frequently Asked Questions

### Why doesn't the default account have a password by default?
It's how SillyTavern's own account system works — a `default-user` account is created on first boot with no password, since the app has no way to know what you'd want set. This template is what actually fixes that, rather than just warning you about it.

### Is my instance ever reachable without a password?
No — the password is set before the server starts accepting connections, using SillyTavern's own recovery script. There's no boot-then-set-later window.

### Can I change the password afterward?
Yes, from inside SillyTavern's own account settings once you're logged in. The auto-generated value is just what gets you in the door the first time.

### Does this template modify SillyTavern's code?
No. It's the unmodified official image — the only difference from a bare `docker run` is the start command's setup sequence, which relocates directories and runs a script the image already ships with.

### Why is there no Railway healthcheck configured?
`GET /` redirects when accounts are enabled, and Railway's healthcheck doesn't follow redirects — a genuine, reproducible mismatch. Rather than force a healthcheck path that adds a point of failure without solving that mismatch cleanly, this template relies on container liveness, which has been stable across every tested deployment.

### Where can I download SillyTavern?
Source is on GitHub at [github.com/SillyTavern/SillyTavern](https://github.com/SillyTavern/SillyTavern). Use this template to deploy the official image with the account-security gap closed automatically, in one click.

Source for this template's docs: https://github.com/shruti060701/sillytavern-railway
