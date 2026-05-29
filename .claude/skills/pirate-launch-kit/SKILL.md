---
name: pirate-launch-kit
description: Scaffold an opinionated agentic-SaaS starter kit (Next.js, Vercel, Clerk, Neon, Stripe, Resend, PostHog, Vercel AI SDK, Workflow SDK, Chat SDK) from an empty folder all the way to a live production URL with custom domain. Use this skill whenever the user wants to start a new agentic SaaS project from scratch, mentions "new project", "new app", "starter kit", "scaffold", or "from empty folder" — even if they don't explicitly name the kit.
license: MIT
allowed-tools: Bash, Read, Write, Edit, WebFetch
metadata:
  author: bensufiani
  version: "1.0.0"
---

# Pirate Launch Kit

Take a non-technical vibe coder from `mkdir new-project` to a **live custom-domain URL** running an agentic SaaS shell — landing page, auth, streaming chat — without ever asking them to download something from a website, edit a config file by hand, or copy-paste an API key.

The user came here to build their idea, not to wire scaffolding. Your job is the runway. Their job starts after the runway exists.

## North Stars (non-negotiable)

These rules trump anything else in this skill. Re-read them when you feel pulled to break one.

1. **Lean forward, always.** After every step, name the next concrete action and ask the user to confirm or redirect. Never end a turn with "I'm done with this step, what's next?" — always end with "Next I'll do X — confirm or redirect?"
2. **Never tell the user to download anything.** If a runtime, CLI, or tool is missing, install it yourself via the terminal (Homebrew, curl, npm, npx). The only things the user does are: make decisions, sign in to dashboards when an account first needs to be created, and approve actions that touch money or production.
3. **Vercel CLI is the env var vault — there is no other.** Every secret lives on Vercel and is pulled from there. The skill never edits `.env.local` by hand and never asks the user to paste a key into a file. The full flow is in the "Security defaults" section below — read it, follow it.
4. **Marketplace > CLI > MCP.** When wiring a third-party service:
   - First choice: **Vercel Marketplace** integration (one click, keys auto-injected to all environments, lowest friction).
   - Second choice: the service's official **CLI** (`stripe`, `clerk`, `neonctl`, etc.) → then `vercel env add` to push the resulting key into the vault.
   - Third choice: the service's **MCP server**, but only when MCP outclasses the CLI for the specific job (e.g. Stripe MCP for product creation).
5. **Confirm before irreversible or money-touching actions.** Live Stripe charges, custom-domain DNS changes, dropping a database, deploying to a production domain. Everything else: just do it (if YOLO mode was approved) and report.
6. **The CHECKLIST file is the only source of truth.** No hidden state, no separate JSON. On every turn, read `docs/pirate-launch-kit/CHECKLIST.md` before deciding what to do next. Update it the moment a step lands.
7. **Calibrate tone in the first turn and never forget it.** The user told you their technical level on a 0–10 scale. 0–3 = explain every term inline. 4–6 = explain jargon on first use. 7–10 = terse, skip explanations.
8. **When the user diverges from the kit, respect them but warn.** If they want Supabase instead of Neon, Auth.js instead of Clerk, etc.: warn that the kit's playbook diverges from here, offer best-effort, note the divergence in CHECKLIST under "Divergences from kit". Never refuse.

## Security defaults (non-negotiable, low-friction)

A non-technical founder can leak credentials in seconds without realising it. The kit has to make leaking *harder than not leaking*. None of the rules below should slow the user down — they're built into the workflow.

### The Vercel CLI is the env var vault

Every secret lives on Vercel. Period. The skill follows this flow for **every** key, every time:

```bash
# Add a secret (skill does this — never asks user to paste into a file)
vercel env add KEY_NAME development   # then prompt the skill (not the user) for the value
vercel env add KEY_NAME preview
vercel env add KEY_NAME production

# Pull secrets down to local (this is the ONLY way .env.local gets populated)
vercel env pull .env.local
```

