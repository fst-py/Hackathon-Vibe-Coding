# Pirate Launch Kit — {{PROJECT_NAME}}

> **This file is the single source of truth for where we are.** The agent reads it on every turn to decide what to do next, and updates it the moment a step lands. Don't edit it by hand mid-flow — tell the agent what to change.

**Started:** {{DATE}}
**Technical level:** {{TECHNICAL_LEVEL}}/10
**Permission mode:** _{{PERMISSION_MODE}}_

---

## What this kit is for

Take {{PROJECT_NAME}} from an empty folder to a **live custom-domain URL** running an agentic SaaS shell — landing, auth, streaming chat — without any manual config or copy-pasted API keys. You came here to build your idea, not to wire scaffolding. The kit is the runway. Your real work starts after the runway exists.

## Definition of done

You hit the finish line when **all six** of these are true on your custom domain:

1. The landing page renders for an anonymous visitor.
2. Sign-up creates a real Clerk production user → row lands in Neon.
3. A real Stripe live-mode charge succeeds (verified by a €0.50 refunded test).
4. A welcome email from your verified domain lands in the inbox (not spam).
5. The chat shell streams a real response from Claude via Vercel AI Gateway.
6. PostHog production project records every event.

## Deliberately out of scope (v1)

The kit gives you the runway, not the product. These are **your** job once the kit is done:

- Your agent's character, tools, memory, personality
- Your specific product features and workflows
- Your visual design beyond the bare shell
- Your pricing tiers and Stripe products
- Your marketing site beyond the landing
- Anything that makes your product *yours*

Not yet in the kit either: Sentry / dedicated error monitoring, CI/CD beyond Vercel's built-in deploys, multi-user / team / org features, mobile native shell.

---

## Phase 0 — Onboarding & Setup

- [ ] Technical level captured
- [ ] Permission mode chosen
- [ ] Working directory ready
- [ ] Runtimes + service CLIs installed (Node 20+, Bun, git, gh, vercel, stripe)
- [ ] Vercel plugin loaded (covers Phase 1; service-specific tooling wired at point-of-use in Phase 2)
- [ ] Git initialized
- [ ] `.gitignore` covers secrets (`.env*`, `.vercel`)
- [ ] Pre-push secret-scan hook installed

## Phase 1 — Frontend + Agent + First Deploy

- [ ] Next.js 16 scaffolded (Tailwind, TypeScript, Turbo)
- [ ] shadcn/ui initialized with starter components + dialog
- [ ] AI Elements installed and trimmed to used components
- [ ] `<TooltipProvider>` wrapping layout
- [ ] Vercel project linked → AI Gateway works automatically via OIDC token
- [ ] AI SDK v6 + Workflow SDK installed (`@vercel/chat-sdk` skipped — not a real package)
- [ ] Workflow stub at `src/workflows/example.ts`; `pageExtensions` includes `js`/`jsx`
- [ ] Geist fonts mapped via `@theme inline` in `globals.css` (verified by inspecting computed font-family)
- [ ] `next-themes` wired with `defaultTheme="system"`; `ModeToggle` in header; light/dark/system all verified by toggling
- [ ] `/api/chat` route wired with password gate
- [ ] `/api/chat-auth` cookie-setting route wired
- [ ] Password modal in `page.tsx` unlocks chat input
- [ ] Local chat works with password
- [ ] GitHub remote created and pushed
- [ ] Vercel project linked
- [ ] **Live URL: https://__________.vercel.app**
- [ ] Live URL chat works with password
- [ ] Playwright golden path passes against live URL

### Phase 1 Access Password (until Clerk swaps it in 2.1)

> Share this with anyone you want to let into the chat right now. Phase 2.1 retires it and replaces it with real sign-in. **Do not commit this password to git** — it's already in `CHAT_ACCESS_PASSWORD` on Vercel.

- **Password:** `__________________`
- **Live URL:** see above
- _Status: active until Clerk auth lands in Phase 2.1_

## Phase 2 — Backend Layer (each service ends with redeploy + live URL smoke)

### Clerk (auth) — replaces the password gate from Phase 1
- [ ] Account: _existing | new (Marketplace)_
- [ ] Wired with dev keys
- [ ] **Password gate retired** (`/api/chat-auth` deleted, `/api/chat` checks Clerk auth, `CHAT_ACCESS_PASSWORD` removed from Vercel, password line in CHECKLIST struck through)
- [ ] Webhook handler `/api/webhooks/clerk` wired with signature verification
- [ ] Webhook registered in Clerk Dashboard against live URL
- [ ] Redeployed to live URL
- [ ] Sign up works on the live URL (no password)
- [ ] Chat works for signed-in users

### Neon + Drizzle (database)
- [ ] Account: _existing | new (Marketplace)_
- [ ] Database provisioned, first migration applied
- [ ] `getOrCreateUser()` lazy-sync helper wired
- [ ] Clerk webhook calls `getOrCreateUser()` (idempotent — `onConflictDoNothing`)
- [ ] Redeployed to live URL
- [ ] Sign up on live URL → row appears in Neon

### Resend (transactional)
- [ ] Account: _existing | new (Marketplace)_
- [ ] Welcome email template + `sendWelcomeEmail` helper
- [ ] Wired into `getOrCreateUser()` after row creation
- [ ] Redeployed to live URL
- [ ] Sandbox send delivers to test inbox

### Stripe (test mode)
- [ ] Account: _existing | new_
- [ ] Test product created via Stripe Sandbox MCP
- [ ] Checkout route + webhook handler wired with signature verification
- [ ] Stripe Dashboard webhook registered against live URL
- [ ] Redeployed to live URL
- [ ] Test checkout completes on live URL; `stripe_customer_id` populated

### PostHog (analytics)
- [ ] Account: _existing | new (Marketplace)_
- [ ] Client init with `person_profiles: "identified_only"`
- [ ] Events captured: `landing_viewed`, `signed_up`, `welcome_email_sent`, `checkout_completed`
- [ ] Redeployed to live URL
- [ ] All events visible in PostHog Live Events

- [ ] Phase 2 end-to-end smoke on live URL (signup → email → checkout → analytics)

## Phase 3 — Production Cutover

### Custom domain
- [ ] Domain: _________
- [ ] DNS verified
- [ ] HTTPS active

### Clerk (production)
- [ ] Production instance created
- [ ] Custom domain in allowlist
- [ ] Production keys on Vercel
- [ ] Production webhook updated

### Stripe (live mode)
- [ ] Account activated (KYC complete)
- [ ] Live product created
- [ ] Restricted key (`rk_live_…`) created with minimum scopes
- [ ] Live keys on Vercel
- [ ] Live webhook updated
- [ ] Real test charge succeeded and refunded

### Resend (verified domain)
- [ ] Sender domain added
- [ ] DNS records (SPF, DKIM, DMARC) verified
- [ ] Welcome email sender updated
- [ ] Test email lands in inbox (not spam)

### PostHog (production)
- [ ] Production project created
- [ ] Production key on Vercel

- [ ] Final smoke passed on custom domain
- [ ] `SHIPPED.md` written
- [ ] Pushed to GitHub `main`

---

## Divergences from kit

> When you pick a tool the kit doesn't recommend, the agent logs it here.

_(none yet)_

---

## Notes

> Free-form scratch space.

_(none yet)_
