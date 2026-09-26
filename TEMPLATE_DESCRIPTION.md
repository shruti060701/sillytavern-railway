## Template Titles

**Railway Title:** `SillyTavern`
**Railway Description:** `SillyTavern — official image, auto-secured admin password, verified live`
**Spreadsheet Title:** `SillyTavern (LLM Frontend, Password Auto-Fixed)`
**GitHub Description:** `SillyTavern on Railway — official AGPL-3.0 image, deployed directly, with the default account's missing password fixed automatically on first boot.`

---

# Deploy and Host SillyTavern on Railway

SillyTavern is an open-source, locally-installed frontend for text-generation LLMs, image-generation engines, and TTS voice models — deep customization over prompts, personas, and world info, built by a community of AI hobbyists since 2023. This template deploys the official image directly and fixes the one real gap every SillyTavern-on-Railway template documents but doesn't actually solve: the default account ships with no password.

## About Hosting SillyTavern

This template runs SillyTavern's own official Docker image unmodified, with a start command that relocates its stateful directories onto a single Railway volume and seeds a real admin password before the server starts accepting requests. Nothing about SillyTavern's core behavior is changed — only the first-boot state it lands in.

## Common Use Cases

- Roleplay and creative writing with LLM-driven characters
- A personal AI assistant/chatbot with advanced prompt and persona controls
- Group chats between multiple AI characters and yourself
- Running your own LLM frontend against any compatible backend (local or API-based)

## Dependencies for SillyTavern Hosting

- Official image: [ghcr.io/sillytavern/sillytavern](https://github.com/SillyTavern/SillyTavern/pkgs/container/sillytavern) (AGPL-3.0)
- One persistent volume, for chats, characters, config, and installed plugins

### Deployment Dependencies

- [SillyTavern's Source Code](https://github.com/SillyTavern/SillyTavern)
- [SillyTavern's Official Documentation](https://docs.sillytavern.app/)

### Implementation Details

The start command does three things in order: relocates `config`, `data`, and `plugins` onto the mounted volume via symlinks (so a single Railway volume covers everything the official multi-directory image expects), runs SillyTavern's own `recover.js` script to set a real password on the `default-user` account, then hands off to the image's own entrypoint. All configuration — networking, accounts, login behavior — is set through SillyTavern's own built-in `SILLYTAVERN_<KEY>` environment-variable override mechanism, not a custom templating layer.

## Why Deploy SillyTavern on Railway?

Railway is a unified platform for deploying your infrastructure without hand-managing servers, TLS, or reverse proxies. For SillyTavern specifically, that means one less place for the "wait, is this instance actually password-protected?" question to go unanswered — the persistent volume and public networking are pre-wired, and the account gap is closed automatically instead of left as a note in the README.

## What Makes This Template Different

Verified by an actual login, not just a healthy-looking deploy log: fresh volume, no cache, the auto-generated `SILLYTAVERN_ADMIN_PASSWORD` value pulled straight from the deployment's variables and used to log in successfully as `default-user`. Every other SillyTavern template found on Railway documents the missing-password issue as something you have to remember to fix yourself after your first login — this one closes that window before the server ever starts serving requests.

## Frequently Asked Questions (FAQs)

### Why does the default account have no password on other SillyTavern templates?
SillyTavern's own account system creates a `default-user` account with an empty password on first boot — by design, since it has no way to know what password you'd want. Most templates leave this as a manual post-deploy step.

### How does this template fix that?
SillyTavern ships its own recovery script (`recover.js`) that can set a password on any account non-interactively. This template runs it automatically on first boot, using a value auto-generated into the `SILLYTAVERN_ADMIN_PASSWORD` variable — no manual step required, though you're free to change it from inside SillyTavern's UI afterward.

### Does this modify SillyTavern itself?
No — it's the unmodified official image. Only the startup sequence (state relocation + one script run) differs from a bare `docker run`.

### Where can I download SillyTavern?
Source is on GitHub at [github.com/SillyTavern/SillyTavern](https://github.com/SillyTavern/SillyTavern). Use this template to deploy it correctly configured, with the password gap closed, in one click.