The Vercel Marketplace integrations (Clerk, Neon, Resend, PostHog, AI Gateway) write directly to all three environments automatically — no manual `vercel env add` needed. For services without Marketplace integration (e.g. Stripe), use `vercel env add` to push the value the CLI just produced.

**Never** edit `.env.local` by hand. **Never** add a secret only to `.env.local` without pushing it to Vercel first. If the local file drifts from the vault, `vercel env pull .env.local --force --yes` rewrites it from truth.

### `.gitignore` is checked in Phase 0

The Next.js scaffold ships a correct `.gitignore`. The skill verifies on entering Phase 0 that all of these are present:

```
.env
.env.local
.env*.local
.vercel
```

If anything is missing, the skill adds it before the first commit.

### No secrets in source

If the skill ever finds itself about to write a literal `sk_...`, `pk_live_...`, `whsec_...`, `rsk_...`, or any other key-shaped string into a `.ts`, `.tsx`, `.js`, `.json`, or `.md` file, **stop**. That's a `process.env.X` reference instead. Always.

### Public vs server keys

The `NEXT_PUBLIC_*` prefix in Next.js means the value is bundled into the browser. Only put truly public values there (Clerk publishable key, Stripe publishable key, PostHog client key). Anything that starts with `sk_`, contains `secret`, or grants write access lives **without** the prefix and is referenced server-side only.

### Webhook signatures are always verified

Every `/api/webhooks/*` route in the kit verifies the signing secret of the incoming request before doing anything with the payload. Clerk uses `svix`, Stripe uses `stripe.webhooks.constructEvent`. The skill writes these verifications into the route from the start — never as a follow-up.

### Restricted Stripe keys for production

When promoting Stripe to live mode in Phase 3, the skill creates a **restricted key** (`rk_live_...`) scoped to the minimum surfaces the app uses (Checkout Sessions, Customers, Webhooks). The full secret key (`sk_live_...`) stays in the dashboard, not in the app.

### No PII in PostHog by default

The kit's PostHog setup uses `person_profiles: "identified_only"` and disables session recordings on form fields by default. If the user wants richer analytics later, they opt in — they don't opt out of an over-collection default.

### A pre-push secret scan

Before any `git push`, the skill grep-scans staged files for the patterns above (`sk_live_`, `sk_test_`, `whsec_`, `rsk_`, `clerk_secret`). If any match, it aborts the push, surfaces the file/line, and offers to move the value into Vercel and replace the source with `process.env.X`.

These defaults are baked into Phase 0 — the user never has to think about them.

## The session-start ritual (run on every fresh session)

This happens **before** any phase work. Don't skip it even if the user seems eager to "just build."

1. **Calibration.** Ask: *"On a scale of 0 to 10, how deep is your technical knowledge — where 0 is 'I've never opened a terminal before today' and 10 is 'I write code professionally'? I'll calibrate how I talk to you for the rest of this session."*
2. **Permission mode.** Recommend Claude Code's auto-edit + auto-approve (YOLO) mode in plain language: *"For the smoothest ride, I'd recommend turning on auto-approve mode in Claude Code (Shift+Tab twice). I'll move much faster and ask you less. I'll always pause and ask before anything that costs money or touches your accounts. Sound good, or do you want me to ask before every action?"* Respect their choice. If they say no to YOLO, warn that the next ~90 minutes will involve a lot of permission prompts.
3. **Working directory check.** Run `ls -la` in the current directory. If it has anything besides `.git`, ask: *"This folder isn't empty. Should I proceed inside it anyway, or would you like to start in a fresh subfolder?"*
4. **Resume check.** If `docs/pirate-launch-kit/CHECKLIST.md` already exists in the working directory, **do not** start over. Read it, find the first unchecked item, and continue from there with: *"Looks like we left off at Phase X step Y. Want me to continue, or restart from scratch?"*
5. **Phase 0 entry.** If this is a fresh start, read `references/0-onboarding.md` and execute it. That phase ends with CHECKLIST.md written to `docs/pirate-launch-kit/`.

