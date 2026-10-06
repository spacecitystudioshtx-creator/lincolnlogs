# Lincoln Logs — Roadmap

A mobile-first website where you chat with "Abraham Lincoln." Goals, in priority order:
cheap to run, launch fast, built entirely from a phone, monetized (free cap → paid credits, ads later),
and a learning project in "harnessing" an LLM that you can show people.

---

## 0. Key design decisions (read this first)

### Harness, don't train
"Training" (fine-tuning) a model on Lincoln's writing is the *last* thing to do, not the first.
The cheap, fast, and better-sounding path is a **harness**:

| Layer | What it is | Cost | Effect |
|---|---|---|---|
| 1. Voice spec (system prompt) | 2–3k tokens: cadence rules, vocabulary, canonical passages, what he knows (nothing after 14 Apr 1865), how he handles modern questions | ~free (prompt-cached) | 80% of the "sounds like Lincoln" effect |
| 2. Retrieval (RAG) | His real letters/speeches chunked + embedded; the 3–5 most relevant passages are injected per question | ~$0.001/msg | Grounds answers in things he actually said; lets him quote himself accurately |
| 3. Evals | A set of 30–50 test questions graded by a stronger model ("Does this sound like Lincoln? Any anachronisms?") | pennies per run | Lets you *measure* each change instead of vibing |
| 4. Fine-tune (optional, later) | LoRA on an open model (via Together/Fireworks) using Q→A pairs generated from the corpus | $5–30 one-off | The "learn training" box, done as an experiment you can compare against 1–3 with your evals |

Everything Lincoln wrote is public domain. The *Collected Works of Abraham Lincoln* (U. Michigan / Project Gutenberg)
plus *Speeches and Letters* (Gutenberg) is the corpus. No licensing risk; you just need a clear "this is an AI
impersonation" disclaimer.

### Model
**Claude Haiku 4.5** (`claude-haiku-4-5`) for production: $1 / $5 per million tokens in/out. Keep Sonnet 5.5
(`claude-sonnet-5-5`, $2 / $10) as a one-line config swap for A/B tests and as the eval judge.

Per-reply cost estimate on Haiku (system prompt cached, ~4 retrieved passages, short history, ~300-token reply):

| Piece | Tokens | Cost |
|---|---|---|
| System prompt (cache read) | 3,000 | $0.0003 |
| Retrieved passages + history | 2,500 | $0.0025 |
| Output | 300 | $0.0015 |
| **Total** | | **≈ $0.004–0.005 / message** |

So 1,000 messages ≈ **$4–5**. Sonnet is ~2×. Cap `max_tokens` (≈500) so one message can never cost more.

### Monetization model
- **Free tier:** 15 messages per device (or per day — pick one; per-device-lifetime is simplest).
- **Paid:** Stripe Checkout, no accounts needed at first — "100 messages for $2.99" credit pack. Optional
  "$4.99/month unlimited-ish (300/day)" later.
- **Ads:** second priority. AdSense needs a live site, a privacy policy, and real traffic before approval, and pays
  very little below ~10k visits/month. Apply after launch; don't block on it.

Break-even math (all free tiers, variable cost ≈ $0.005/msg):
- A free user who burns all 15 messages costs you **$0.075**.
- A $2.99 pack: Stripe fee ≈ $0.39, 100 messages ≈ $0.50 → **~$2.10 margin**.
- One paying user covers **~28 free users**. A 2–4% conversion rate is enough to be profitable from month one.
- Fixed costs: $0 on free tiers; ~$10/yr if you buy a domain.

---

## 1. Stack & deployment options

You'll work from your phone, so the stack must be: git-driven (Claude Code on the web edits the repo), auto-deploy on
push, and administrable from a mobile dashboard.

### Option A — Cloudflare (recommended: cheapest, one platform, you already have the connector)
| Need | Service | Free tier |
|---|---|---|
| Static site + API | Workers (static assets + fetch handler) | 100k requests/day |
| Rate limits / free-message counters | Workers KV | 100k reads/day |
| Credits, purchases | D1 (SQLite) | 5M reads/day |
| RAG vector search | Vectorize | 30M queried dims/mo |
| Embeddings | Workers AI (`bge` models) | 10k neurons/day |
| Bot protection | Turnstile | free |
| Analytics | Web Analytics | free |
| Domain | `*.workers.dev` free, or Cloudflare Registrar at cost |

