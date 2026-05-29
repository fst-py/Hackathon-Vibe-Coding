# Phase 1 — Frontend + Agent + First Deploy

**Goal:** A live `*.vercel.app` URL serves the agent shell with **streaming chat working from minute one** — gated behind a single password the kit generates and writes into CHECKLIST. Same shell renders locally too.

**You finish this phase when:** the user opens the live URL on their phone, enters the password, types "hello," and watches Claude stream a response back. **This is the magic moment of the whole kit.** Set the room for it.

**The shape of this phase:**
1. Scaffold Next.js + Tailwind + shadcn + AI Elements (the bones)
2. Wire Vercel AI Gateway + AI SDK + Workflow + Chat SDK (the agent layer)
3. Add a hardcoded-password gate over the chat endpoint
4. Wire the chat UI with a password modal
5. Deploy + smoke against the live URL

**Why password-gated and not open:** an open chat endpoint on a public Vercel URL gets indexed, scraped, and drained. A non-technical founder is one accidental share away from a real bill. Phase 2.1 swaps the password for real Clerk auth — until then, the password is the safety wire.

## Step-by-step

### 1.1 Scaffold Next.js 16

```bash
bunx create-next-app@latest . \
  --typescript \
  --tailwind \
  --app \
  --src-dir \
  --import-alias "@/*" \
  --turbo \
  --no-eslint
```

If `create-next-app` complains the folder isn't empty (the `docs/` from Phase 0), pass `--use-bun` and accept the overwrite — it won't touch `docs/`.

### 1.2 Add shadcn/ui

```bash
bunx shadcn@latest init -d
bunx shadcn@latest add input card avatar dialog
```

(`button` is added by `init -d` already; `dialog` is for the password modal in 1.6.)

### 1.3 Add AI Elements + TooltipProvider

```bash
bunx ai-elements@latest
```

This installs the **entire** AI Elements catalog into `src/components/ai-elements/` (~50 files). There's no separate `add` step — that's a no-op now.

**Trim to what the kit uses** to keep the dep tree tight and avoid type-error noise from unused components. AI Elements ships **one file per primitive group** (each file exports many named components), so the keep list is short:

```bash
# conversation.tsx exports Conversation, ConversationContent, ConversationEmptyState, ConversationScrollButton, ConversationDownload
# message.tsx exports Message, MessageContent, MessageResponse, MessageActions, ...
# prompt-input.tsx exports PromptInput, PromptInputBody, PromptInputFooter, PromptInputTextarea, PromptInputSubmit, PromptInputTools, ...
KEEP="conversation message prompt-input"
cd src/components/ai-elements
for f in *.tsx; do
  base="${f%.tsx}"
  case " $KEEP " in *" $base "*) ;; *) rm "$f";; esac
done
cd -
```

(There is no separate `response.tsx` or `loader.tsx` anymore — `MessageResponse` lives in `message.tsx` and replaces what used to be `<Response>`.)

**Wrap the layout with `<TooltipProvider>`** — AI Elements components need this and the install message says so but it's easy to miss. In `src/app/layout.tsx`, wrap the body content:

```tsx
import { TooltipProvider } from "@/components/ui/tooltip";
// ...
<body><TooltipProvider>{children}</TooltipProvider></body>
```

