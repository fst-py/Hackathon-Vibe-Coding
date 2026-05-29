# Phase 2 — Backend Layer

**Goal:** Wire Clerk (auth), Neon + Drizzle (database), Resend (email), Stripe (test mode), and PostHog (analytics) — all using **dev/test keys**, all installed via Vercel Marketplace where possible. **Each service ends with a redeploy and a smoke against the live URL** so the user watches their app come alive on the internet, one piece at a time.

**You enter this phase with:** chat already works on the live URL behind a password (from Phase 1). The first thing Phase 2 does is **replace that password with real Clerk auth** — the password disappears, anyone can sign up, the chat is now properly gated.

**You finish this phase when:** the user signs up on the live URL (no password), gets a welcome email, sees their row in Neon, completes a Stripe test checkout, and sees the events land in PostHog.

## The three patterns this phase uses

### A. Marketplace-or-CLI install pattern

For each service:

1. **Ask:** *"Do you already have a [Service] account, or should I set one up for you?"*
2. **New account path:** open the Vercel Marketplace integration URL. The integration injects env vars into all Vercel environments automatically.
3. **Existing account path:** authenticate the local CLI, then push the resulting key into the Vercel vault yourself via the stdin pattern below.
4. **After install:** `vercel env pull .env.local` to sync to local.

### B. Lazy-sync user creation (no dev-time webhook tunnels)

When an authenticated request hits the server, a `getOrCreateUser()` helper checks Neon and creates the row if missing. Welcome email fires from the same helper. The Clerk webhook gets wired in 2.1 too — pointing at the live URL from day one — as redundancy on top of lazy sync. No ngrok, no tunnel.

### C. Deploy after each service, smoke against the live URL

After every service wires:

```bash
git add -A
git commit -m "feat(<service>): wire <service>"
vercel deploy --yes   # ~30s; the live URL stays the same: https://<project>.vercel.app
```

Then run the service's smoke test **against the live URL** (not localhost). The user watches the live URL come alive in their browser. This is the dopamine loop.

## `vercel env add` (non-interactive)

The CLI is interactive by default. To add a secret from inside the skill, pipe via stdin:

```bash
printf "%s" "VALUE_HERE" | vercel env add KEY_NAME development --force
printf "%s" "VALUE_HERE" | vercel env add KEY_NAME preview     --force
printf "%s" "VALUE_HERE" | vercel env add KEY_NAME production  --force
```

`--force` overwrites if the key already exists. Run all three environments — Vercel doesn't sync between them.

## Step-by-step (in this exact order)

### 2.1 Clerk (auth) — replaces the password gate

**Load Clerk patterns now (point-of-use, not earlier):** Clerk doesn't publish an official skill on skills.sh as of this writing. `WebFetch` the live Next.js quickstart so you're working from current syntax instead of training data:

```
WebFetch: https://clerk.com/docs/nextjs/getting-started/quickstart
```

**Marketplace URL:** `https://vercel.com/marketplace/clerk`

After install, `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` and `CLERK_SECRET_KEY` are auto-injected.

```bash
bun add @clerk/nextjs svix
vercel env pull .env.local
```

Then (following the `clerk-setup` skill):
- Wrap `src/app/layout.tsx` with `<ClerkProvider>`
- Create `src/middleware.ts` with `clerkMiddleware()`
- Add `/sign-in/[[...sign-in]]/page.tsx` and `/sign-up/[[...sign-up]]/page.tsx`
- Replace the placeholder Sign in button on `page.tsx` with `<SignInButton mode="modal"><Button>Sign in</Button></SignInButton>`, and add `<UserButton />` for signed-in users

**Rip out the password gate.** It served its purpose for Phase 1; Clerk replaces it.

1. **Update `src/app/api/chat/route.ts`** — swap the cookie check for Clerk:
   ```typescript
   import { streamText } from "ai";
   import { auth } from "@clerk/nextjs/server";

   export async function POST(req: Request) {
     const { userId } = await auth();
     if (!userId) return new Response("Unauthorized", { status: 401 });

     const { messages } = await req.json();
     const result = streamText({
       model: "anthropic/claude-sonnet-4-6",
       messages,
     });
     return result.toUIMessageStreamResponse();
   }
   ```