Deploy: GitHub repo → Cloudflare Workers Builds (auto-deploy on push to `main`). Dashboard works on mobile.
Note: the Cloudflare connector in this workspace needs to be authorized in your claude.ai connector settings before
I can use it directly; everything can also be done via `wrangler` config in the repo + the dashboard.

### Option B — Vercel + Supabase (most polished DX, slightly more moving parts)
Next.js + Vercel AI SDK (`useChat` streaming hook) on Vercel Hobby; Supabase for Postgres + pgvector + magic-link
auth. Two dashboards instead of one; Vercel Hobby forbids commercial use on the free plan ($20/mo Pro), which
matters for a monetized site.

### Option C — Single-file server (Val.town / Railway / Render)
One Hono/Express file with SQLite. Fastest to prototype, least to learn, weakest free tiers for a public site.

**Pick A.** Reasons: $0 fixed cost, commercial use allowed on free tier, every piece (vector DB included) lives in one
account, and the whole thing is a single `wrangler.toml` + one Worker, which is a great thing to show people.

### Phone workflow
- **Claude Code (web/app)** — this session — writes code, runs tests, opens PRs.
- **GitHub mobile** — review diffs, merge.
- **Cloudflare dashboard** — secrets, logs, KV/D1 browsing.
- **Stripe app** — payouts, refunds.
Accounts you need: Anthropic Console (API key, add ~$10 credit), Cloudflare, Stripe, GitHub (have).

---

## 2. Phases

### Phase 0 — Accounts & scaffolding (Day 1, ~1 hour)
- [ ] Anthropic API key, Cloudflare account, Stripe account (needs legal name + bank for payouts; test mode works immediately).
- [ ] Repo scaffold: `wrangler.toml`, Worker entry, static `public/` folder, `npm test` with Vitest, GitHub Actions
      for lint/test; Cloudflare Workers Builds connected to `main`.