(`bunx shadcn@latest add tooltip` if it isn't there.)

### 1.4 Link to Vercel (AI Gateway is automatic)

The user needs a Vercel account first. If new to Vercel, open `https://vercel.com/signup` and let them sign up with GitHub (one click). Then:

```bash
vercel login                     # browser flow
vercel link --yes                # creates Vercel project; --yes skips the interactive scope/name picker so it works in non-interactive shells
vercel env pull .env.local       # syncs env vars including the OIDC token
```

**No marketplace install for the AI Gateway.** Linked Vercel projects automatically get a `VERCEL_OIDC_TOKEN` in `.env.local`, and the AI SDK uses it transparently to route through the Gateway. Provider strings like `"anthropic/claude-sonnet-4-6"` Just Work — no API key, no marketplace step.

### 1.5 Install AI SDK v6 + Workflow SDK

```bash
bun add ai @ai-sdk/react workflow next-themes
```

⚠ **AI SDK is v6** (as of writing). The patterns below are v6-specific. Verify with `cat node_modules/ai/package.json | grep '"version"'` — if you see `^4` or `^5`, either upgrade (`bun add ai@latest @ai-sdk/react@latest`) or reach for v4/v5 patterns instead. **Don't mix the two.**

What changed from v4 → v6:
- `useChat()` returns `{ messages, sendMessage, status, stop, regenerate }` — **no** `input`, `handleInputChange`, or `handleSubmit`.
- The `api` option is gone from the top level — passing it as `useChat({ api: "..." })` won't compile. Default endpoint is `/api/chat` already; for a custom URL use a transport.
- Messages have `m.parts: UIMessagePart[]` instead of `m.content`. Text lives in `parts.filter(p => p.type === "text")`.
- `convertToModelMessages` is **async** — `await convertToModelMessages(messages)` in the route.

(Section 1.7 has the full working v6 templates.)

**Replace `next.config.ts` with the kit's canonical version** — three things baked in: workflow extensions, the Base UI / AI Elements upstream type-error workaround, and a turbopack root pin to silence the workspace warning.

```typescript
// next.config.ts
import type { NextConfig } from "next";

const config: NextConfig = {
  // Workflow SDK ships compiled .js / .jsx route files — without these
  // extensions every workflow endpoint 404s.
  pageExtensions: ["js", "jsx", "ts", "tsx", "md", "mdx"],

  // AI Elements' prompt-input.tsx ships internal TS errors against the
  // current Base UI release (lines around BaseUIEvent / openDelay /
  // closeDelay). Build-time fail until upstream patches. Drop this when
  // ai-elements ships a fix.
  typescript: { ignoreBuildErrors: true },

  // Pin Turbopack to this folder. Without this, Next detects ~/bun.lock
  // as a higher workspace root and warns on every build.
  turbopack: { root: __dirname },
};

export default config;
```

The `workflow` skill (loaded by the Vercel plugin) has the full gotcha catalog if you need it.

Create `src/workflows/example.ts`:

```typescript
"use workflow";

export async function exampleWorkflow({ message }: { message: string }) {
  await logStep({ message });
  return { ok: true };
}

async function logStep({ message }: { message: string }) {
  "use step";
  console.log("Workflow step ran with:", message);
}
```

**`@vercel/chat-sdk` is NOT a real npm package.** Vercel's "Chat SDK" is a [template repo](https://chat-sdk.dev), not something you install. The kit doesn't wire it in Phase 1 — when the user wants persistent threads in Phase 2.2 (after Neon exists), we either copy the template patterns or scaffold from `git clone`. For now: nothing to install.

### 1.5b Wire fonts + theming (the difference between "shipped" and "self-made-looking")

**Fix Geist font rendering.** The current `bunx shadcn@latest init -d` already writes an `@theme inline` block in `src/app/globals.css`, but the `--font-sans` line self-references (`--font-sans: var(--font-sans)`) — broken. **Read the file, then patch the one line:**

```bash
# Read first to confirm the broken line is there
grep -n "font-sans" src/app/globals.css
```

Edit the line in `@theme inline` from:

```css
  --font-sans: var(--font-sans);
```

to:

```css
  --font-sans: var(--font-geist-sans);
```

Don't add a new `@theme inline` block — patching the existing line is the whole fix. (Without it, the page renders in Tailwind's default `system-ui` and looks generic.)

**Wire dark mode + system preference** with `next-themes` (already installed in 1.5):

```tsx
// src/app/layout.tsx — wrap children with both providers
import { ThemeProvider } from "next-themes";
import { TooltipProvider } from "@/components/ui/tooltip";

<body>
  <ThemeProvider attribute="class" defaultTheme="system" enableSystem disableTransitionOnChange>
    <TooltipProvider>{children}</TooltipProvider>
  </ThemeProvider>
</body>
```

