Plan a proof of concept. Produce a plan only; don't write code until I approve it.

## Product
A private personal agent for macOS. It reads the user's Gmail and Google Calendar, drafts
replies and proposes actions. Data stays on the Mac. The only remote call is model inference
to Phala Private AI. Nothing is sent without Touch ID approval.

## Stack (fixed)
Tauri 2, React + Vite + TypeScript, shadcn/ui + Tailwind, Rust core (tokio), rusqlite with
SQLCipher, macOS Keychain, tauri-plugin-biometry for Touch ID, Phala Private AI
(OpenAI-compatible, tool calling).

## Read first
CLAUDE.md. It overrides anything in this prompt.

## Milestones (adjust if needed, and say why)
- M0 Scaffold: Tauri + React + shadcn, mock IPC layer, browser dev mode, `/dev/states` page,
  Playwright set up, checks wired.
- M1 UI on mock data: chat, approvals queue, approval dialog, activity log, verify screen.
- M2 Rust core: encrypted store, Keychain wrapper, Phala client with attestation check,
  attested routing, zero retention and receipt check.
- M3 Agent loop and read-only tools: Gmail search/read, Calendar read, web search, activity events.
- M4 Actions: draft and send through hard rules, auditor verdict and Touch ID approval enforced in Rust.
- M5 Red team: synthetic malicious emails; the agent must never send, leak or reveal secrets.

## For each milestone, give
Goal; modules and files touched; the tests that prove it works; how you'll visually verify UI
changes; anything touching vault/, approvals/, inference/, auditor/, capabilities/ or the CSP,
flagged for my review.

## Done means
Typecheck, lint, vitest, cargo test and clippy pass, and I've reviewed screenshots of every UI state.

If something blocks the plan, ask me up to 3 questions. Otherwise pick sensible defaults and
list them as assumptions.