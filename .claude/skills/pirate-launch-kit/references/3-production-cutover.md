# Phase 3 — Production Cutover

**Goal:** A live custom domain, every service on production keys, real charges possible, real users can sign up.

**You finish this phase when:** the user can hit `https://<their-domain>.com`, sign up, complete a real Stripe charge, and you've written `docs/pirate-launch-kit/SHIPPED.md` summarizing the launch.

## High-stakes warning

This phase contains the irreversible-or-expensive-to-reverse moves. **Confirm before every one:**

- Buying or wiring a custom domain (annual cost, DNS propagation)
- Switching Stripe to live mode (real money charges)
- Changing Clerk to production instance (separate user pool from dev)
- Configuring DNS records on an external registrar (mistakes can take hours to undo)

Before starting Phase 3, say to the user:

*"From here on, decisions cost real money or take real time to reverse. I'll pause and confirm before each irreversible step. If at any point you want to pause and resume tomorrow, just say so — I'll save state and we pick up clean."*

## Step-by-step

### 3.1 Custom domain

**Ask:** *"Do you already own a domain for this project, or should I help you buy one?"*

#### Buying a new domain

Use Vercel Domains (cleanest path — no DNS to configure):

```bash
vercel domains buy <domain>
```

Vercel registers, points DNS at the project, and verifies — all in one step. Confirm price with user before pulling the trigger.

#### Using an existing domain

```bash
vercel domains add <domain>
```

Vercel will print the DNS records the user needs to add at their registrar (likely an A record `76.76.21.21` and/or a CNAME for `www`). Walk them through their registrar UI — say *"This is the only step in the kit where I can't do it for you. Open your domain registrar in another tab. Tell me which one you use and I'll give you the exact clicks."*

Common registrars: GoDaddy, Namecheap, Cloudflare, Google Domains (now Squarespace), Porkbun. Skill should know each one's DNS UI roughly. If unsure, fetch the registrar's DNS docs via WebFetch.

After DNS records are added: `vercel domains inspect <domain>` and wait for verified status. Can take 1–60 minutes.

### 3.2 Promote Clerk to production instance

In Clerk Dashboard:
1. Top-left dropdown → "Create production instance"
2. Add the custom domain to the production instance's allowlist

Get the production keys (`pk_live_...`, `sk_live_...`). The skill writes them to Vercel:

```bash
vercel env add NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY production
vercel env add CLERK_SECRET_KEY production
vercel env add CLERK_WEBHOOK_SECRET production
```

Skill pastes the values itself — never asks the user to. Then update the Clerk webhook endpoint to `https://<custom-domain>/api/webhooks/clerk` and update the new signing secret on Vercel.

⚠️ **Production Clerk users are a different pool from dev.** Tell the user explicitly: *"Heads up — anyone who signed up during testing on the .vercel.app URL is in a different bucket from real production users. Real signups start counting from the moment we flip to the live domain."*

### 3.3 Stripe to live mode

In Stripe Dashboard:
1. **Activate account** → complete KYC if not already done. (This is the one place where the user has to do real work — bank details, ID, business info. If KYC isn't done, pause Phase 3 here, save state, resume when they're activated.)
2. Toggle to "Live mode" (top-right)
3. **Create a restricted key**, not the secret key. Stripe Dashboard → Developers → API keys → Create restricted key. Scope it to: Checkout Sessions (write), Customers (write), Webhook Endpoints (read), Refunds (write). Name it `pirate-launch-kit-live`. The format is `rk_live_...`. The full `sk_live_...` stays in the dashboard, never on Vercel.
4. Get the publishable key (`pk_live_...`)
5. Re-create the test product in live mode

Skill writes keys to Vercel:

```bash
vercel env add STRIPE_SECRET_KEY production       # paste the rk_live_... restricted key
vercel env add NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY production
vercel env add STRIPE_WEBHOOK_SECRET production
```