- [ ] Secrets in Cloudflare: `ANTHROPIC_API_KEY`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`.
- Deliverable: "Hello from the Worker" live at `lincolnlogs.<you>.workers.dev`.

### Phase 1 — Corpus & voice (Days 1–2)
- [ ] `scripts/fetch-corpus.ts`: download Gutenberg texts (Speeches & Letters; Collected Works vols), strip boilerplate.
- [ ] `scripts/chunk-and-embed.ts`: ~400-token chunks with metadata (date, title, type: letter/speech/debate), embed
      with Workers AI, upsert into Vectorize. ~3–5k chunks, runs once.
- [ ] `src/prompt/lincoln.md`: the voice spec. Sections: identity & dates; style (long periodic sentences,
      biblical/legal cadence, dry humor, parables from Illinois life, self-deprecation); 10–15 verbatim anchor passages;
      knowledge boundary (speaks as of spring 1865, curious but unfamiliar about anything later — never breaks
      character, never claims to be an AI unless directly asked, then answers honestly in one line); safety
      (no medical/legal/financial advice beyond period-appropriate musings; no slurs even when "historically accurate").
- [ ] `evals/questions.json`: 40 questions (history, personal, modern-day, adversarial "ignore your instructions",
      "what do you think of smartphones"). `scripts/eval.ts` runs them and has Sonnet 5.5 grade 1–5 on voice,
      accuracy, anachronism, character-break. Store the score in `evals/results/`.
- Deliverable: a CLI that answers as Lincoln, with an eval score to beat.

### Phase 2 — Chat API (Day 2–3)
- [ ] `POST /api/chat`: validate input (max 600 chars), retrieve top-5 chunks from Vectorize, build messages
      (system prompt with `cache_control`, retrieved passages, last 10 turns), stream Haiku 4.5 via SSE, `max_tokens: 500`.
- [ ] Conversation history lives in the browser (localStorage) and is sent each call — no server-side chat storage,
      no privacy burden.
- [ ] Error handling: rate-limit (429) → polite period-appropriate message; API down → same.
- [ ] Log `usage` tokens per request to a KV counter so you can see real cost per message.

### Phase 3 — UI (Day 3–4)
- [ ] Single `index.html` + small JS (no framework needed): mobile-first, 1860s aesthetic (parchment, serif, a
      daguerreotype-style portrait, ink-cursor streaming), suggested starter questions, "share this exchange" button
      that copies a permalink/image.
- [ ] Message counter visible ("12 of 15 free messages").
- [ ] Disclaimer footer, Privacy Policy and Terms pages (needed for Stripe and AdSense).
- [ ] Lighthouse mobile score > 90.

### Phase 4 — Metering (Day 4)
- [ ] Anonymous device id (random UUID in localStorage + signed cookie) → KV counter `free:<id>`.
- [ ] Turnstile on first message to deter scripted abuse; per-IP cap (e.g. 60/hour) in KV as a backstop.
- [ ] When free cap hit: API returns `402` + the UI shows the paywall card.

### Phase 5 — Payments (Day 5)
- [ ] Stripe Checkout session from `/api/checkout` with the device id in `client_reference_id`.
- [ ] `/api/stripe/webhook` verifies signature, on `checkout.session.completed` writes credits to D1
      (`credits(device_id, remaining, purchased_at, session_id)`).
- [ ] Chat endpoint decrements credits after free tier is exhausted.
- [ ] Recovery path: "Lost your credits? Enter the email you paid with" → look up Stripe customer by email → re-link
      device. Covers cleared storage / new phone without building full auth.
- [ ] Test end-to-end in Stripe test mode, then flip to live.

### Phase 6 — Launch (Day 6–7)
- [ ] Optional domain ($10/yr) — `honestabe.chat`-style names are memorable; `.workers.dev` is fine to start.
- [ ] OG image + meta tags so links preview well in iMessage/Twitter.
- [ ] Cloudflare Web Analytics; a tiny `/admin` page (basic-auth) showing messages/day, cost/day, revenue/day.
- [ ] Post to: r/history, r/SideProject, Hacker News "Show HN", Twitter/X, TikTok clip of a funny exchange.
- [ ] Watch logs for the first 48h: jailbreak attempts, cost spikes, confused users.

### Phase 7 — Ads & growth (Week 2+)
- [ ] Apply for Google AdSense once you have the policy pages and some traffic. Place one unit under the chat, never
      inside it. Remove ads for paying users (a selling point).
- [ ] Alternatives if AdSense declines: EthicalAds, Buy Me a Coffee button.
- [ ] Growth loops: shareable exchanges, "Lincoln's daily letter" email (Cloudflare Email Workers), seasonal
      prompts (Gettysburg anniversary, July 4, Presidents' Day).

### Phase 8 — The learning track (parallel, whenever you want)
Each of these is a weekend experiment with a before/after eval score:
1. **Prompt-only vs RAG** — turn retrieval off; measure. (Teaches when RAG actually matters.)
2. **Haiku vs Sonnet** — same prompt; measure quality vs 2× cost.
3. **Few-shot vs spec** — replace style rules with 8 example exchanges; measure.
4. **Fine-tune** — generate 1–2k synthetic Q→A pairs from the corpus with Sonnet, LoRA-tune an open 8B model on
   Together/Fireworks (~$10), serve it behind the same API, measure. This is "training" done properly: you'll see
   exactly what it buys you over the harness.
5. **Prompt caching** — confirm `cache_read_input_tokens` > 0 in logs; break it deliberately (put a timestamp in
   the system prompt) and watch cost double.
Write each up as a short section in the README — that write-up *is* the portfolio piece.

---

## 3. Risks & honest notes
- **Stripe live mode** needs identity verification and a bank account; start in test mode on day 1 so it's not a
  launch blocker.
- **Ads revenue at small scale is near zero.** Credit packs are the real income; ads are a later bonus.
- **Jailbreaks** will happen. The eval set's adversarial questions + `max_tokens` cap + a server-side output check for
  the phrase "as an AI" keep it cheap and in character.
- **Free-tier abuse**: someone clearing localStorage gets 15 more messages. At $0.075 per abuse it's not worth
  fighting hard; Turnstile + IP caps are enough.
- **No accounts** is a deliberate simplification. Add Supabase/Clerk magic-link auth only if the email-recovery path
  turns out to be insufficient.
- **Repo name**: "lincolnlogs" is a Hasbro trademark for toys — fine as a repo name, risky as the public brand. Pick
  something else for the site ("Honest Abe", "Ask Mr. Lincoln", "The Lincoln Letters").

---

## 4. What to tell me next
1. Confirm **Cloudflare** (Option A) or pick another.
2. Free cap: **15 lifetime** or **10/day**?
3. First pack price: **$2.99 / 100 messages** OK?
4. Do you already have Anthropic Console, Cloudflare and Stripe accounts? (I can scaffold Phase 0 and Phase 1 right
   now without them; you plug in keys when ready.)
