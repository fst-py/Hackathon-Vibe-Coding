# Phase 0 — Onboarding & Setup

**Goal:** Empty folder → calibrated tone, approved permissions, runtimes + service CLIs installed, **Builder Layer loaded** (Vercel plugin + Clerk skill + Stripe Sandbox MCP), `CHECKLIST.md` on disk, git initialized with secret-scan hook.

**You finish this phase when:** the user can `cat docs/pirate-launch-kit/CHECKLIST.md` and see the full plan ahead of them.

## Step-by-step

### 0.1 Calibrate tone

Ask exactly: *"On a scale of 0 to 10, how deep is your technical knowledge — where 0 is 'I've never opened a terminal before today' and 10 is 'I write code professionally'? I'll calibrate how I talk to you for the rest of this session."*

Save the answer mentally and write it as the first line of CHECKLIST.md ("Technical level: X/10"). Use these brackets:
- **0–3**: Explain every technical term inline. Avoid jargon. Use analogies.
- **4–6**: Use jargon, but explain on first use ("env vars — short for environment variables, basically secret settings the app reads at runtime").
- **7–10**: Terse mode. Skip explanations. Show commands, not explanations of commands.

### 0.2 Recommend permission mode

Say: *"For the smoothest ride, I'd recommend turning on auto-approve mode in Claude Code (press Shift+Tab twice). I'll move much faster and ask you less. I'll always pause and ask before anything that costs money or touches your accounts. Sound good, or do you want me to ask before every action?"*

If they pick auto-approve: *"Great. Let's go."*
If they decline: *"Got it. Heads up: the next ~90 minutes will involve a lot of permission prompts. Tap accept when they pop up. Ready?"*

### 0.3 Verify the working directory is empty AND has a valid npm name

**Two checks here, both critical:**

(a) **Empty?** Run `ls -la`. If anything besides `.git` (and the agent's own files) is present, ask: *"This folder isn't empty — I see [list 3-5 things]. Should I proceed inside it anyway, or would you like to start in a fresh subfolder?"*

(b) **Valid npm package name?** `create-next-app` will reject the folder name in Phase 1.1 if it has spaces, uppercase letters, or other npm-illegal characters. Catch this **now**, not 30 minutes in:

```bash
NAME=$(basename "$PWD")
if ! echo "$NAME" | grep -Eq '^[a-z0-9][a-z0-9._-]*$'; then
  # Folder name is npm-illegal
fi
```

If invalid (spaces, capitals, leading non-alphanum), ask: *"Your folder name `$NAME` has [spaces/capitals/etc.] — npm won't accept it as a project name. Should I (a) operate in a kebab-case subfolder you name now, or (b) stay here and patch package.json after scaffold?"*

Default to (a). If they pick (a): `mkdir <kebab-name> && cd <kebab-name>` and continue from there with absolute paths.

### 0.4 Detect OS and install runtimes + service CLIs

Run `uname -s` to detect:
- `Darwin` → macOS
- `Linux` → Linux
- `MINGW*` / `MSYS*` → Windows (likely Git Bash or WSL)

Check what's missing:

```bash
node --version 2>/dev/null
bun --version 2>/dev/null
git --version 2>/dev/null
gh --version 2>/dev/null
vercel --version 2>/dev/null
stripe --version 2>/dev/null
```

For anything missing, install it **yourself**. Do not tell the user to download.

#### macOS (preferred path)

```bash
# Homebrew first if missing
which brew || /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Runtimes + GitHub CLI
brew install node bun git gh

# Service CLIs we'll need in Phase 2
brew install vercel-cli              # if it fails, use: bun i -g vercel
brew install stripe/stripe-cli/stripe
```

#### Linux

```bash
# Node via NodeSource
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs git gh

# Bun
curl -fsSL https://bun.sh/install | bash

# Vercel + Stripe CLIs
bun i -g vercel
curl -fsSL https://github.com/stripe/stripe-cli/releases/latest/download/stripe-linux-x86_64.tar.gz | tar -xzC /usr/local/bin stripe
```

#### Windows (WSL preferred)

If they're on bare PowerShell, suggest they install WSL first (`wsl --install`) and resume from inside WSL. Inside WSL, follow the Linux path.

For each install, tell the user one sentence: *"Installing Node — this takes about 30 seconds."* Don't paste the install logs unless something fails.

### 0.5 Install the Vercel plugin (the only Phase 0 builder install)

**Install only what Phase 1 actually needs.** Service-specific skills/MCPs (Stripe sandbox, Clerk, etc.) get wired the moment you reach the phase that uses them — not earlier. Pre-loading them is noise that confuses the user and risks failed registrations against placeholder keys.

The one exception is the Vercel plugin: it ships ~25 skills for the entire frontend + agent layer (Next.js, shadcn, AI Elements, AI SDK, AI Gateway, Workflow SDK, Chat SDK, env-var hygiene, deploys), and Phase 1 uses all of it.

```bash
yes | npx plugins add vercel/vercel-plugin
```

The `yes |` pipe handles the plugin's own `Y/n` confirmation (which `npx -y` doesn't suppress because it comes from the plugin tool, not npx itself).

If `npx plugins` isn't recognized, install the Claude Code plugins runtime first: `npm i -g @anthropic-ai/claude-code-plugins` then retry. (Bun is required by the Vercel plugin — already installed in 0.4.)

**Verify the load:** ask Claude Code "do you have the vercel-plugin shadcn skill?" — if yes, you're good. If no, retry the plugin install before proceeding.

