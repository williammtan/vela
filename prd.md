Private Personal Agent — PRD
Oct 15, 2026 · @William Tan
TL;DR
We are building a personal agent that keeps user data private by construction, not by policy. The agent runs locally on the user's Mac, and its only remote call is model inference inside an attested Phala enclave. Credentials never reach the model, and every action that leaves the device passes hard rules, an AI auditor and, for anything sent, a Touch ID approval.
The MVP is a macOS app with a few reference connectors (proposed: email, calendar, web search) built on shared primitives, so new integrations need no new security work.
Problem
Personal agents now ask for full access to email, calendars, messages and accounts, but today's agents protect that data by policy, not by construction. The provider can technically read, keep and act on everything, and users are asked to trust a promise. As a result, users hesitate to give agents the full access that makes them useful.
The evidence so far:
• Instinct (TechCrunch, Aug 24, 2026): testers flagged a perpetual, irrevocable license to train on user data, plain-text email storage, easy phishing, an agent that kept summarizing Gmail after access was revoked, and an email sent without the user's approval.
• Muse (Meta): each user gets a dedicated cloud VM, but Meta says it can still technically access the data. A Confidential VM with user-held keys is promised for later in 2026. Meta may use sanitized inference records for training unless users opt out.
• Dots (OpenAI): each dot runs on an OpenAI cloud computer. Protection comes from auto-review and approvals, not from keeping data away from OpenAI.
The failures fall into three kinds:
1. The company can see, keep or train on your data. This covers the license terms and plain-text storage.
2. Data doesn't go away. The agent kept summarizing Gmail after access was revoked.
3. The agent itself leaks or misbehaves. It was easy to phish and sent messages the user didn't approve.
Open question: hesitancy is anecdotal today. We need a number, such as the share of beta users who decline to connect email, or a short survey.
Sources from the original draft:
• TechCrunch
• X post
Layers of safety and security issues
No current personal agent provides Layers 0 and 1 by construction, and Instinct fails all four. Each layer answers one question:
• Layer 0, Verifiability (can I check it?): every guarantee below is enforced by published code in attested hardware, and the client verifies it.
• Layer 1, Confidentiality (who can see it?): no custodian (us, the infrastructure provider, the model provider) can read user data, and data is used only to serve that user.
• Layer 2, Lifecycle (how long does it exist?): disconnecting stops access immediately, and deletion covers derived data such as summaries and memory.
• Layer 3, Agent integrity (what does the agent do with it?): consequential actions need user approval, data can't flow somewhere new without permission, credentials never enter the model's context, and untrusted content can't steer the agent.
Layer
Instinct
Muse (Meta)
Dots (OpenAI)
Us (MVP)
0 Verifiability
None
Promised with Confidential VM (published binaries, transparency log); not shipped
None announced
Attestation check on every session; open-source app
1 Confidentiality
Fails: perpetual training license, plain-text email storage
Policy only today; may train on sanitized inference records unless opt-out
Policy only; runs on OpenAI's cloud computer
Data stays on device; only inference leaves, to an attested enclave
2 Lifecycle
Fails: kept summarizing Gmail after revocation
Not specified
Not specified
Local delete; disconnect revokes token and deletes connector data
3 Agent integrity
Fails: easy to phish, sent an unapproved email
A separate Sentinel approves anything leaving the VM; credentials hidden from the agent
Auto-review, approvals, saved passwords hidden from the model
Hard rules, AI auditor, Touch ID approvals, credentials in Keychain
Layer
Instinct
Muse (Meta)
Dots (OpenAI)
Us (MVP)
0 Verifiability
✗
✗
✗
✓
1 Confidentiality
✗
✗
✗
✓
2 Lifecycle
✗
✗
✗
✓
3 Agent integrity
✗
✓
✓
✓
✓ = enforced by design. ✗ = missing, only promised, or policy only.
Layer 1 is necessary but not sufficient. A Layer 3 failure leaks data through a legitimate channel, so encryption alone would not have prevented Instinct's worst incidents.
Goals and non-goals
The MVP proves that an agent can be both useful and private by construction.
Goals:
1. Users connect email and calendar without trusting us with their data, because we cannot read it.
2. No action leaves the device without passing hard rules and the auditor, and every send is confirmed by the user.
3. Anyone can verify our privacy claims through open source and attestation.
4. The agent is useful enough for daily triage, drafting and scheduling.
Non-goals for the MVP:
• Running while the Mac is asleep (always-on)
• A mobile app
• Closed frontier models
• Payments, and browser automation with passwords
• Hiding network metadata from Phala (unlinkability)
• Multi-device sync and cloud backup
Target users and user stories
Proposed wedge, to validate: privacy-sensitive professionals who work on a Mac, such as lawyers, founders and executives, journalists, and anyone handling confidential client information. They have the most to lose from a leak and are most likely to pay for privacy.
Core user stories:
1. As a founder, I want my inbox triaged every morning without my email leaving my laptop, except to an attested model.
2. As a lawyer, I want drafted replies to clients that I review and approve before anything is sent.
3. As a user, I want to see everything the agent did, and why.
4. As a user, I want to disconnect an account and know its data is gone.
5. As a technical user, I want to verify that the app I run is the published open-source build.
Solution
A secure and private personal agent that secures the entire stack, so every guarantee is enforced by code and hardware users can verify, not by our policy.
Principles:
• Data stays on the user's device unless a task needs it somewhere else.
• The model gets capabilities, never credentials. It calls tools by name; tools attach the secrets themselves.
• Nothing irreversible happens without the user's signature.
• Every claim is verifiable. Open source, reproducible builds, attestation.
• Integrations are connectors on shared primitives. Each connector declares its tools, what each tool does (read, draft, send) and which domains it may reach. Adding Slack or a bank later needs no new security work.
The full stack:
• Secure inference: Phala Confidential AI, an OpenAI-compatible API served through an attested gateway. Every request requires an attested upstream and zero data retention, and every response carries a signed receipt we verify.
• Secure VM: Phala confidential VM. Not needed for the local MVP; it is the path to an always-on, mobile agent later (see the mobile PRD and cloud architecture tabs).
MVP scope and requirements
Everything runs on the user's Mac except model inference, which goes only to an attested Phala enclave.
The agent only proposes actions. Hard rules, the auditor and the user decide, and tools attach Keychain tokens only after approval.
1. Desktop app (macOS). The agent loop, connectors, memory and activity log run locally. Data is stored in an encrypted local database whose key lives in the macOS Keychain.
2. Inference via Phala. The only remote call. Before each session the app fetches Phala's gateway attestation, bound to a fresh nonce, and sends nothing if it fails. Every request sets attested routing and zero data retention, so it fails rather than reach a provider that isn't attested, and each response's signed receipt must show an attested upstream.
3. Credentials in the Keychain. Accounts connect through the standard native-app OAuth flow. Tools read the token from the Keychain at the moment of the call and attach it themselves; the model only ever sees tool names and results.
4. Connectors. Each declares its tools, what each does (read, draft, send) and the domains it may reach. Proposed MVP connectors: email, calendar, web search.
5. Hard rules (code, not AI). Every send asks the user. Security emails (one-time codes, password resets, security alerts) are withheld from the model. The agent cannot change the rules.
6. AI auditor. Reviews every action that leaves the device before it runs (next section).
7. Approvals. A native dialog shows the exact email, invite or message, confirmed with Touch ID. For the MVP, every send asks the user.
8. Activity log. A local record of every tool call, approval and outgoing request, visible to the user.
9. Disconnect and delete. Disconnecting a connector revokes its token with the provider and deletes its local data.
AI auditor
The auditor is a separate model call that reviews every action leaving the device, after the hard rules and before any approval. The design follows OpenAI's auto-review, which Dots and Codex use to approve or deny actions that cross a boundary.
• Job: catch actions that drift from what the user asked for, or that send data somewhere unexpected.
• Inputs: the user's original request, the user's rules, the proposed action (tool, arguments, recipients or destination) and where the data in that action came from.
• Excluded inputs: raw email bodies and web pages. Otherwise the same prompt injection that tricked the agent can trick the auditor.
• Output: allow, ask the user, or block, with a one-line reason shown to the user.
• Isolation: runs as a separate process; the agent cannot edit its prompt, the rules or its configuration.
• Evaluation: a red-team set of malicious emails and pages that try to make the agent leak or send data. Track the catch rate and how often it interrupts legitimate work. For reference, OpenAI reports its Codex auto-review approves about 99% of the actions it reviews.
• Limits: the auditor is probabilistic. Hard rules and user approvals are the backstop for anything irreversible.
Open question: run the auditor through the same Phala endpoint, or as a small local model on the Mac.
Platform: macOS desktop first
A local agent needs a machine that can run it, so the MVP ships on macOS. A phone companion for approvals comes in v1.
Need
macOS desktop
Mobile (iOS)
Run the agent loop in the background
Yes, whenever the Mac is awake
Heavily restricted by iOS
Reach files, browser and local tools
Yes
Blocked by app sandboxing
Run a small local model (auditor, redaction)
Feasible on Apple Silicon
Tight on memory and battery
Connect accounts with OAuth
Native-app flow, no server needed
Supported
Approve actions quickly
Touch ID dialog, only when at the Mac
Push notification plus Face ID, always at hand
The tradeoff: the agent only runs while the Mac is awake. We accept that for the MVP; always-on is the reason to move to a Phala confidential VM in v2.
How the MVP covers the four layers
The local-first design makes Layers 1 and 2 small. The real work is Layer 3, and the main trust assumption left is the TEE hardware.
Layer
MVP mechanism
Residual risk
0 Verifiability
Attestation check before every session; open-source app with reproducible builds
Users must still trust our release signing; reproducible builds let them check it
1 Confidentiality
Data stays on the device; only inference leaves, to an attested Phala enclave
We trust Intel and NVIDIA hardware; physical attacks such as TEE.fail can forge attestation; Phala sees network metadata (IP, timing, volume)
2 Lifecycle
Local delete; disconnect revokes the token and deletes connector data
Copies stay in connected services (sent email remains in Gmail); local backups such as Time Machine may hold copies
3 Agent integrity
Hard rules, AI auditor, Touch ID approvals, credentials in the Keychain
Approval fatigue; auditor misses; a compromised Mac is out of scope
TEE.fail mitigation: use Phala's Proof-of-Cloud assurance that hardware sits in vetted datacenters, and add a second attested provider (such as Tinfoil) in v1.
Improving privacy over time
Privacy claims are only as strong as the evidence behind them, so we publish the evidence.
• Red teaming. A standing suite of malicious emails and web pages that try to exfiltrate data, send messages, extract credentials or slip past the auditor. Run it on every release and publish the pass rate.
• Open source. The app, hard rules and auditor prompt are public, with reproducible builds so anyone can check that the binary matches the source.
• Visible verification. A "Verified" screen in the app shows that Phala's attested measurement matches the published one.
• External audit and bug bounty before public launch.
FAQ: what happens if X?
Law enforcement serves us a subpoena. We don't hold user content; it lives on the user's Mac. We can only produce what we actually have, such as an account email or billing records. Phala runs inference in an attested enclave without prompt logging, so it has no prompts to hand over either. Authorities can still go to the source services (Google holds the user's email) or to the user's device. The harder case is being compelled to ship a malicious update; reproducible builds, signed releases and a public changelog make that visible. Our legal posture needs counsel's review.
A user's laptop is stolen. The local database is encrypted, with its key in the Keychain, behind the macOS login. We should require FileVault during onboarding.
Phala is compromised or its TEE is broken. If attestation fails, the app refuses to send anything. A physical attack that forges attestation (TEE.fail) is a residual risk, mitigated by Proof-of-Cloud and a second provider in v1.
Phala goes down. The agent pauses anything that needs the model; local data is unaffected.
The agent drafts the wrong email. Nothing is sent without the user's Touch ID approval of the exact message, and the activity log shows what happened.
A malicious email tries to hijack the agent. Hard rules block the non-negotiables, the auditor checks the action without reading the raw email, and any send still needs the user's approval.
Do we or Phala train on user data? No. We never receive it, and every request requires zero data retention at the model provider.
A user disconnects Gmail. The token is revoked with Google and that connector's local data is deleted. Nothing remains on our side because nothing was there.
A user loses their Mac. In the MVP, data lives only on that Mac, with no cloud backup. Encrypted backup is a later decision.
Is the model as good as ChatGPT? Not yet. We use open-weight models, which trail closed frontier models; we benchmark them on our core tasks and say so plainly.
What does Phala see? Network metadata (IP address, timing, request size), not content.
Competitive positioning
Our real competitor on privacy is Meta's Muse Confidential VM, built with Moxie Marlinspike and due later in 2026, not Instinct. Norton also markets its Family Assistant as a capable agent that keeps data private.
Where we win:
• Available and verifiable now, not promised.
• The inference path is attested. Meta hasn't explained how model calls leave its Confidential VM.
• No training at all, rather than training on sanitized records unless the user opts out.
• Local-first and open source, with no ads business behind it.
Where we lose:
• Capability. Open-weight models trail GPT-6 Astra and Muse Spark.
• Always-on. The MVP only runs while the Mac is awake.
• Reach. Mac only at launch.
Success metrics
The MVP succeeds if privacy measurably unlocks access and the agent stays safe while doing real work. Targets are still open.
• Connect rate: share of new users who connect both email and calendar in their first session. This tests the core hypothesis directly.
• Weekly tasks completed per active user.
• Approval acceptance rate. Low means the agent proposes bad actions; near 100% warns of approval fatigue.
• Auditor catch rate on the red-team suite, and its false-interruption rate on real work.
• Week-4 retention.
• Trust survey: share of users who say they would give the agent more access.
Roadmap
Every privacy guarantee ships in v0; capability and convenience come after the public launch gate.
Dates are not set. The gate requires the red-team suite to pass, an external audit, and Google's restricted-scope review.
Open questions
[ ] Which wedge user do we launch with?
[ ] Which confidential model on Phala? Only models in Phala's TEE-capable catalog qualify; benchmark two or three on our 20 core tasks.
[ ] Does the auditor run through Phala or as a small local model?
[ ] Does sending email content to remote inference trigger Google's restricted-scope security assessment (CASA)? Until verified, test mode caps the beta at 100 users.
[ ] How do we measure hesitancy before launch (survey, waitlist data)?
[ ] Pricing, given no data monetization.
[ ] Legal posture on subpoenas, compelled updates and abuse handling.
[ ] Backup and recovery if a user loses their Mac.
[ ] Product name.
Sources
• TechCrunch on Instinct (original draft link) and summary of the report
• X post (original draft link)
• Meta: Introducing Muse
• Review of Muse's privacy boundaries
• Axios on Dots safety
• BetaNews on Dots auto-review
• OpenAI: Auto-review of agent actions
• Phala Confidential AI docs
• TEE.fail coverage and Phala's response
• Norton on Instinct and Family Assistant
• Phala Private AI: trust boundary
• Phala Private AI: attested routing