2. **Delete `src/app/api/chat-auth/route.ts`** — no more password endpoint.
3. **Strip the password modal from `page.tsx`** — replace the `unlocked` state and `<Dialog>` with a Clerk-aware check: render the chat input enabled if `useUser().isSignedIn`, else show a "Sign in to chat" placeholder that opens the Clerk modal.
4. **Remove the env var:**
   ```bash
   for env in development preview production; do vercel env rm CHAT_ACCESS_PASSWORD $env --yes; done
   vercel env pull .env.local
   ```
5. **Update CHECKLIST.md** — strike through the Phase 1 password line and add a note: *"Replaced by Clerk auth on YYYY-MM-DD."*

**Wire the Clerk webhook handler** at `src/app/api/webhooks/clerk/route.ts` — it verifies `svix-*` headers and (after 2.2 lands) calls `getOrCreateUser()`. For now leave a TODO for the call site. Webhook signature verification is mandatory from the start (per Security defaults).

**Register the webhook in the Clerk Dashboard** → Webhooks → Add endpoint → `https://<project>.vercel.app/api/webhooks/clerk` → subscribe to `user.created` → copy signing secret → push to Vercel:

```bash
for env in development preview production; do
  printf "%s" "$CLERK_WEBHOOK_SECRET" | vercel env add CLERK_WEBHOOK_SECRET $env --force
done
vercel env pull .env.local
```

**Deploy and smoke:**

```bash
git add -A && git commit -m "feat(clerk): swap password gate for real authentication"
vercel deploy --yes
```

Then on the live URL: confirm the password modal is gone → click Sign in → create a test account → chat works without any password → confirm the user appears in Clerk Dashboard. Tick. *"Password gate retired. Anyone can now sign up at [URL] and use the chat. The friends you shared the password with last phase will need to sign up properly now — message them."*

### 2.2 Neon + Drizzle (database)

**Marketplace URL:** `https://vercel.com/marketplace/neon`

After install, `DATABASE_URL` is auto-injected.

```bash
bun add drizzle-orm pg
bun add -d drizzle-kit @types/pg
vercel env pull .env.local
```

Create `src/db/schema.ts`:

```typescript
import { pgTable, serial, text, timestamp } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: serial("id").primaryKey(),
  clerkId: text("clerk_id").notNull().unique(),
  email: text("email").notNull(),
  stripeCustomerId: text("stripe_customer_id"),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});
```

Create `drizzle.config.ts` and `src/db/index.ts` exporting a `db` client. Run the first migration:

```bash
bunx drizzle-kit generate
bunx drizzle-kit push --force
```

Wire **lazy sync** at `src/lib/users.ts`:

```typescript
import { auth, currentUser } from "@clerk/nextjs/server";
import { db } from "@/db";
import { users } from "@/db/schema";
import { eq } from "drizzle-orm";
import { cache } from "react";

export const getOrCreateUser = cache(async () => {
  const { userId } = await auth();
  if (!userId) return null;

  const existing = await db.select().from(users).where(eq(users.clerkId, userId)).limit(1);
  if (existing[0]) return existing[0];

  const clerkUser = await currentUser();
  const email = clerkUser?.emailAddresses[0]?.emailAddress ?? "";

  const [created] = await db.insert(users)
    .values({ clerkId: userId, email })
    .onConflictDoNothing({ target: users.clerkId })
    .returning();

  // Resend hook lands in 2.3
  return created;
});
```

The `cache()` wrapper dedupes within a single request. `onConflictDoNothing` makes the function idempotent — safe whether triggered by lazy sync or by the Clerk webhook.

Wire `getOrCreateUser()` into the Clerk webhook handler from 2.1 (replace the TODO). Also call it from any signed-in Server Component.

**Deploy and smoke:**

```bash
git add -A && git commit -m "feat(neon): wire database with lazy-sync user creation"
vercel deploy --yes
```

Then on the live URL: sign up a fresh user → load the post-signin page → row appears in Neon. Verify via `bunx drizzle-kit studio` (opens at `https://local.drizzle.studio` pointing at the same Neon instance). Tick. *"Live URL now has a real database under it."*

### 2.3 Resend (transactional email)

**Marketplace URL:** `https://vercel.com/marketplace/resend`

After install, `RESEND_API_KEY` is auto-injected.