Add a `ModeToggle` button to the header (we use it in the page template in 1.7). Create `src/components/mode-toggle.tsx` from shadcn's recipe:

```tsx
"use client";
import { Moon, Sun } from "lucide-react";
import { useTheme } from "next-themes";
import { Button } from "@/components/ui/button";

export function ModeToggle() {
  const { theme, setTheme } = useTheme();
  return (
    <Button variant="ghost" size="icon" onClick={() => setTheme(theme === "dark" ? "light" : "dark")}>
      <Sun className="h-4 w-4 rotate-0 scale-100 dark:-rotate-90 dark:scale-0 transition-all" />
      <Moon className="absolute h-4 w-4 rotate-90 scale-0 dark:rotate-0 dark:scale-100 transition-all" />
      <span className="sr-only">Toggle theme</span>
    </Button>
  );
}
```

(`bunx shadcn@latest init -d` already writes `@custom-variant dark (&:is(.dark *));` in `globals.css` — that's the equivalent and it works. Don't add `@variant dark (.dark &);` on top of it.)

### 1.6 Generate the password and wire the gate

Generate a friendly random password:

```bash
PASSWORD=$(openssl rand -base64 12 | tr -d '/+=' | head -c 12)
echo "Generated password: $PASSWORD"
```

Push it to Vercel as `CHAT_ACCESS_PASSWORD` (using the stdin pattern — see SKILL.md security defaults):

```bash
# 2>/dev/null suppresses Vercel CLI's noisy "Run one of the commands in next[]"
# hint that fires on success and looks like an error.
for env in development preview production; do
  printf "%s" "$PASSWORD" | vercel env add CHAT_ACCESS_PASSWORD $env --force 2>/dev/null
done
vercel env pull .env.local
```

**Write the password into CHECKLIST.md** under the new "Phase 1 Access Password" section (template below). Tell the user explicitly: *"This is the password that lets you and anyone you share it with use the chat. It's only here as a safety wire — Phase 2 swaps it for real sign-in. Until then, share `[live URL] + [password]` with anyone you want to let in."*

Create the gate route at `src/app/api/chat-auth/route.ts`:

```typescript
import { cookies } from "next/headers";

export async function POST(req: Request) {
  const { password } = await req.json();
  if (password !== process.env.CHAT_ACCESS_PASSWORD) {
    return new Response("Wrong password", { status: 401 });
  }
  (await cookies()).set("chat-access", password, {
    httpOnly: true,
    sameSite: "lax",
    secure: process.env.NODE_ENV === "production",  // localhost is http, not https
    maxAge: 60 * 60 * 24 * 30,
  });
  return new Response("ok");
}
```

Create the streaming endpoint at `src/app/api/chat/route.ts` (AI SDK v6):

```typescript
import { streamText, convertToModelMessages, type UIMessage } from "ai";
import { cookies } from "next/headers";

export async function POST(req: Request) {
  const access = (await cookies()).get("chat-access")?.value;
  if (access !== process.env.CHAT_ACCESS_PASSWORD) {
    return new Response("Locked", { status: 401 });
  }

  const { messages }: { messages: UIMessage[] } = await req.json();
  const result = streamText({
    model: "anthropic/claude-sonnet-4-6",      // routed via AI Gateway (auto-OIDC)
    messages: await convertToModelMessages(messages),  // v6: this is async
  });
  return result.toUIMessageStreamResponse();
}
```

### 1.7 Build the polished agent shell with password modal

Replace `src/app/page.tsx` — this is the AI SDK **v6** version with the full AI Elements polish (empty state, response markdown rendering, footer with submit + tools, disabled Sign in with tooltip, theme toggle).

