# Vela handoff

Written 2026-10-16 by the agent that ran the first design and research session. Read this
before anything else. Everything below was verified against live sources on that date unless
marked "unverified". Dates and version numbers go stale; re-check anything load-bearing.

## 1. State of the repo

| File | What it is |
|---|---|
| `CLAUDE.md` | Rules for agents working here. Binding. Mac-local Tauri 2 POC, Rust holds everything, webview display-only, Phala for inference only, stop-and-ask list. |
| `prd.md` | Product PRD, Mac-local MVP. A separate "PRD v2 (mobile, cloud)" and an "Architecture" doc exist outside the repo and are referenced but not checked in. |
| `plan.md` | A brief addressed to an agent: "plan a proof of concept, produce a plan only, don't write code until I approve." Milestones M0–M5. Not yet executed. |
| `planv0.md` | The approved plan for week 1: run stock OpenClaw with Phala as a custom provider on the real inbox before building anything. Details in section 6. |
| `HANDOFF.md` | This file. |

There is no code. The directory was not a git repository at the time of writing. Before
pushing: `git init`, add a `.gitignore` that excludes `experiments/**/receipts.jsonl`,
`*.log`, `.env*`, and any OAuth client JSON. CLAUDE.md forbids committing credentials or real
email data; logs must use message ids and counts, never bodies.

## 2. The product in one paragraph

A personal agent that reads Gmail and Google Calendar, triages, drafts replies and proposes
actions, where privacy is enforced by construction. v0 runs on the user's Mac. The only remote
call carrying user content beyond its source service is inference, which goes to Phala's
attested confidential AI gateway with zero retention and a signed receipt per response.
Credentials never enter the model's context. Every action that leaves the device passes hard
rules in code, an AI auditor, and for anything sent, Touch ID approval. A later v2 moves the
same design into Phala confidential VMs with an iOS app, on-chain timelocked upgrades, and a
phone-signed approval chain.

## 3. Decisions made this session, with rationale

1. **v0 is Mac-local; the cloud design is v2.** The PRD was rewritten from the cloud version
   to the Mac version during the session. The architecture is portable if the worker, vault,
   auditor, event log and approval payload are separate components with explicit interfaces.
2. **The agent proposes; effects are gated.** Restrictions go on effects, not cognition. The
   model gets a full loop, rich read tools, memory it writes itself, a heartbeat, and skills.
   The only hard boundary is that side effects outside the Mac go through the vault, which acts
   only on approved proposals. An earlier, over-restricted design (code-driven playbooks, model
   as a schema function, two write ops, no memory writes) was rejected by William as something
   nobody would use. Do not resurrect it.
3. **Do not fork OpenClaw. Use it as a dependency through its extension points.** Upstream
   moves weekly; a diverged fork falls behind security patches. Tool policy, MCP servers,
   custom providers and plugins cover everything needed. Rebuilding the runtime from scratch
   was also rejected: the loop is small but compaction, sessions, memory search, heartbeat,
   streaming, retries and provider quirks are months of work.
4. **Before building, run stock OpenClaw on Phala for a week** against the real inbox to learn
   whether an attested open-weight model inside OpenClaw's runtime is a good assistant. If it
   is, the product is the privacy layer. If not, the limit is the model and no runtime fixes it.
5. **Week-1 experiment decisions by William:** chat surface local only (TUI and Control UI, no
   channels); Gmail read plus create drafts, send blocked at two layers; runs on the main
   MacBook; days 6–7 calibrate against a frontier model on the real inbox, knowing real mail
   leaves the attested path on those days.
6. **Three build-phase defaults proposed, not yet confirmed by William:** Touch ID called from
   Rust through LocalAuthentication bindings rather than a Tauri plugin; web search through
   Brave's API, chat only, key in Keychain; auditor through Phala on a different model family
   from the drafter. All three sit behind CLAUDE.md's stop-and-ask line.

## 4. Agentic design: final position

- **Loop.** A real agent loop with read tools over mail, calendar, contacts, web, and its own
  memory, plus propose tools for draft, reply, send, label, archive, event. Grants auto-execute
  reversible proposals. Sends need Touch ID.