```bash
bun add resend react-email @react-email/components
vercel env pull .env.local
```

Create `src/emails/welcome.tsx`:

```tsx
import { Html, Heading, Text } from "@react-email/components";

export default function Welcome({ projectName }: { projectName: string }) {
  return (
    <Html>
      <Heading>Welcome to {projectName}.</Heading>
      <Text>The runway is built. The clock starts now.</Text>
    </Html>
  );
}
```

Create `src/lib/email.ts`:

```typescript
import { Resend } from "resend";
import Welcome from "@/emails/welcome";

const resend = new Resend(process.env.RESEND_API_KEY);

export async function sendWelcomeEmail(to: string, projectName: string) {
  await resend.emails.send({
    from: "onboarding@resend.dev",  // Phase 3 swaps for verified domain
    to,
    subject: `Welcome to ${projectName}`,
    react: Welcome({ projectName }),
  });
}
```

Wire it into `getOrCreateUser()` from 2.2 — after the `db.insert`, call `await sendWelcomeEmail(email, "{{PROJECT_NAME}}")`. Wrap in try/catch so a Resend outage doesn't break signup.

**Phase 2 limitation:** Resend's sandbox sender (`onboarding@resend.dev`) only delivers to the email address the Resend account was created with. Expected — Phase 3 wires a verified domain.

**Deploy and smoke:**

```bash
git add -A && git commit -m "feat(resend): wire welcome email"
vercel deploy --yes
```

Then on the live URL: sign up with the Resend account email → welcome email lands in inbox. Tick.

### 2.4 Stripe (test mode)

**Load Stripe patterns now (point-of-use, not earlier):** Stripe doesn't publish an official skill on skills.sh as of this writing. `WebFetch` the live Checkout quickstart so you're working from current API shapes:

```
WebFetch: https://docs.stripe.com/checkout/quickstart
```

**No Marketplace integration for Stripe.** Stripe CLI is already installed from Phase 0.4.

```bash
stripe login   # browser flow, one-time
```

Get the test keys non-interactively and push to Vercel:

```bash
STRIPE_SK=$(stripe config --list | awk -F"'" '/test_mode_api_key/ {print $2}')
STRIPE_PK=$(stripe config --list | awk -F"'" '/test_mode_publishable_key/ {print $2}')

for env in development preview production; do
  printf "%s" "$STRIPE_SK" | vercel env add STRIPE_SECRET_KEY $env --force
  printf "%s" "$STRIPE_PK" | vercel env add NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY $env --force
done
vercel env pull .env.local
```

**Register the Stripe Sandbox MCP now** (with the real `$STRIPE_SK` from above — registering earlier with a placeholder fails). Restart Claude Code after this so the MCP loads:

```bash
claude mcp add stripe-sandbox npx @stripe/mcp -- \
  --tools=products.create,prices.create,products.list \
  --api-key="$STRIPE_SK"
```

Create one test product **via Stripe Sandbox MCP** (where MCP genuinely outclasses CLI — structured input, idempotent):

```
mcp__Stripe_Sandbox call: products.create
  name: "Starter"
  default_price_data: { currency: "usd", unit_amount: 1000 }
```

Wire `src/app/api/checkout/route.ts` (a POST creating a Checkout Session) and a "Buy" button visible only to signed-in users.

Wire `src/app/api/webhooks/stripe/route.ts` that verifies signatures with `stripe.webhooks.constructEvent` and updates the user's `stripe_customer_id` on `checkout.session.completed`.

**Register the live webhook in Stripe Dashboard** (test mode) → Developers → Webhooks → Add endpoint → `https://<project>.vercel.app/api/webhooks/stripe` → subscribe to `checkout.session.completed` → copy signing secret → push to Vercel:

```bash
for env in development preview production; do
  printf "%s" "$STRIPE_WEBHOOK_SECRET" | vercel env add STRIPE_WEBHOOK_SECRET $env --force
done
vercel env pull .env.local
```

**Deploy and smoke:**

```bash
git add -A && git commit -m "feat(stripe): wire test-mode checkout"
vercel deploy --yes
```

Then on the live URL: signed-in user clicks Buy → completes checkout with `4242 4242 4242 4242` → webhook fires → user's `stripe_customer_id` populated in Neon. Tick.

