# Master Prompt for Cursor — Exam Practice MVP (Landing → Onboarding → Recommendations → Practice)

You are the lead engineer. Generate a production-ready web app with the following requirements.

## Tech Stack

* **Frontend**: Next.js 14 (App Router), TypeScript, Tailwind CSS.
* **Auth & DB**: Supabase (Postgres + Auth) or Prisma + Postgres (prefer Supabase for speed).
* **Payments**: Stripe (monthly/yearly).
* **Storage**: Supabase Storage (audio, exports).
* **Analytics**: PostHog (stubbed if not configured).
* **Backend API**: Next.js Route Handlers (`/app/api/*`).
* **AI Scoring**: Stub endpoints (will later call FastAPI), return mock data for now.

## High-Level Flow

1. **Landing page** `/` with “Start Practicing” CTA.
2. **Onboarding wizard** `/onboarding`:

   * Step 1: Name (text), Age (number).
   * Step 2: Country of Residence (single-select).
   * Step 3: Destination Countries (**multi-select**; use checkboxes or multi-select control; DO NOT use radio). Include CIS medicine destinations.
   * Step 4: Purpose (**multi-select**): Study, Migrate, Professional Development.
     On submit → create/update `study_profile`, then route to `/recommendations`.
3. **Recommendations** `/recommendations`:

   * Show selected Destinations & Purposes as chips.
   * For each Purpose and Destination, list required/typical tests (cards) and “Start Practice” buttons.
   * Sample tests are viewable by all; full practice is gated by plan.
4. **Practice** `/practice/[examCode]`:

   * Tabs: **Sample Test** (view only), **Full Test** (timed, realistic).
   * When starting Full Test, enforce plan rules.
5. **Dashboard** `/dashboard`:

   * Locked (preview only) for Free plan.
   * Full charts for Premium/Unlimited: Band/Score trend, Skill radar, Error heatmap, Time on Task, Recommendations.
6. **Pricing** `/pricing`:

   * Plans: Free, Premium ($3.99/mo or $39/yr), Unlimited ($9.99/mo or $99/yr).
   * Stripe checkout; update user’s plan on webhook.
7. **Account** `/account`: profile, plan, billing link, data export/delete.

## Plans & Entitlements

* **Guest (no account)**: view Sample Tests only (no submissions).
* **Free (account)**: 1 **full test** per rolling 7 days. No dashboards.
* **Premium**: 1 **full test** per 24 hours. Dashboards enabled.
* **Unlimited**: unlimited full tests. Dashboards enabled.

**Definition**: “Full test” = all sections of an exam (e.g., IELTS 4 sections). Partial attempts don’t count until submitted or 72h passes.

## Data Model (SQL/Prisma shape — adapt to Supabase)

**users** (from Supabase):

* id (uuid, pk), email, created_at

**profiles**

* id (pk, uuid = users.id)
* name (text)
* age (int)
* residence_country (text)
* plan (enum: 'guest' | 'free' | 'premium' | 'unlimited', default 'free')
* current_period_end (timestamptz, null)
* created_at, updated_at

**study_profile**

* id (pk)
* user_id (fk -> users.id)
* destination_countries (text[])  // multi-select
* purpose_multi (text[])           // ['study','migrate','prodev']
* created_at

**exam_items**  // minimal for MVP

* id (pk)
* exam_code (text)  // 'IELTS_A','IELTS_GT','TOEFL','PTE_A','PTE_CORE','DET','SAT','GRE_VERBAL','GMAT_VERBAL'
* skill (text)      // 'reading','listening','writing','speaking'
* item_type (text)  // 'mcq','cloze','task1','task2','cue_card'
* stem_md (text)
* choices (jsonb)   // optional
* answer_key (jsonb)// optional
* cefr_level (text) // 'A2'..'C1'
* metadata (jsonb)
* created_at

**attempts**