- **Memory.** The agent writes its own memory. Writes carry a provenance label and are rendered
  back as data. Standing instructions are user-authored only. That single rule closes the
  OpenClaw HEARTBEAT.md attack class, where a web page wrote itself into the instruction file.
- **Learning signal.** Edits the user makes to a draft before sending are stored and retrieved
  when drafting similar replies. Same for what gets archived unread and what gets a reply
  within the hour.
- **Voice.** Retrieve the user's own past replies to the same person or request type into the
  drafting context. Biggest quality lever for an open-weight drafter; the data is local.
- **Proactivity.** A heartbeat loop with a budget, driven by the agent, not by code.
- **Security sits in four places.** The vault holds tokens and executes only approved
  proposals. A short list of hard rules in code: security mail withheld from the model, new
  recipients ask, sends ask, daily caps. Every proposal cites its sources so the auditor and the
  approval card see what it drew on. Standing instructions have user provenance only.
- **Auditor.** Single-shot, no tools, different model family from the drafter. Inputs: the
  request, the rules, the full outgoing payload, the cited sources with trust tiers, a digest of
  the tool trace. Output: allow, ask, block, one line. Probabilistic; hard rules and approval
  are the backstop.
- **Evals.** Use a frontier model as judge and red-team generator in the harness only, on
  synthetic fixtures. Never in the product.

## 5. Recommended v0 architecture (post-experiment)

- OpenClaw gateway as a local process: loopback only, no channels, no ClawHub, tool profile
  `minimal` plus MCP, memory and web search; exec, browser, filesystem, message and nodes denied.
  Pinned to the extended-stable channel.
- Rust vault exposed to OpenClaw as an MCP server over stdio: mail search and read, calendar
  read, web search, and the propose tools. Tokens never leave Rust. A send tool returns
  "pending approval".
- Rust inference proxy on loopback: OpenClaw's provider points at it; Rust verifies the gateway
  attestation, sets attested-routing and zero-retention flags, verifies the receipt, forwards to
  Phala. Phala's `@phala/pi-provider-aci` does this at the provider layer for Pi, the runtime
  OpenClaw embeds; it may drop in directly (unverified).
- Auditor, proposal gate and Touch ID in Rust. Tauri UI talks to Rust; Rust talks to the
  gateway WebSocket for chat and sessions.
- CLAUDE.md will need one amendment William must approve: the agent loop and plaintext context
  live in a Node process that holds no credentials and reaches the world only through Rust.

## 6. Week-1 experiment

Full plan in `planv0.md`. Summary: day 0 setup (Phala key, Google Desktop OAuth client in
Testing status, gog CLI, a zero-dependency Node proxy that injects the Phala provider flags and
logs receipt headers, OpenClaw config fragment, workspace files, smoke tests including negative
tests that send and shell are unavailable); days 1–5 on Phala models with a model rotation
(GLM 5.3, Kimi K3, Qwen 3.5 397B), a 20-task list built on day 1, a daily log, and three
injection probes from day 3; days 6–7 frontier calibration; day 7 report with soft thresholds
and a decision rule (best Phala model within roughly 80% of the frontier run and clearing
thresholds means build v0 around OpenClaw).

Artifacts to create under `experiments/openclaw-week/`: `README.md`, `openclaw.config.json5`,
`phala-proxy.mjs`, `verify-receipts.mjs`, `workspace/{USER,AGENTS,HEARTBEAT}.md`, `tasks.md`,
`log.md`, `report.md`. None of this is product code; no new dependencies.

## 7. Research findings

### 7.1 OpenClaw (as of 2026-10-16)

**Project.** MIT, TypeScript and Swift. Renamed Warelay → CLAWDIS → Clawdbot → Moltbot →
OpenClaw (Jan 30, 2026). Creator Peter Steinberger joined OpenAI Feb 14–15, 2026. OpenClaw
Foundation announced Jul 8, 2026; OpenAI is a donor; openclaw.ai states it is not an OpenAI
product. 247k GitHub stars by Mar 2026. 2.0 = v2026.8.1 released Aug 30, 2026 after 106
releases in 230 days. Extended-stable/LTS channel announced Jul 30. Security audit via OpenAI's
"Patch the Planet" Sep 21. OpenClaw Enterprise (OCE) announced Sep 29, pre-1.0, 1.0 expected
later 2026: multi-tenancy, sandboxing, LLM-based reviews, swappable harness/model/sandbox,
docker-compose or Kubernetes. Blog Sep 25: "Microsoft Autopilot is built on OpenClaw". Tencent
and Z.ai run OpenClaw-based services.

