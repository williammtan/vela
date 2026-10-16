# Private Agent POC

macOS desktop app built with Tauri 2. A personal agent that reads Gmail and Google Calendar,
drafts replies and proposes actions. User data stays on this Mac; the only remote call is model
inference to Phala Private AI. Privacy is the product: when in doubt, choose the more
restrictive option and ask me.

## Commands
- `npm run dev:web`: UI in the browser with mock IPC (`VITE_MOCK_IPC=1`) at http://localhost:1420
- `npm run tauri dev`: full app
- `npm run check`: tsc + eslint + vitest
- `cargo test && cargo clippy -- -D warnings` (run in `src-tauri/`)
- `npm run test:e2e`: Playwright against `dev:web`, screenshots in `e2e/__screenshots__/`

## Architecture boundary (non-negotiable)
- The webview (`src/`) is display-only. It never holds tokens, keys, raw email HTML or API keys.
- All secrets, network calls, model calls, tool execution, the auditor and approvals live in
  Rust (`src-tauri/src/`).
- The UI talks to Rust only through commands in `src-tauri/src/commands.rs`, wrapped by
  `src/lib/ipc.ts`. Add a command only when a screen needs it, and return display data only.

## Never
- Never add Tauri plugins (especially shell, fs, or the JS-side http plugin) without asking.
- Never loosen the CSP in `tauri.conf.json` or add permissions in `capabilities/` without asking.
- Never render email or web HTML (no `dangerouslySetInnerHTML`, no iframes in the main window).
  Render plain text.
- Never put user content (email bodies, subjects, addresses, prompts, model output) in logs,
  errors, panics or test snapshots. Use IDs and counts.
- Never touch Keychain items outside `src-tauri/src/vault/`.
- Never call a model except through `src-tauri/src/inference/`. Every request sets
  `provider.aci_verified = true` and `provider.zdr = true`. Never fall back to a non-attested model.
- Never send, delete or share anything unless: hard rules pass, then the auditor returns a
  verdict, then Touch ID approval is triggered from Rust with the recipient and subject in the
  Touch ID reason string.
- Never pass raw email bodies or web pages to the auditor.
- Never commit real credentials, tokens or real email data. Fixtures are synthetic.

## Stop and ask before changing
`src-tauri/src/vault/`, `approvals/`, `inference/`, `auditor/`, `capabilities/`,
`tauri.conf.json`, and any new Cargo or npm dependency.

## UI conventions
- Use shadcn/ui (`npx shadcn@latest add <name>`). Don't edit `src/components/ui/` except theme tokens.
- Every feature has fixtures in `src/fixtures/` covering empty, normal, long and error states.
- `/dev/states` renders every screen in every fixture state (dev builds only).
- Mac feel: system font, compact density, no marketing-style layouts.

## Verifying your work
- After any UI change: run `dev:web`, open `/dev/states` with the Playwright MCP, screenshot every
  affected state, confirm the browser console is clean, and tell me what changed.
- Don't call a task done until `npm run check`, `cargo test` and `clippy` pass.
- Touch ID and Keychain can't run in browser mode. Tell me exactly what to test by hand in
  `npm run tauri dev`.

## POC scope
In: Gmail read, draft and send-with-approval; Calendar read; web search; chat; approvals;
activity log; verify screen.
Out: everything else. Ask before adding scope.