* id (pk)
* user_id (fk)
* exam_code (text)
* mode (text)         // 'sample' | 'full'
* is_full_test (bool) // true if full test
* started_at, submitted_at
* raw_response (jsonb)
* score (jsonb)
* feedback (jsonb)
* flags (jsonb)       // anti-cheat (tab_blurs, paste_events, etc.)

**entitlements**

* user_id (fk)
* daily_tests_used (int, default 0)
* weekly_tests_used (int, default 0)
* last_reset_daily (timestamptz, null)
* last_reset_weekly (timestamptz, null)

**selector_rules**  // cache basic rules for recommendations

* id (pk)
* purpose (text)
* country (text)
* accepted_exams (jsonb)
* min_scores (jsonb)
* caveats (text)
* last_checked (date)

**recommendations_cache**

* user_id (fk)
* inputs_hash (text)
* payload (jsonb)
* created_at

## Seed Data

Create seed scripts for:

* **Country list** (residence & destination):
  Popular: US, Canada, UK, Ireland, Australia, New Zealand, Singapore, Germany, France, Netherlands, Italy, Spain, Portugal, Switzerland, Austria, Sweden, Norway, Denmark, Finland, Belgium, Poland, Czechia, Hungary, Greece, Estonia, Latvia, Lithuania, Japan, South Korea, Hong Kong SAR, UAE, Saudi Arabia, Qatar, Oman, Malaysia, Thailand.
  **CIS/Medicine**: Russia, Kazakhstan, Kyrgyzstan, Uzbekistan, Armenia, Azerbaijan, Belarus, Georgia.
* **Selector rules** (few starter entries, e.g., Canada migrate, US study, Australia study): fill with placeholder min_scores and caveats.
* **Exam items**: 3–5 items per exam code with simple MCQs to make screens functional.

## API Routes (Next.js Route Handlers)

Implement with server auth checks and return typed JSON.

* `POST /api/profile`
  Body: { name, age, residence_country, destination_countries[], purpose_multi[] }
  Upserts `profiles` and `study_profile`.

* `GET /api/recommendations`
  Query uses user’s `study_profile`; returns grouped cards: by Purpose → Destinations → Tests. Data comes from `selector_rules` + hardcoded catalog for now.

* `POST /api/attempts/start`
  Body: { exam_code, mode }
  Logic: compute plan from profile → check entitlements (see gating) → if allowed, create `attempts` row and increment counters; return attemptId.

* `POST /api/attempts/submit`
  Body: { attempt_id, payload }
  Updates attempt, (optional) calls scoring stubs, returns score + feedback.

* `GET /api/entitlements`
  Returns current plan + remaining quota (next reset timestamps).

* `POST /api/stripe/create-checkout-session`
  Body: { priceId } → returns Stripe session URL.

* `POST /api/stripe/webhook`
  Handles `checkout.session.completed`, `customer.subscription.updated|deleted` → updates `profiles.plan` & `current_period_end`.

* `POST /api/score/writing` (stub)
  Input: { attempt_id, text, prompt_meta } → returns mock bands + improvements.

* `POST /api/score/speaking` (stub)
  Input: { attempt_id, audio_url, prompt_meta } → returns mock bands + transcript.

## Gating Logic (Server-Side)

Pseudocode to implement in `/api/attempts/start`:

```ts
// resolve plan
const plan = profile.plan ?? 'free';
const now = new Date();

// reset windows
if (!sameDay(profile.last_reset_daily, now)) daily_tests_used = 0, last_reset_daily = now;
if (!withinRolling7Days(last_reset_weekly, now)) weekly_tests_used = 0, last_reset_weekly = now;

// enforce
if (mode === 'full') {
  if (plan === 'guest') block(401, 'Create a free account to practise full tests.');
  if (plan === 'free' && weekly_tests_used >= 1) block(402, 'Free allows 1 test/week. Try again after reset.');
  if (plan === 'premium' && daily_tests_used >= 1) block(402, 'Premium allows 1 test/day. Try again after reset.');
  // unlimited: no cap
}
```