**Security record.** Texas A&M analysis of 470 advisories
([arXiv 2603.27517](https://arxiv.org/html/2603.27517v2)): structural finding is per-layer
enforcement with no unified policy boundary; skills run at operator-level trust; exec allowlist
relies on lexical parsing and was bypassed three ways; `system.execApprovals.set` was reachable
through `node.invoke`, chaining with SSRF and token exfiltration to RCE; container isolation
depended on Docker defaults. CVE-2026-25253 one-click RCE patched in 2026.1.29. Censys found
21,639 exposed instances on Jan 31, 2026. ClawHub malicious skills: Koi reported 341 of 2,857;
Snyk reported 1,467; VirusTotal scanning added, maintainers say it is "not a silver bullet".
Unit 42 documented a web page writing instructions into HEARTBEAT.md. `inputProvenance`
(`external_user | inter_session | internal_system`) added after GHSA-w5c7. The Register's 2.0
review (Aug 31): secret store not encrypted at rest, sandbox off by default. Docs state the
gateway is "not a hostile multi-tenant boundary". China's MIIT warned about misconfiguration;
state enterprises restricted in Mar 2026.

**Architecture.** Gateway: WebSocket control plane on `127.0.0.1:18789`, token auth by default,
sessions, routing, cron, hooks. Channels: 15–20 messaging platforms, each with allowlists and
DM pairing. Agent runtime on Pi Agent Core. Nodes: macOS menu bar app, iOS, Android, watchOS;
"peripherals, not gateways"; `node.invoke` commands for exec, camera, screen, voice,
notifications. Skills: SKILL.md, ClawHub, Skill Cards (Jun 1). Plugins: `api.registerTool(...)`
with `contracts.tools` in the manifest.

**Config keys verified from docs.**
- Install: `npm install -g openclaw@latest --allow-scripts=openclaw`; Node 24.16+ or 26.1+,
  26 recommended; `openclaw onboard --install-daemon` creates a LaunchAgent; `openclaw doctor`,
  `openclaw security audit`, `openclaw gateway status`, `openclaw dashboard`, `openclaw` opens
  the TUI. Workspace default `~/.openclaw/workspace`; state in `~/.openclaw/state/openclaw.sqlite`.
- Custom provider: `models.providers.<id>` with `baseUrl`, `apiKey`, `api: "openai-completions"`,
  `models: [{id, name, reasoning, input, cost, contextWindow, contextTokens, maxTokens}]`,
  `headers`, `request.{headers,auth,proxy,tls,allowPrivateNetwork}`. A custom `baseUrl` trusts
  that exact origin. **No documented way to inject extra request body fields**, hence the proxy.
- Tool policy: profiles `minimal` (session_status, gateway update only), `coding`, `messaging`,
  `full`. `tools.allow` / `tools.deny` / `alsoAllow`; groups `group:runtime|fs|sessions|memory|
  web|ui|automation|messaging|nodes|agents|media|openclaw|plugins`; MCP via `bundle-mcp` or exact
  tool names or server globs; per agent under `agents.entries.<id>.tools`; deny wins; policy is
  applied before the model call so denied schemas are never seen.
- Exec: `tools.exec.mode: deny|allowlist|ask|auto|full`; approvals `security: deny|allowlist|
  full`, `ask: off|on-miss|always`, `askFallback`; allowlists per agent in SQLite; auto mode =
  policy, then reviewer model with a bounded packet, then human.
- MCP: `mcp.servers.<name>` with `command`, `args`, `cwd`, `enabled`, `connectionTimeoutMs`,
  `requestTimeoutMs`, `toolFilter.{include,exclude}`; `openclaw mcp add`, `openclaw mcp doctor
  <name> --probe`; failed starts back off 30s doubling to 10m.
- Memory: `USER.md`, `MEMORY.md`, `memory/YYYY-MM-DD.md`, optional `DREAMS.md`; tools
  `memory_search` (hybrid vector plus keyword), `memory_get`, `intent`; **default embeddings
  provider is OpenAI** and must be changed via `memory.search.provider` (adapter ids include
  `openai-compatible`, or a configured `models.providers.<id>`); pre-compaction memory flush on
  by default (`agents.defaults.compaction.memoryFlush.{enabled,model}`); dreaming on by default
  (`plugins.entries.memory-core.config.dreaming.enabled`).
- Heartbeat: `agents.defaults.heartbeat.{every,prompt,model,timeoutSeconds,lightContext,
  isolatedSession,session,target,directPolicy,activeHours}`; default `30m`; the runtime does not
  read HEARTBEAT.md, it reads "monitor scratch" (`openclaw cron scratch <jobId>`; `openclaw
  doctor --fix` imports HEARTBEAT.md); agent replies `NO_REPLY` or calls `heartbeat_respond`;
  requires `cron.enabled`.
- Web: `tools.web.search.{enabled,provider:"brave",maxResults}` with key under
  `plugins.entries.brave.config.webSearch.apiKey` or `BRAVE_API_KEY`;
  `tools.web.fetch.provider` defaults to auto-detect (firecrawl shown), so treat fetch as
  remote unless confirmed local.
- iMessage channel exists as a plugin (`@openclaw/imessage`) using the local Messages database.

**Phala's Pi provider.** `@phala/pi-provider-aci` 0.7.3 (2026-10-13), MIT, provider extension
for `pi-coding-agent` (peer deps `@earendil-works/pi-ai`, `pi-tui`). Verifies the gateway
workload, pins the attested TLS key, verifies every receipt; `ACI_BASE_URL`,
`ACI_ACCEPTED_COMPOSE_HASHES`, `ACI_ACCEPTED_SESSION_IDS`, `ACI_API_KEY`. Whether it loads in
OpenClaw's embedded runtime is unverified.

**Google access.** `gog` CLI ([openclaw/gogcli](https://github.com/openclaw/gogcli)): `brew
install openclaw/tap/gogcli`, macOS 15+. Desktop OAuth client JSON via `gog auth credentials
set`; `gog auth add <acct> --services gmail,calendar`. Default Gmail scopes `gmail.modify`,
`gmail.settings.basic`, `gmail.settings.sharing`; `--gmail-scope send|read-send`; `--readonly`;
root `--gmail-no-send`. Tokens in the OS keychain (99designs/keyring) or an encrypted file for
headless use. **`gog mcp`** is a stdio MCP server, read-only by default; writes need
`--allow-write` plus `--allow-tool <names>`; send needs `--allow-gmail-send`; delete needs
`--allow-gmail-delete` and `--force`. Read tools: `gmail_search`, `gmail_get_message`,
`gmail_get_thread`, `gmail_list_drafts`, `gmail_get_draft`, `gmail_list_labels`,
`calendar_events`. Draft tools: `gmail_create_draft`, `gmail_update_draft`.

### 7.2 Phala Confidential AI

- Endpoint `https://inference.phala.com/v1`, OpenAI-compatible, billed from the Phala Cloud
  workspace balance; key under Private AI → API Keys.
- The ACI gateway runs in a TEE, publishes an attestation report, signs a per-response receipt.
  Response headers: `x-receipt-id`, `x-aci-identity`, `x-aci-keyset-digest`. Endpoints:
  `GET /v1/aci/attestation`, `GET /v1/aci/receipts/{id}`. Verification: receipt `workload_id`
  and `workload_keyset_digest` match the attestation; receipt `event_log` contains
  `upstream.verified` with `result: "verified"` and `required: true`.
- Request fields: `provider: {"zdr": true}` (API docs) and `provider: {"aci_verified": true}`
  (Phala's private-inference solutions page, about a year old; may have changed). Model catalog
  flag `is_tee`. `GET /v1/models?zdr=true` returned the same 22 models as without the filter and
  no model record carries a zdr field; treat the filter as unverified.
- Catalog on 2026-10-16: 22 models, all `is_tee: true`, 17 advertise tools. Prices per 1M
  tokens in/out: `z-ai/glm-5.3` $1.40/$4.40 (1M ctx), `qwen/qwen3.5-397b-a17b` $0.55/$3.50,
  `moonshotai/kimi-k3` $3.00/$15.00 (1M ctx), `openai/gpt-oss-120b` $0.15/$0.60,
  `google/gemma-4-31b-it` $0.15/$0.46, `z-ai/glm-5.3-flash` $0.15/$0.50,
  `meta-llama/llama-3.3-70b-instruct` $2/$2, `deepseek/deepseek-v3.2` $1/$1,
  `moonshotai/kimi-k2.6` $1.09/$4.60. Model records list upstream `providers`, e.g.
  `["phala", "privatemode"]`, so a second attested upstream already exists behind the gateway.
- Embeddings (`GET /v1/embeddings/models`): `qwen/qwen3-embedding-8b` and
  `sentence-transformers/all-minilm-l6-v2`, both TEE.
- Private AI Gateway (GitHub `Dstack-TEE/private-ai-gateway`): runs on dstack, routes across TEE
  providers by proof properties, optional field-level E2EE to the attested gateway key
  (`x-e2ee-version: aci-1`), receipts cover request hash, route, upstream verification, response
  hash.
- `dcap-qvl` Rust crate (Phala) 0.6.1: pure-Rust SGX/TDX quote verification, PCCS or offline
  collateral, defaults to Phala's PCCS; CVE-2026-22696 fixed in ≥0.3.9. Also `@phala/dcap-qvl`
  pure JS.
- Pricing: `tdx.small` 1 vCPU / 2 GB at $0.058/h ≈ $42/month plus storage at ~$0.10/GB/month,
  20 GB minimum. Older pages quote $0.069/h ≈ $50.
- Proof-of-Cloud: Phala's Oct 28, 2025 response to TEE.fail. "Attestation + Provenance +
  Physical Security"; OVH and bare metal in Tier-3+ datacenters; hyperscaler integrations in
  progress. Builds on the Flashbots / TU Munich "Proof of Cloud" paper (DCEA, vTPM-anchored
  chassis binding). Vendor claim; the client cannot verify it today.

### 7.3 dstack and on-chain governance

- dstack: Docker-compose workloads in TDX CVMs; `dstack-kms` releases keys only to attested
  workloads; `dstack-gateway` reverse proxy; audited (zkSecurity). dstack-cloud runs on GCP TDX
  with vTPM and on AWS Nitro.
- Contracts: `DstackKms` (OS image allowlist, app registration) and per-app `DstackApp`
  (`allowedComposeHashes`, device ids; `ComposeHashAdded` event).
- Prior art for the PRD v2 timelock: Longjing PR #117 adds a Safe plus 7-day TimelockController
  and a guardian role that can only remove hashes, through a wrapper that owns the DstackApp.
  Wulong issue #48 proposes the same with each release removing the previous hash.
- Open issues: #1287 KMS root keys cannot rotate; #1293 an app can never be deregistered, so a
  compromised app-owner key can re-add a hash; #1298 allowlist setters accept `bytes32(0)`.
  The multisig that owns the DstackApp is the crown jewel.

### 7.4 Google OAuth and CASA

- Restricted Gmail scopes: `gmail.readonly`, `gmail.compose`, `gmail.insert`, `gmail.modify`,
  `gmail.metadata`, `gmail.settings.basic`, `gmail.settings.sharing`, full `mail.google.com`.
  `gmail.send` is sensitive, not restricted. Reading mail needs a restricted scope, so CASA
  (Tier 2, annual) applies at production regardless of where content goes. Vendors estimate
  4–12+ weeks.
- Google's Workspace user-data policy lists "providing generative AI summaries" as an approved
  Gmail use. The real policy question is whether attested zero-retention inference counts as a
  processor under Limited Use; counsel or Google's reviewer answers that.
- Official OAuth docs: a project in Testing status issues refresh tokens that expire after 7
  days (except basic profile scopes); 100 refresh tokens per Google account per client id. Test
  user cap is on the consent screen. A vendor claims a Sep 13, 2026 change removed the 100-user
  cap for CASA-assessed apps; unverified.
- For installed apps Google says the client secret "is obviously not treated as a secret".

### 7.5 Touch ID in Tauri 2 on macOS

- Official `tauri-plugin-biometric` was iOS and Android only. PR #3264 adding macOS Touch ID via
  LocalAuthentication through `objc2` was merged into `v2` on Oct 12, 2026; planned release
  2.5.0, not confirmed shipped.
- Community `tauri-plugin-biometry` (Choochmeque, v0.2.8) supports macOS and bundles its own
  Keychain storage, which collides with CLAUDE.md's rule that only `src-tauri/src/vault/`
  touches the Keychain.
- Either plugin means the webview initiates the prompt and receives the result, contradicting
  CLAUDE.md's "Touch ID approval is triggered from Rust". Recommendation: call `LAContext`
  from Rust in the approvals module via `objc2-local-authentication` (pattern in `ex_local_auth`:
  fresh context per call, result over a channel). `tid-rs` exists but is stale (2022). New Cargo
  dependency, needs approval.

### 7.6 Competitors

- **Instinct** (Spear Street Technology): TechCrunch late Aug 2026. Terms before Aug 26 granted
  a perpetual, irrevocable, sub-licensable license including for training; revised Aug 26;
  plain-text email storage; inbox summary after Google access revoked; an email sent without
  approval; terms allow entering binding agreements for the user; reported $250M raise.
- **Meta Muse**: launched Sep 8, 2026 with a per-user "Secure VM", a "Sentinel" agent that
  reviews everything leaving the VM, Muse Spark 1.3. Meta's David Singleton admitted access is
  technically possible today. A Confidential VM with a user-held key, published binaries, a
  transparency log and source access for select auditors is promised "later 2026", with Moxie
  Marlinspike. Training on sanitized records is opt-out and on by default. Topped the US iPhone
  free chart Sep 21.
- **OpenAI Dots**: launched Sep 29, 2026 (DevDay) on GPT-6 Astra for Pro and Business Premium.
  Custom Rules (act, act if pre-approved, ask, hand off), a separate auto-review model, fixed
  limits on unasked work (no messages, no browser or computer control, password changes stay
  with the user). OpenAI says protections "do not eliminate" malicious-instruction risk.
- **OpenAI Codex auto-review** (alignment blog, Apr 30, 2026): most actions run in the sandbox
  without review; of the roughly 7% escalated, about 99.1% are approved; 99.93% effective rate;
  200x fewer human interruptions; reviewer on GPT-5.4 Thinking; red-teaming has beaten it.
- **Norton Family Assistant** (Gen Digital, Jun 4, 2026, beta): per-user cloud instance, no
  training, sampled quality evaluation with opt-out; connects Gmail, calendars, messaging.
  Policy-based like Muse's Secure VM.

### 7.7 TEE hardware risk

TEE.fail (Georgia Tech, Purdue, Synkhronix; disclosed Oct 28, 2025): DDR5 memory-bus
interposer under $1,000, extracts attestation keys from SGX, TDX and SEV-SNP with
"UpToDate" TCB, which also breaks NVIDIA GPU confidential computing's trust anchor. Needs
physical access and root. Intel and AMD say physical attacks are out of their threat model.

### 7.8 Prior art on injection-resistant agents

- CaMeL ("Defeating Prompt Injections by Design", IEEE SaTML/S&P 2026): privileged planner
  writes a restricted program, quarantined model extracts values without tools, capability tags
  flow with data, policies run at tool-call time. 77% of AgentDojo tasks under attack with no
  successful injections, vs 84% undefended.
- Progent: per-call least-privilege rules; indirect injection success 39.9% → 1.0% on AgentDojo.
- FIDES: taint tracking with integrity and confidentiality labels.
- "CaMeLs Can Use Computers Too" (Jan 2026): single-shot planning for computer-use agents;
  branch-steering remains.
- OpenClaw auto mode reviewer packet: command, argv, cwd, env key names not values, parser
  analysis; everything treated as data; defers to a human when the data tries to instruct.
- Position taken for Vela: the planner will read untrusted mail, so the defence is on effects
  (proposals, rules, auditor, approval) and data flow (provenance labels), Progent and FIDES
  style, not full CaMeL isolation.

## 8. Outstanding review findings on `prd.md` and `plan.md`

Not yet applied. Apply or consciously reject.

- "The only remote call is model inference" is false as written; Gmail, Calendar and web search
  are remote. Reword to "the only remote call carrying user content beyond its source service",
  and route search queries through hard rules and the auditor.
- Layer 3 overclaims: "untrusted content can't steer the agent" should be "untrusted content
  can't cause an action the user didn't approve".
- Memory is mentioned once in the PRD and never designed; CLAUDE.md's scope omits it. Define it
  with the provenance rule or drop it from the MVP wording.
- CASA question is mis-framed (see 7.4). Also note the 7-day refresh token in Testing status.
- Proof-of-Cloud is a vendor claim the Verify screen cannot show; keep in residual risk.
- The attestation "nonce" in the PRD is not documented on Phala's endpoint; the stronger design
  is local quote verification with `dcap-qvl` plus compose-hash pinning, which Phala's own SDK
  does.
- The PRD has two duplicate layer tables and empty "sources from the original draft" bullets.
- Provenance needs a concrete taxonomy. Drafts should cite the threads they drew on so the
  auditor can flag cross-thread content and the approval card can show it.
- Residual risks to add: the app binary is the real root of trust; the DstackApp owner key.
- `plan.md` brief: "tauri-plugin-biometry" conflicts with CLAUDE.md (see 7.5); the model spike
  needs a milestone; red teaming should start as fixtures in M3, not M5; M2 should include the
  proposal, grant, provenance and run-record schemas; the "only remote call" line repeats.

## 9. Things to verify early

- Phala still accepts `provider.aci_verified`; otherwise use `zdr` plus receipts.
- OpenClaw accepts model ids containing slashes under a custom provider (e.g.
  `phala/z-ai/glm-5.3`); otherwise alias in the proxy.
- Heartbeat alert delivery when no channel is configured (local-only surface).
- `memory.search.provider` config shape for an OpenAI-compatible endpoint through the proxy.
- Whether `@phala/pi-provider-aci` loads inside OpenClaw's embedded Pi runtime.
- `tools.web.fetch` default provider is remote (firecrawl); keep off unless confirmed local.
- Google OAuth Testing status: 7-day refresh token expiry hits on day 7.
- Whether `gmail.modify` is needed for `gmail_create_draft` or `gmail.compose` suffices.

## 10. Mistakes made this session

- Designed an over-restricted agent (code-driven playbooks, model as a schema function, two
  write operations, no memory writes). William's reaction: nobody would use it. The correction
  is section 4. Put restrictions on effects, not cognition.
- Claimed the runtime was "a week of Rust". It is months. Hence OpenClaw as a dependency.
- Initially doubted `provider.aci_verified` existed; it is on Phala's solutions page. Still
  verify it works today.
- Spent time on security-architecture detail when William wanted agentic product design. Lead
  with what makes the agent good; the gate makes it safe.

## 11. Conventions for the next agent

- CLAUDE.md is binding. Stop and ask before touching `vault/`, `approvals/`, `inference/`,
  `auditor/`, `capabilities/`, `tauri.conf.json`, or adding any Cargo or npm dependency or Tauri
  plugin. When in doubt choose the more restrictive privacy option and ask.
- Never put email bodies, subjects, addresses, prompts or model output in logs, errors, tests
  or commits. Ids and counts only. Fixtures are synthetic.
- William wants opinions, not surveys; high-level agentic design over infrastructure detail;
  short, direct answers. He pushes back hard when the product would be boring. Build the thing
  people would use, then make it safe.
- Every Phala request sets attested routing and zero retention and fails closed. The only
  exception is the explicitly approved frontier calibration on days 6–7 of the experiment.

## 12. Sources

OpenClaw: [security analysis of 470 advisories](https://arxiv.org/html/2603.27517v2),
[security docs](https://docs.openclaw.ai/gateway/security), [tools](https://docs.openclaw.ai/tools),
[tool policy](https://docs.openclaw.ai/gateway/config-tools/tool-policy),
[custom providers](https://docs.openclaw.ai/gateway/config-tools/custom-providers),
[built-in tools](https://docs.openclaw.ai/gateway/config-tools/built-in-tools),
[exec approvals](https://docs.openclaw.ai/tools/exec-approvals), [MCP](https://docs.openclaw.ai/tools/mcp),
[heartbeat](https://docs.openclaw.ai/gateway/heartbeat), [memory](https://docs.openclaw.ai/concepts/memory),
[memory config](https://docs.openclaw.ai/reference/memory-config), [nodes](https://docs.openclaw.ai/nodes),
[install](https://docs.openclaw.ai/install), [getting started](https://docs.openclaw.ai/start/getting-started),
[iMessage](https://docs.openclaw.ai/channels/imessage), [blog](https://openclaw.ai/blog),
[auto mode](https://openclaw.ai/blog/safer-than-yolo-auto-mode-for-exec-approvals),
[Enterprise](https://openclaw.ai/blog/openclaw-enterprise), [Wikipedia](https://en.wikipedia.org/wiki/OpenClaw),
[Unit 42 supply chain](https://unit42.paloaltonetworks.com/openclaw-ai-supply-chain/),
[gogcli](https://github.com/openclaw/gogcli), [@phala/pi-provider-aci](https://registry.npmjs.org/@phala/pi-provider-aci).

Phala and dstack: [Confidential AI API](https://docs.phala.com/phala-cloud/confidential-ai/confidential-model/confidential-ai-api),
[overview](https://docs.phala.com/phala-cloud/confidential-ai/overview),
[private inference page with aci_verified example](https://phala.com/solutions/private-ai-inference),
[Private AI Gateway](https://phala.com/posts/private-ai-gateway-verified-private-ai-compute),
[instance types](https://cloud.phala.com/about/instance-types), [pricing](https://docs.phala.com/phala-cloud/pricing),
[TEE.fail response / Proof-of-Cloud](https://phala.com/posts/phala-ddr5),
[Proof of Cloud paper](https://arxiv.org/html/2510.12469v1), [dcap-qvl](https://docs.rs/dcap-qvl),
[dcap-qvl CVE](https://vulert.com/vuln-db/dcap-qvl-has-missing-verification-for-qe-identity),
[dstack attestation](https://docs.phala.com/dstack-cloud/attestation-integration),
[Longjing timelock PR](https://github.com/w3hc/longjing/pull/117), [Wulong issue](https://github.com/w3hc/wulong/issues/48),
[dstack #1287](https://github.com/Dstack-TEE/dstack/issues/1287), [#1293](https://github.com/Dstack-TEE/dstack/issues/1293),
[#1298](https://github.com/Dstack-TEE/dstack/issues/1298).

Google and Tauri: [OAuth 2.0 docs](https://developers.google.com/identity/protocols/oauth2),
[Workspace user data policy](https://developers.google.com/workspace/workspace-api-user-data-developer-policy),
[restricted scopes and CASA](https://developer.nylas.com/docs/provider-guides/google/google-verification-security-assessment-guide/),
[CASA 2026](https://deepstrike.io/blog/google-casa-security-assessment-2025),
[Tauri biometric macOS PR](https://github.com/tauri-apps/plugins-workspace/pull/3264),
[tauri-plugin-biometry](https://github.com/Choochmeque/tauri-plugin-biometry), [tid-rs](https://docs.rs/tid-rs).

Competitors and prior art: [Instinct coverage](https://aiweekly.co/alerts/techcrunch-instinct-ais-always-on-personal-agent-draws-privacy-alarm-over),
[Muse Confidential VM](https://forkast.news/when-the-building-owner-doesnt-have-the-key-metas-muse-confidential-vm-and-the-third-trust-architecture/),
[Muse training opt-out](https://captaincompliance.com/?p=13651), [Dots security](https://neuraltrust.ai/blog/chatgpt-agent-security),
[Dots launch](https://thenextweb.com/news/openai-dots-always-on-ai-agents-cloud-computers-devday),
[OpenAI auto-review](https://alignment.openai.com/auto-review),
[Norton Family Assistant](https://newsroom.gendigital.com/2026-06-04-Norton-Introduces-Family-Assistant,-the-Secure-AI-Agent-Built-to-Help-Families-Manage-the-Chaos-of-Modern-Parenting-in-the-Digital-Age),
[TEE.fail](https://bleepingcomputer.com/news/security/teefail-attack-breaks-confidential-computing-on-intel-amd-nvidia-cpus),
[CaMeL](https://www.computer.org/csdl/proceedings-article/satml/2026/785900a587/2jZks3ksGT6),
[out-of-band defenses with Progent and FIDES](https://arxiv.org/html/2606.26479v1),
[CaMeLs Can Use Computers Too](https://arxiv.org/html/2601.09923v1),
[Tinfoil attestation](https://docs.tinfoil.sh/verification/attestation-architecture).