## State tracking — CHECKLIST.md is the spine

Every project the skill touches gets these two files in the user's working directory:

```
docs/pirate-launch-kit/
├── CHECKLIST.md    # the only live doc — manifesto + scope + ticking checkpoints, all in one
└── SHIPPED.md      # written at the end of Phase 3 with the final URLs
```

CHECKLIST.md uses GitHub-flavored task list syntax (`- [ ]` and `- [x]`). When a step lands, **immediately** flip its box and append the relevant value (URL, account email, key location). Example:

```markdown
- [x] Vercel project created — pirate-skills.vercel.app
- [x] Clerk dev instance wired — keys auto-injected via Marketplace
- [ ] Neon database provisioned
```

The user sees this file evolving in their editor — it's part of the experience. Don't treat it as scratch state.

## Phase routing

Each phase has a dedicated reference file. **Read it when you enter the phase**, not before. Don't load all references upfront.

| Phase | Goal | Reference |
|-------|------|-----------|
| 0 | Onboarding — calibration, permissions, runtimes + service CLIs, **Vercel plugin install** (covers all of Phase 1), working docs, git + secret-scan hook | `references/0-onboarding.md` |
| 1 | Frontend + agent + first deploy — scaffold + AI Gateway + AI SDK + Workflow + Chat SDK, **chat streams Claude responses behind a hardcoded password the kit generates**, live `*.vercel.app` URL exists by the end | `references/1-frontend.md` |
| 2 | Backend — Clerk **swaps the password gate for real auth** → Neon → Resend → Stripe → PostHog, dev/test keys, redeploy + smoke against the live URL after each service, lazy-sync user creation + webhooks point at the live URL from day one | `references/2-backend.md` |
| 3 | Production cutover — custom domain, all services on prod keys, restricted Stripe key, real €0.50 charge verified | `references/3-production-cutover.md` |

**The structural insight:** deploying isn't a phase, it's a primitive. Phase 1 ends with a live URL **and a working chat behind a password** — the magic moment lands ~30 minutes in, not 90. Every Phase 2 service ends with a redeploy + smoke against that same URL. Phase 2.1's first move is to rip out the password gate and replace it with Clerk auth, so the URL the user just shared with friends works for real signups too. The user watches their app come alive on the internet one piece at a time, and any "works locally but not on Vercel" surprise surfaces instantly instead of accumulating until a final big-bang deploy.

**Install AI assistance at point-of-use, not all at once.** Phase 0.5 installs only the Vercel plugin (Next.js / shadcn / AI Elements / AI SDK / AI Gateway / Workflow SDK / Chat SDK — everything Phase 1 needs). Service-specific tooling lands the moment you wire that service: Stripe Sandbox MCP gets registered in Phase 2.4 with the real `sk_test_…` key (registering earlier with a placeholder fails); Clerk and Stripe quickstart patterns get `WebFetch`'d in Phases 2.1 and 2.4 (no skills published on skills.sh yet). Neon, PostHog, and Resend use CLI only — their MCPs add no value for setup-only tasks.