Edge: partial attempts don’t count until submitted or after 72h timeout; handle with cron/edge job.

## Pages & Components

* `/` (Landing)

  * Hero: H1 “Ace IELTS, TOEFL & PTE faster.”
  * Sub: “AI feedback, realistic mocks, country-specific guidance.”
  * CTA: **Start Practicing** → `/onboarding`
  * Value cards (4), How-it-works (3 steps), Pricing teaser, Footer.

* `/onboarding`

  * Stepper UI (4 steps).
  * Validate required fields; write `profiles` + `study_profile` then route `/recommendations`.

* `/recommendations`

  * Chips for Destinations & Purposes.
  * Cards grouped by Purpose → Destination: each test card shows About, Format, Duration, Typical score targets, Caveats (note: data placeholder), and **Start Practice**.
  * When “Start Practice”: if Sample Test → open sample; if Full Test → POST `/api/attempts/start` (enforces plan).

* `/practice/[examCode]`

  * Tabs: Sample (view), Full (timed).
  * Minimal test runner: section list, timer, autosave, submit.
  * On submit → POST `/api/attempts/submit` → show score/feedback (stubbed).

* `/dashboard` (plan-aware)

  * If Free: locked preview with upgrade CTA.
  * If Premium/Unlimited:

    * Charts: Band/Score trend (line), Skill radar, Error heatmap, Time on task.
    * Recommendations block (weak skills).
    * Use mock data if needed.

* `/pricing`

  * Cards with plan features.
  * Buttons create Stripe checkout sessions.

* `/account`

  * Profile, plan status, “Manage billing” link, data export/delete.

### UI/Copy Snippets

* Wizard headers: “Basics → Where you live → Where you’re going → Your purpose”
* Purpose helper: “Choose all that apply: Study, Migrate, Professional Development.”
* Paywall: “Create a free account to practise full tests and get instant feedback.”
* Free quota hit: “You’ve used your free test this week. Upgrade for more practice.”

## Styling

* Tailwind + modern, accessible defaults.
* Buttons: rounded-xl, focus-visible rings.
* Typography: Inter.
* Dark/light theme switch.

## Stripe

Create products/prices:

* `premium_monthly` = $3.99, `premium_yearly` = $39.00
* `unlimited_monthly` = $9.99, `unlimited_yearly` = $99.00
  On webhook, set `profiles.plan` accordingly and `current_period_end`.

## Env Vars (`.env.local`)

```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_POSTHOG_KEY=
OPENAI_API_KEY= # for later, not required to run stubs
```

## Tests / Acceptance

* ✅ Onboarding writes profile + study_profile; multi-select Destinations and multi-select Purpose work.
* ✅ Recommendations renders grouped tests by Purpose and Destination.
* ✅ Sample tests viewable by everyone; starting Full Test enforces plan rules.
* ✅ Free: exactly 1 full test per rolling 7 days; Premium: 1 per 24h; Unlimited: no cap.
* ✅ Dashboards locked for Free; fully visible for Premium/Unlimited.
* ✅ Stripe checkout upgrades the plan; webhook updates DB.
* ✅ Lighthouse: LCP < 2.5s, CLS < 0.1 on landing.

## Developer Notes

* Provide seed scripts (TS/SQL) for countries, selector rules, and a few exam items.
* Abstract plan/entitlement checks into a shared server util.
* Keep UI copy in a constants file for future i18n.
* Use Zod for input validation on API routes.
* Comment all TODOs where mock data stands in for a future FastAPI scoring service.

---

**Deliverables**:

* Full Next.js app with pages, components, API routes, DB schema/migrations, seed scripts, and basic tests.
* Ready to run locally (`pnpm dev`) with Supabase & Stripe test keys.
* Clear README with setup steps.

Build now.