⚠ **Substitute `{{PROJECT_NAME}}` before writing the file.** The template below contains the literal token `{{PROJECT_NAME}}` in the `<h1>`. Replace it with the project name from CHECKLIST.md (or the folder name if the user didn't pick a different one). This is the same value you wrote in Phase 0.7. **Don't ship the file with the token still in it** — the user will see `{{PROJECT_NAME}}` in their browser header.

```tsx
"use client";

import { useState, useEffect } from "react";
import { useChat } from "@ai-sdk/react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Dialog, DialogContent, DialogTitle } from "@/components/ui/dialog";
import { Tooltip, TooltipContent, TooltipTrigger } from "@/components/ui/tooltip";
import { ModeToggle } from "@/components/mode-toggle";
import {
  Conversation,
  ConversationContent,
  ConversationEmptyState,
} from "@/components/ai-elements/conversation";
import { Message, MessageContent, MessageResponse } from "@/components/ai-elements/message";
import {
  PromptInput,
  PromptInputBody,
  PromptInputFooter,
  PromptInputTextarea,
  PromptInputSubmit,
  PromptInputTools,
} from "@/components/ai-elements/prompt-input";

export default function Home() {
  const [unlocked, setUnlocked] = useState<boolean | null>(null);
  const [password, setPassword] = useState("");
  const { messages, sendMessage, status } = useChat();

  // Probe the chat endpoint to detect whether the access cookie is set.
  useEffect(() => {
    fetch("/api/chat", { method: "POST", body: JSON.stringify({ messages: [] }) })
      .then((r) => setUnlocked(r.status !== 401));
  }, []);

  async function unlock(e: React.FormEvent) {
    e.preventDefault();
    const r = await fetch("/api/chat-auth", {
      method: "POST",
      body: JSON.stringify({ password }),
    });
    if (r.ok) setUnlocked(true);
    else alert("Wrong password");
  }

  return (
    <main className="min-h-screen flex flex-col font-sans">
      <header className="flex items-center justify-between px-6 py-4 border-b">
        <h1 className="text-xl font-semibold">{{PROJECT_NAME}}</h1>
        <div className="flex items-center gap-2">
          <ModeToggle />
          <Tooltip>
            <TooltipTrigger render={<Button variant="outline" disabled>Sign in</Button>} />
            <TooltipContent>Sign-in arrives in Phase 2 (Clerk)</TooltipContent>
          </Tooltip>
        </div>
      </header>

      <section className="flex-1 flex flex-col items-center px-6 py-8 max-w-3xl mx-auto w-full">
        <Conversation className="w-full flex-1">
          <ConversationContent>
            {messages.length === 0 ? (
              <ConversationEmptyState
                title="An agent that does the work."
                description="Replace this line with what your agent actually does."
              />
            ) : (
              messages.map((m) => (
                <Message key={m.id} from={m.role}>
                  <MessageContent>
                    {m.parts
                      .filter((p) => p.type === "text")
                      .map((p, i) => (
                        <MessageResponse key={i}>{(p as { text: string }).text}</MessageResponse>
                      ))}
                  </MessageContent>
                </Message>
              ))
            )}
          </ConversationContent>
        </Conversation>

        <PromptInput
          className="w-full mt-4"
          onSubmit={(message) => {
            if (!message.text?.trim() || !unlocked) return;
            sendMessage({ text: message.text });
          }}
        >
          <PromptInputBody>
            <PromptInputTextarea
              placeholder={unlocked ? "Type a message…" : "Locked — enter the access password"}
              disabled={!unlocked}
            />
          </PromptInputBody>
          <PromptInputFooter>
            <PromptInputTools />
            <PromptInputSubmit disabled={!unlocked} status={status} />
          </PromptInputFooter>
        </PromptInput>
      </section>

      <Dialog open={unlocked === false}>
        <DialogContent>
          <DialogTitle>Enter the access password</DialogTitle>
          <form onSubmit={unlock} className="flex flex-col gap-3 mt-4">
            <Input
              type="password"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              placeholder="Password"
              autoFocus
            />
            <Button type="submit">Unlock</Button>
          </form>
        </DialogContent>
      </Dialog>
    </main>
  );
}
```

**Key v6 details to copy precisely (mistaking these for v4 patterns is the #1 footgun):**
- `useChat()` — no `api` option, no `input`, no `handleInputChange`, no `handleSubmit`
- `sendMessage({ text })` instead of submitting a form value
- `m.parts.filter(p => p.type === "text")` instead of `m.content`
- `<MessageResponse>` (not raw `<span>`) renders streaming markdown with a typing cursor — exported from `message.tsx`, replaces what older docs called `<Response>`
- `<ConversationEmptyState>` instead of an empty `<Conversation>` (huge polish win)
- `<PromptInputBody>` + `<PromptInputFooter>` + `<PromptInputTools>` for the chrome that makes it look "shipped"

Phase 2.1 swaps the `<Tooltip><Button disabled>Sign in</Button></Tooltip>` for `<SignInButton mode="modal"><Button>Sign in</Button></SignInButton>` and rips out the password modal entirely.

### 1.8 (Optional) See it locally first

You can skip this and go straight to deploy — the canonical Phase 1 validation runs against the deployed URL, not localhost. But if the user wants to lay eyes on it before pushing:

```bash
bun dev
open http://localhost:3000
```

Enter the password, type "hello," confirm a streaming response. `Ctrl+C` to stop the dev server when done.

### 1.9 Commit + GitHub remote + first deploy

```bash
git add -A
git commit -m "feat: scaffold frontend + agent layer with password-gated chat"
gh repo create --private --source=. --remote=origin --push
vercel deploy --yes
```

**Why no `--prod`:** Claude Code's harness blocks `vercel deploy --prod` even in YOLO mode. We don't need it — a Vercel project linked to the `main` branch (which we just pushed) auto-promotes the deploy to production. `vercel deploy --yes` returns `target=production` and aliases the canonical `<project>.vercel.app` URL. Same outcome, no harness prompt.

### 1.10 Smoke against the live URL — the canonical Phase 1 check

The deployed URL is the source of truth — it tests the real artifact, not a local dev server that may disagree with prod runtime. Run a curl-based SSE smoke:

```bash
URL="https://<project>.vercel.app"
PW="<the password you generated in 1.6>"
JAR=$(mktemp)

# 1. Authenticate (sets the chat-access cookie in the jar)
curl -s -c "$JAR" -X POST "$URL/api/chat-auth" \
  -H content-type:application/json \
  -d "{\"password\":\"$PW\"}" > /dev/null

# 2. Stream a chat response and inspect the first ~400 bytes
curl -sN -b "$JAR" -X POST "$URL/api/chat" \
  -H content-type:application/json \
  -d '{"messages":[{"id":"1","role":"user","parts":[{"type":"text","text":"Say hello in five words."}]}]}' \
  --max-time 30 | head -c 400

rm -f "$JAR"
```

**Pass criterion:** the stream starts with `data: {"type":"start"}` and contains at least one `text-delta` chunk. If yes, the live URL is good — open it in the user's browser, hand them the password, and watch them have the magic moment.

### 1.11 (Optional) Add Playwright for ongoing CI

If the user wants browser tests for future iterations:

```bash
bun add -d @playwright/test
bunx playwright install --with-deps chromium
```

Create `tests/golden-path.spec.ts`:

```typescript
import { test, expect } from "@playwright/test";

const baseUrl = process.env.BASE_URL ?? "http://localhost:3000";

test("landing renders with the three required things", async ({ page }) => {
  await page.goto(baseUrl);
  await expect(page.getByRole("heading", { level: 1 })).toBeVisible();
  await expect(page.getByRole("button", { name: /sign in/i })).toBeVisible();
  await expect(page.getByText(/locked|type a message/i)).toBeVisible();
});
```

Add a `playwright.config.ts` with `webServer: { command: "bun dev", url: "http://localhost:3000", reuseExistingServer: true }`. Run against the live URL with `BASE_URL=https://<project>.vercel.app bunx playwright test`.

### 1.12 Confirm checkpoint

Tick in CHECKLIST.md:
- `[x] Next.js 16 scaffolded with Tailwind + TypeScript + Turbo`
- `[x] shadcn/ui initialized with starter components + dialog`
- `[x] AI Elements installed`
- `[x] Vercel AI Gateway wired (Marketplace install)`
- `[x] AI SDK / Workflow SDK / Chat SDK installed; example workflow stub`
- `[x] Password gate wired at /api/chat`
- `[x] Phase 1 access password generated and saved in CHECKLIST`
- `[x] GitHub remote created and pushed`
- `[x] Vercel project linked`
- `[x] Live URL: https://<project>.vercel.app`
- `[x] Live URL curl SSE smoke passes (start event + text-delta)`
- `[x] Live URL chat works with password (browser smoke)`

Then say:

*"Phase 1 done. **Your agentic SaaS is live.** Open `[URL]` on your phone, enter the password I just wrote into CHECKLIST, and chat with Claude. Share the URL + password with three friends and let them try it — it's the realest version of your idea you've ever had. When you're ready, Phase 2 wires the proper SaaS layer underneath: Clerk replaces the password with real sign-in, Neon stores users, Stripe charges them, Resend emails them, PostHog measures them. Confirm or redirect?"*

On confirm → load `references/2-backend.md` and enter Phase 2.

## Common pitfalls

| Issue | Fix |
|-------|-----|
| `create-next-app` rejects folder name with spaces or capitals | Should have been caught in Phase 0.3. If not: `mkdir <kebab-name> && cd <kebab-name>` and retry. |
| `create-next-app` says "directory not empty" | `--use-bun` and accept the overwrite. `docs/` is preserved. |
| `shadcn init` errors on Tailwind v4 | Tailwind v4 ships in Next.js 16 by default. If it complains, `bunx shadcn@canary init -d`. |
| `bunx shadcn@latest add button` says "skipped" | Expected — `init -d` already added it. Cosmetic warning, ignore. |
| `vercel env add` hangs waiting for input | Use the `printf "%s" "VALUE" \|` stdin pattern from SKILL.md. Don't run it bare. |
| AI Gateway 401 on first chat | Run `vercel env pull .env.local` again — OIDC token may not be down locally yet. Check `cat .env.local \| grep VERCEL_OIDC_TOKEN`. |
| `useChat is not a function` / `handleSubmit is undefined` | You're on AI SDK v6 but copied a v4 pattern. Re-read 1.5 — `useChat` returns `{ messages, sendMessage, status }`, not the v4 shape. |
| `convertToModelMessages is not a function` | Same root cause: v6 made it async. `await convertToModelMessages(messages)`. |
| `bun add @vercel/chat-sdk` fails 404 | The package doesn't exist. Skip it — Chat SDK is a template, not a publishable package. |
| TypeScript errors in unused AI Elements files | The trim step in 1.3 removes them. If you skipped trim, either run it now or set `typescript: { ignoreBuildErrors: true }` in `next.config.ts` as a temporary patch. |
| AI Elements components don't render / Tooltip errors | You forgot to wrap layout with `<TooltipProvider>`. See 1.3. |
| Page renders in wrong font (system-ui instead of Geist) | Tailwind v4 doesn't auto-map `font-sans`. Add the `@theme inline` block from 1.5b to `globals.css`. |
| Dark mode toggle does nothing | Check `<ThemeProvider attribute="class">` is set, and Tailwind has `@variant dark (.dark &);` in `globals.css`. |
| Password modal doesn't close on submit | The `useEffect` polls `/api/chat` to detect unlock. Check the cookie set has `secure: process.env.NODE_ENV === "production"` (not hardcoded `true`). |
| Live URL chat returns "Locked" with cookie set | `secure: true` cookies don't transmit on `http://localhost`. Use the conditional pattern from 1.6. |
| User wants to share without sharing the password | Tell them: Phase 2.1 (Clerk) lands soon and lets anyone sign up. Until then, password is the lock. |
| Workflow routes 404 | `pageExtensions` in `next.config.ts` must include `"js"` and `"jsx"`. |
| Build fails: TypeScript errors | `bunx tsc --noEmit` locally first. Fix until clean before deploying. |