(Reference the loaded `stripe-best-practices` skill for detailed integration patterns. If the user wants faster iteration during dev, `stripe listen --forward-to localhost:3000/api/webhooks/stripe` works against localhost — but the live URL is the source of truth.)

### 2.5 PostHog (analytics)

**Marketplace URL:** `https://vercel.com/marketplace/posthog` *(if available — fall back to dashboard signup at `https://posthog.com/signup`, create a project, push the project key to Vercel via the stdin pattern above)*

After install, `NEXT_PUBLIC_POSTHOG_KEY` and `NEXT_PUBLIC_POSTHOG_HOST` are injected.

```bash
bun add posthog-js posthog-node
vercel env pull .env.local
```

Wire `src/app/providers.tsx` with PostHog client init using `person_profiles: "identified_only"` (security default — see SKILL.md). Disable session-recording autocapture on form fields.

Capture events from the signup chain:
- Landing: `posthog.capture("landing_viewed")`
- After `getOrCreateUser` creates a row: `posthog.capture("signed_up")`
- After Resend sends: `posthog.capture("welcome_email_sent")`
- In Stripe webhook handler: `posthog.capture("checkout_completed")`

For server-side telemetry, create `src/lib/posthog-server.ts` exporting a singleton.

**Deploy and smoke:**

```bash
git add -A && git commit -m "feat(posthog): wire analytics"
vercel deploy --yes
```

Then on the live URL: load landing → `landing_viewed` appears in PostHog Live Events within 30s. Sign up → `signed_up`. Buy → `checkout_completed`. Tick.

## Wrapping up Phase 2

After all five services are wired, run the integration smoke from a fresh user against the **live URL**:

1. Sign up at `https://<project>.vercel.app`
2. Land on signed-in page → row in Neon (lazy sync) + Clerk webhook also fires (idempotent — `onConflictDoNothing`) + welcome email arrives
3. Click Buy → complete test checkout → `stripe_customer_id` populated
4. PostHog Live Events shows the four events in order

Tick the Phase 2 boxes in CHECKLIST.md and say:

*"Phase 2 done. **Your live URL is now a real SaaS.** Anyone can sign up (no password), they hit a real database, get a welcome email, can pay in test mode, and every action is measured. The chat that's been working since Phase 1 is now properly gated by Clerk instead of a shared password. Last phase: production cutover — custom domain, real Clerk prod keys, Stripe live mode, Resend on a verified sending domain, real PostHog project. This is where decisions become harder to undo, so I'll pause and confirm before each irreversible step. Ready to start, or want to take a break here first?"*

On confirm → load `references/3-production-cutover.md` and enter Phase 3.

## Common pitfalls

| Issue | Fix |
|-------|-----|
| Marketplace install opens but env vars don't appear in `vercel env pull` | Wait 10–20 seconds — propagation isn't instant. Retry the pull. |
| `vercel env add` hangs waiting for input | You forgot to pipe via `printf "%s" "VALUE" \|`. Always use stdin pattern. |
| `vercel deploy --yes` redeploys old code | Make sure you committed first. Vercel deploys what's in the committed git tree by default. |
| Bun fails to install `@clerk/nextjs` or `posthog-node` | Some packages still ship CJS-only. Fall back to `npm i <pkg>`. |
| Drizzle push asks "are you sure?" interactively | `bunx drizzle-kit push --force` skips confirmation. |
| User wants Supabase instead of Neon | Warn: kit's playbook diverges. Note in CHECKLIST → Divergences. |
| Clerk webhook 401s on the live URL | `CLERK_WEBHOOK_SECRET` must be set on the **production** Vercel env (not just preview). Re-run the `vercel env add ... production` line and redeploy. |
| Resend rejects send to a non-account-owner address | Sandbox sender only delivers to the Resend account email. Normal until Phase 3 verifies a domain. |
| PostHog events not appearing | Check ad blocker isn't running. Confirm `NEXT_PUBLIC_POSTHOG_HOST` matches your region (`us.i.posthog.com` vs `eu.i.posthog.com`). |
| User worries about deploying after every service ("isn't that a lot?") | Reassure: ~30s each, 5 deploys, and each one shows them their app getting more capable. This is the dopamine loop. |