**Service tooling — wired later, not now:**

| Tool | Wired in | Why deferred |
|------|----------|--------------|
| Stripe Sandbox MCP | Phase 2.4 (when Stripe enters the picture) | Needs a real `sk_test_…` key; placeholder registrations fail and pollute the MCP list |
| `clerk-setup` skill / WebFetch quickstart | Phase 2.1 | Patterns drift fast — fetch at use-site, not days earlier |
| `stripe-best-practices` patterns / WebFetch quickstart | Phase 2.4 | Same reason |
| Neon / PostHog / Resend MCPs | Never (in this kit) | Their CLIs cover everything we need for setup |

(Clerk and Stripe don't publish official skills to skills.sh as of this writing. When you reach Phase 2, fall back to `WebFetch` against `https://clerk.com/docs/nextjs/getting-started/quickstart` and `https://docs.stripe.com/checkout/quickstart`.)

### 0.6 Initialize git + the secret-leak guardrails

```bash
git init
git branch -M main
```

If they don't have a GitHub account or aren't authenticated:
```bash
gh auth status || gh auth login
```

Walk them through `gh auth login` — they'll authenticate in their browser. This is a "first time only" thing.

**Verify `.gitignore` excludes secrets.** Even though Next.js scaffolds a sensible `.gitignore` in Phase 1, it doesn't exist yet at Phase 0. Write a minimal one now so we never commit a secret by accident:

```
# .gitignore (initial)
.env
.env.local
.env*.local
.vercel
node_modules
.DS_Store

# Skill internals — the kit installs ~5 large markdown files under
# .claude/skills/pirate-launch-kit/. They're scaffolding, not part of
# the user's repo. Keep them local.
.claude/
```

**Install the pre-push secret-scan hook.** This is the safety net if any of the kit's other defenses ever miss something:

```bash
cat > .git/hooks/pre-push <<'EOF'
#!/bin/sh
# Pirate Launch Kit — block pushes containing secret-shaped strings.
PATTERNS='sk_live_|sk_test_|whsec_|rsk_live_|rsk_test_|CLERK_SECRET_KEY=[^[:space:]]'

# On first push, origin/main doesn't exist yet — fall back to scanning
# everything currently tracked. Otherwise the hook silently no-ops and
# the user thinks it's working when it isn't.
if git rev-parse --quiet --verify origin/main >/dev/null 2>&1; then
  DIFF=$(git diff origin/main..HEAD)
else
  DIFF=$(git ls-files | xargs -I {} git show "HEAD:{}" 2>/dev/null)
fi

if printf "%s" "$DIFF" | grep -E "$PATTERNS" >/dev/null; then
  echo "❌ Push blocked: secret-shaped string detected."
  echo "   Move it to Vercel with 'vercel env add' and reference it via process.env.X."
  exit 1
fi
EOF
chmod +x .git/hooks/pre-push
```

Don't create the GitHub remote repo yet — that happens in Phase 1 when we connect to Vercel and do the first deploy.

### 0.7 Write CHECKLIST.md

Create the directory and copy the template:

```bash
mkdir -p docs/pirate-launch-kit
```

Then write `docs/pirate-launch-kit/CHECKLIST.md` from `references/checklist-template.md`. Replace:
- `{{PROJECT_NAME}}` — the folder name, or ask them what they want to call the project (fine to ask here, it's a one-question break)
- `{{TECHNICAL_LEVEL}}` — their calibration answer
- `{{PERMISSION_MODE}}` — `YOLO` or `strict`
- `{{DATE}}` — today's date in YYYY-MM-DD

### 0.8 Confirm checkpoint

Tick these in CHECKLIST.md as they land:
- `[x] Technical level captured: X/10`
- `[x] Permission mode chosen: <YOLO | strict>`
- `[x] Working directory ready: <path>`
- `[x] Runtimes + CLIs installed: Node, Bun, git, gh, vercel, stripe`
- `[x] Vercel plugin loaded (service skills/MCPs wired later, at point-of-use)`
- `[x] Git initialized + .gitignore + pre-push secret-scan hook`
- `[x] CHECKLIST.md written to docs/pirate-launch-kit/`

Then say: *"Phase 0 done. The runway has its first stones. Next I'll scaffold the frontend (Next.js + Tailwind + shadcn + AI Elements) plus the agent layer (Vercel AI Gateway + AI SDK + Workflow + Chat SDK), wire up a streaming chat with Claude behind a one-time password I'll generate for you, and deploy it to Vercel — so within ~30 minutes you'll have a live URL where you can chat with your agent on your phone. Confirm or redirect?"*

On confirm → load `references/1-frontend.md` and enter Phase 1.

## Common pitfalls

| Issue | Fix |
|-------|-----|
| `brew` install hangs at "downloading Xcode CLT" | This is a one-time 5-min install. Tell user to wait. Don't kill it. |
| User has Node 18 from years ago | `brew upgrade node` to get Node 20+. Next.js 16 needs ≥20. |
| User on Apple Silicon with x86 brew | Detect via `uname -m`. If `arm64` but `/usr/local/bin/brew` exists, install ARM brew at `/opt/homebrew/bin/brew`. |
| `gh auth login` browser doesn't open | Fall back to device code flow: `gh auth login --web` then read the code aloud. |
| User has 2FA on GitHub but no recovery | Note in CHECKLIST under Divergences, proceed without GitHub remote for now. We'll wire it in Phase 3. |