(`STRIPE_SECRET_KEY` is the env var name the SDK reads, but the value is the restricted key — that's the point.)

Set up the live webhook endpoint at `https://<custom-domain>/api/webhooks/stripe` with the same events as the test webhook. Paste the new signing secret to Vercel.

Run a real test charge: skill suggests €0.50 with a real card (the user's), then refunds it via:

```bash
stripe refunds create --charge=<charge-id>  # or via Stripe MCP
```

Tick: *"Real charge succeeded and refunded — Stripe live wiring is verified."*

### 3.4 Resend verified sending domain

In Resend Dashboard:
1. Domains → Add Domain → enter the user's email-sending domain (often the same as the custom domain, or a subdomain like `mail.<domain>`)
2. Resend prints DNS records: SPF (TXT), DKIM (TXT), DMARC (TXT optional but recommended), MX or Return-Path

Walk the user through adding these at their registrar. Same registrar-specific guidance as 5.1.

After verification (can take 5–60 min), update the sender address in the welcome email template:

```typescript
from: "Welcome <hello@<custom-domain>>"
```

Send a real test email from the new sender to the user's address. Confirm it lands in their inbox (not spam).

### 3.5 PostHog production project

In PostHog Dashboard:
1. Create a new project named `<project>-prod`
2. Copy the project API key

Update Vercel:

```bash
vercel env rm NEXT_PUBLIC_POSTHOG_KEY production  # remove dev key from prod env
vercel env add NEXT_PUBLIC_POSTHOG_KEY production  # add prod key
```

Trigger a redeploy: `vercel deploy --yes`.

### 3.6 Final smoke against the custom domain

Run the Playwright golden path:

```bash
BASE_URL=https://<custom-domain> bunx playwright test tests/golden-path.spec.ts
```

Then walk the user through a real end-to-end:
1. Sign up with their personal email
2. Welcome email arrives (from the verified domain, in their inbox not spam)
3. Open chat, type "hello", get streaming response
4. PostHog production project shows the events

### 3.7 Write SHIPPED.md

Create `docs/pirate-launch-kit/SHIPPED.md`:

```markdown
# Shipped — {{PROJECT_NAME}}

**Live URL:** https://{{CUSTOM_DOMAIN}}
**Shipped on:** {{DATE}}

## Service dashboards

- Vercel: https://vercel.com/<scope>/{{PROJECT_NAME}}
- GitHub: https://github.com/<user>/{{PROJECT_NAME}}
- Clerk (production): https://dashboard.clerk.com
- Neon: https://console.neon.tech
- Stripe (live mode): https://dashboard.stripe.com
- Resend: https://resend.com/domains
- PostHog (prod project): https://<region>.posthog.com

## What works on day 1

- Landing page renders at the custom domain
- Real users sign up via Clerk; rows land in Neon
- Real Stripe charges work in live mode
- Welcome emails send from the verified domain
- Chat streams real responses from Claude via Vercel AI Gateway
- PostHog records every event on a production project

## What's deliberately not here yet

- Anything that's product, not runway. The agent's character. The specific tools you want it to have. The visual language. The magic interaction only your product needs. None of that is the kit's job.

## You shipped

The runway is built. The clock you've been running against hasn't even started. From here, every minute goes into the part only you can build.

Build. Grow. Repeat.

— The Pirate Launch Kit
```

### 3.8 Final commit

```bash
git add -A
git commit -m "feat: production cutover — custom domain, Clerk prod, Stripe live, Resend verified"
git push origin main
```

### 3.9 Tick everything in CHECKLIST.md and announce

Tick:
- `[x] Custom domain live: <domain>`
- `[x] Clerk on production instance`
- `[x] Stripe in live mode (KYC complete; real test charge + refund verified)`
- `[x] Resend sending from verified domain`
- `[x] PostHog on production project`
- `[x] Final smoke passed on custom domain`
- `[x] SHIPPED.md written`
- `[x] Pushed to GitHub main`

Then say:

*"You shipped. https://<custom-domain> is live. Real users can sign up, real charges go through, real emails send from your domain. From here, every minute goes into the part only you can build. Want me to help draft the first three product features now, or are you ready to take it from here?"*

## Common pitfalls

| Issue | Fix |
|-------|-----|
| User doesn't have Stripe KYC done | Pause Phase 3. Save state in CHECKLIST. Tell them: "Complete KYC in your Stripe dashboard, then come back and tell me 'continue'." |
| DNS hasn't propagated after 30 min | Run `dig <domain>` to verify. If still pointing wrong, the user's registrar might have low-TTL caching. Wait another 30 min before escalating. |
| Resend verification stuck "pending" | Common cause: TTL on TXT records. Tell user to set TTL to 300s if registrar allows. Otherwise wait. |
| Clerk production users colliding with dev users | They don't — they're literally different databases. Reassure user. |
| User wants to skip custom domain and ship on .vercel.app | Fine. Note in CHECKLIST → Divergences. Skip 5.1 and 5.4, the .vercel.app subdomain is fine for soft launches. |
| User's first real Stripe charge fails (3DS challenge) | Normal in EU/UK. Tell them to complete the 3DS prompt. If using `4242 4242 4242 4242` they're still in test mode — they must use a real card for live mode verification. |
| Resend emails landing in spam even on verified domain | Add DMARC record (`_dmarc.<domain>` TXT `v=DMARC1; p=none;`). Run again after 1 hour. |
