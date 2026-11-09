# antigravity-cloak-fixed

A small fix for [Magpie](https://usemagpie.ai) gateway middleware on Antigravity accounts, distributed as a **local path plugin**.

## The problem

Antigravity answers requests whose system prompt reads as Claude Code / Claude Agent SDK (and some Codex GPT-5 or Hermes Agent personas) with a fake `429 Resource has been exhausted`, unrelated to actual quota. The upstream community middleware [@magpie-community/middleware-antigravity-cloak](https://github.com/magpie-community/plugins/tree/main/packages/antigravity-cloak) (v0.1.2) works by inserting a zero-width space (U+200B) into those identity phrases before the request is sent to Antigravity only.

v0.1.2's `anthropic` branch cloaks only the top-level `body.system`. Claude Code interleaves `role: "system"` / `role: "developer"` messages **inside the `messages[]` array**, so the identity phrases still reach Antigravity untouched and the fake 429 returns. The built-in `prompt=default` rewriting and the `extraWords` option do not help, because neither touches those interleaved messages.

## What this fork changes

Exactly one thing, in `onProvider` under `case "anthropic"`:

```js
if (Array.isArray(body.messages)) for (const m of body.messages) {
  if (m && (m.role === "system" || m.role === "developer"))
    m.content = typeof m.content === "string" ? text(m.content) : parts(m.content, text)
}
```

Everything else (the word list, the U+200B cloaking, the `accounts` default of `antigravity`, `chat` / `responses` / `gemini` handling, `extraWords`, `off`) is unchanged from upstream 0.1.2. Other providers still receive the prompt exactly as the agent wrote it.

## Install

Magpie supports installing a middleware from a local file path. Clone or download this repository, then:

```sh
magpie plugin rm middleware-antigravity-cloak   # if the upstream package is installed
magpie plugin add /absolute/path/to/antigravity-cloak-fixed.middleware.js
magpie plugin list
```

Using a path outside `node_modules` (rather than patching the installed npm package) keeps the fix intact across `magpie plugin update`, and removing the upstream spec means `plugin update` has no defective package left to reinstall.

Verified with Magpie `0.1.3423`. Removing the upstream plugin needs `magpie` to reload its plugin host; confirm with `magpie plugin list` that only this path plugin remains.

## Options

```json
{ "extraWords": [], "accounts": ["antigravity"], "off": false }
```

- `extraWords`: additional phrases to cloak, matched case-insensitively.
- `accounts`: provider account kinds it acts on; defaults to `antigravity`.
- `off`: leave every request untouched.

## Verification

The published middleware file is byte-identical to the instance running locally (`sha256 2703f1472abd67703525a3317ebca378c738c7615950cee3ddfe0dbe2d7bb7eb`). It was checked with `node --check`, 8 no-network unit cases, and end-to-end A/B probes through `127.0.0.1:3425/v1/messages`:

- plugin ON: decisive `role:system` identity phrase → 200; array parts, developer role, the newer Claude Code line, top-level system, user-only and assistant-only phrases → 200.
- plugin OFF: the same decisive request → `429 Resource has been exhausted`; plugin ON again → 200.
- A ~470 KB Claude-Code-shaped request (137 messages, 15 system/developer messages, two cached `system` blocks, `anthropic-beta` header) → 200.

## Caveats (from upstream, still apply)

- This changes what Google is sent so that a request it means to refuse isn't refused. It may be against Antigravity's terms, and the account could be banned. Use an account you can afford to lose.
- Google can change what it matches at any time; add new phrases to `extraWords` or turn the plugin off if the refusal returns.

## License and attribution

MIT. Derived from [@magpie-community/middleware-antigravity-cloak](https://github.com/magpie-community/plugins) v0.1.2 (MIT, `Copyright (c) 2026 magpie-community contributors`), itself after [hexiaolv/magpie-antigravity-cloak](https://github.com/hexiaolv/magpie-antigravity-cloak) (MIT, `Copyright (c) 2026 hexiaolv`). See `LICENSE`.