**Template** (copied verbatim into the user's working directory at the start of Phase 0):
- `references/checklist-template.md` → `docs/pirate-launch-kit/CHECKLIST.md`

## Account-creation defaults

For every third-party service in Phase 2 and Phase 3, follow this script:

1. *"Do you already have a [Service] account, or should I set one up for you?"*
2. **If new account:**
   - Try Vercel Marketplace first (open the marketplace URL, walk them through the OAuth/signup flow once — the rest is handled automatically including env injection).
   - Fall back to direct signup only if Marketplace doesn't cover this service. Walk them through the dashboard signup, then immediately authenticate the local CLI.
3. **If existing account:**
   - Use the service's CLI to authenticate (`stripe login`, `clerk login`, `vercel login`, etc.).
   - Pull keys via the CLI or via `vercel env pull`. Never ask the user to find a key in their dashboard and paste it.
4. **MCP servers:** prefer CLI over MCP for one-shot setup tasks. Use MCP only where it genuinely outclasses the CLI for an ongoing job (e.g. Stripe MCP is excellent for product creation; the Stripe CLI is better for `stripe listen`).

## The "lean forward" cadence

After every meaningful action, follow this pattern:

1. **Tick the box** in CHECKLIST.md.
2. **One-sentence summary** of what just landed (with the actual value — URL, key location, account email).
3. **Propose the next action** as a concrete sentence: *"Next I'll wire Neon via Vercel Marketplace — that adds Postgres with one click and injects the connection string into Vercel automatically. Confirm or redirect?"*

Do not list options. Do not ask "what would you like to do?" Pick the next correct step from the playbook and propose it.

## When errors hit

The user is non-technical. Translate. When something fails:

1. **Say what's wrong in one plain sentence.** ("Vercel can't reach Neon because the connection string isn't set yet.")
2. **Say what you're doing about it.** ("I'll re-pull the env vars from Vercel and try again.")
3. **Skip the technical autopsy.** Don't paste the stack trace unless they ask.

If you hit a problem you genuinely can't solve from inside the skill: be honest, propose a workaround, and offer to log a divergence note in CHECKLIST so they can come back to it.

## Common pitfalls

| Issue | Fix |
|-------|-----|
| User says "I have a Vercel account" but their CLI isn't authenticated | Run `vercel login` and wait for browser auth |
| Marketplace integration created but env vars not visible locally | Run `vercel env pull .env.local` |
| `next dev` fails because port 3000 is already in use | `lsof -ti:3000 \| xargs kill -9` then retry — don't ask the user to "find the other thing" |
| User on Linux instead of macOS | Use `apt`/`curl` instead of `brew`. Detect via `uname -s`. |
| User on Windows (WSL or PowerShell) | Use `winget` for Node/git/gh; `npm i -g` for everything else. Detect early. |
| User doesn't have Homebrew on macOS | Install it first, silently: `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"` |
| User hits Clerk free-tier MAU limit during testing | Note in CHECKLIST, don't block — they'll hit it later in real life anyway |
| User hits Stripe live-mode KYC requirement | Pause Phase 3, save state, tell them to complete KYC in their Stripe dashboard, resume |

## When the user wants to stop or pause

If they say *"let's stop here"* / *"continue tomorrow"* / *"I need a break"*:

1. Update CHECKLIST.md with the exact next action you would have taken (write it as the first unchecked item).
2. Commit + push if a git remote is wired (so they don't lose state).
3. Tell them: *"Saved. When you come back, just open Claude Code in this folder and say 'continue the launch kit' — I'll pick up from CHECKLIST."*

## When you finish (end of Phase 3)

Write `docs/pirate-launch-kit/SHIPPED.md` with:
- The live custom-domain URL
- Links to every service dashboard the user now controls
- One paragraph: *"You shipped. The runway is built. The clock you've been running against hasn't even started — this is the moment to start building the part only you can build."*

Then propose the post-kit handoff: *"From here, the rest of the work is yours. Want me to help you draft the first three product features, or are you ready to take it from here?"*

## Related skills

- `clerk-setup` — invoked from Phase 2 for Clerk wiring detail
- `vercel-env` — invoked from Phase 2/4 for env var hygiene
- `playwright-cli` — invoked from Phase 1/4/5 for golden-path smoke tests
- `dev-server` — invoked from Phase 1 to manage `next dev`
- `stripe-best-practices`, `stripe-projects` — invoked from Phase 2 for Stripe setup
- `workflow` — invoked from Phase 3 for Workflow SDK examples
