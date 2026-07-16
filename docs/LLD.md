# SweatDash — Low-Level Design

## Architecture

```
Web dashboard (supabase-js CDN, magic link)         WhatsApp (user's own number)
   |                                                       ^  |
   |  HTTPS                                   confirmation |  | text or voice note
   v                                                        |  v
Cloudflare Worker (single worker, TypeScript ESM)
   |-- Workers Assets: dashboard (public/)
   |-- /api/link                 -> generate a linking code (prove WhatsApp ownership)
   |-- /api/whatsapp/webhook     -> GET verify challenge, POST inbound (Meta signature auth)
   |-- /api/workouts             -> list/correct parsed workouts
   |-- /api/weekly-card/:week    -> fetch a generated card
   |-- /api/me
   |-- cron: weekly-card (weekly, proactive push — needs an approved template)
   |
   |  plain fetch                              Workers AI (Whisper, voice -> text)
   v                                                |
Supabase (GoTrue, PostgREST, RPC spend_credit)       v
                                              Claude Sonnet (text -> structured workout)
```

Inbound flow: a WhatsApp message arrives at `/api/whatsapp/webhook` -> if it's a voice note, the audio is transcribed via Workers AI Whisper first (30-second chunking, the same pattern already proven in the ai-caption-studio project) -> the resulting text (or the original text message) goes to Claude for structured parsing -> the parsed `workouts` row is inserted -> a short confirmation is sent back to the same WhatsApp thread. Because this reply is within the 24-hour customer-service window opened by the user's own inbound message, **no pre-approved template is needed for the confirmation** — a meaningful contrast with DoseLark, where every send is proactive. The only proactive (template-requiring) send in this product is the weekly progress-card push from cron.

## Data model

```sql
-- Proves WhatsApp ownership: user generates a code on the web, texts it from their WhatsApp
-- number, the webhook handler matches code -> user_id and binds profiles.whatsapp_number.
create table link_codes (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id),
  code text not null unique,
  expires_at timestamptz not null,
  used_at timestamptz
);
alter table link_codes enable row level security;
create policy link_codes_own on link_codes for select using (auth.uid() = user_id);

-- profiles.whatsapp_number (unique, nullable until linked) added alongside the standard
-- profiles.credits / plan / subscription_expires_at columns from the reference schema.

create table workouts (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id),
  raw_message text not null,                 -- original text, or the Whisper transcript
  source text not null check (source in ('whatsapp_text', 'whatsapp_voice')),
  exercise_type text not null,                -- 'strength' | 'cardio' | 'other'
  structured jsonb not null,                  -- shape depends on exercise_type — see LLM schema
  confidence text not null check (confidence in ('high', 'medium', 'low')),
  needs_review boolean not null default false,
  logged_at timestamptz not null,             -- when the workout happened (parsed or defaulted to now)
  created_at timestamptz not null default now()
);
alter table workouts enable row level security;
create policy workouts_own on workouts for select using (auth.uid() = user_id);

create table personal_records (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id),
  exercise_type text not null,
  metric text not null,                       -- 'max_weight' | 'best_time' | 'longest_distance'
  value numeric not null,
  unit text not null,
  workout_id uuid references workouts(id),
  achieved_at timestamptz not null
);
alter table personal_records enable row level security;
create policy personal_records_own on personal_records for select using (auth.uid() = user_id);

create table weekly_cards (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id),
  week_start date not null,
  content jsonb not null,
  sent_at timestamptz,
  unique (user_id, week_start)
);
alter table weekly_cards enable row level security;
create policy weekly_cards_own on weekly_cards for select using (auth.uid() = user_id);
```

No complex state machine — a `workout.needs_review` boolean flag (set when the parse `confidence` is `low`) is the only "pending" state, cleared when the user edits/confirms the entry in the dashboard.

## API routes

| Route | Method | Auth | Behavior | Failure modes |
|---|---|---|---|---|
| `/api/link` | POST | JWT | Generates a 6-digit code, 15-minute expiry | — |
| `/api/whatsapp/webhook` | GET | none | Meta verification challenge | 403 on token mismatch |
| `/api/whatsapp/webhook` | POST | Meta signature | Handles a linking code, or (if already linked) parses a workout and spends 1 credit on the free tier | 401 bad signature; unmatched/expired code gets a plain-text "code not found, try again" reply |
| `/api/workouts` | GET/PATCH | JWT | List workouts, correct a `needs_review` entry | 404 not owned |
| `/api/weekly-card/:week` | GET | JWT | Fetch a generated card (subscriber-only past the free tier) | 403 if not subscribed and past the free-tier window |
| `/api/me` | GET | JWT | Credits, plan, linked number, workout count | 401 unauthenticated |

Cron: `weekly-card` (weekly) generates and pushes each subscribed user's card via an approved WhatsApp template.

## LLM strategy

Provider: Anthropic, `claude-sonnet-4-6` for both parsing calls below; Workers AI `@cf/openai/whisper-large-v3-turbo` for voice transcription (~$0.00051/min per the challenge's own verified OSS/API research — cheap enough that per-log transcription cost is negligible next to the Rs149/mo price).

**1. Text-to-schema parse (sync, per inbound message).** The transcript or text message goes to Claude with a system prompt instructing it to extract a structured workout and nothing else. `output_config.format` json_schema sketch:

```json
{
  "type": "object",
  "required": ["exercise_type", "confidence"],
  "properties": {
    "exercise_type": { "type": "string", "enum": ["strength", "cardio", "other"] },
    "confidence": { "type": "string", "enum": ["high", "medium", "low"] },
    "strength": { "type": "object", "properties": {
      "exercise_name": { "type": "string" },
      "sets": { "type": "array", "items": { "type": "object", "properties": {
        "reps": { "type": "integer" }, "weight_kg": { "type": "number" } } } }
    }},
    "cardio": { "type": "object", "properties": {
      "activity": { "type": "string" },
      "distance_km": { "type": "number" }, "duration_min": { "type": "number" } } }
  }
}
```

**2. Weekly progress card (sync, once per subscriber per week, from cron).** Input is the week's structured `workouts` rows plus any new `personal_records`; output is card content (headline, per-activity stats, PR callouts).

**Hard wellbeing guardrail on both calls, enforced in the system prompt and checked by a keyword guard on the output before it's stored or sent:** never estimate calories burned, never suggest a diet or nutrition change, never comment on body composition or weight loss — this product logs *activity*, full stop. This is the same "guard the output before it ships" pattern as DoseLark's no-medical-guidance check, applied to a different boundary.

Cost estimate (planning-only, coarse): a text parse call is small (~300 input / ~150 output tokens, well under a cent). A voice note adds ~$0.0003 for a typical 30-second clip. A weekly card aggregates maybe 5-10 workouts (~800 input tokens) into a short card (~300 output tokens), also well under a cent. Total LLM cost per active user per week is a few cents against a Rs149/mo (~$1.75/mo) subscription — ample margin.

## Frontend pages

- `/` — landing + pricing (Rs149/mo, 15 free logs), wellbeing-guardrail statement ("activity logging only — no diet or calorie advice").
- `/link` — shows the linking code and instructions to text it from WhatsApp.
- `/app` — workout history, inline correction UI for `needs_review` entries.
- `/app/cards` — weekly progress card history.
- `/account` — credits/plan, subscription upgrade instructions.

## Error handling and credits/subscription flow

Free tier: `spend_credit` is called once per successfully parsed workout (15 free credits granted at signup); once exhausted, `plan='active'` gates further logging *and* the weekly card feature entirely (a free user never receives a card). A failed parse (Claude error, malformed JSON, Whisper failure) does **not** spend a credit and replies to the WhatsApp thread with a plain "couldn't parse that, try rephrasing" message — no refund logic needed because the spend happens only after a successful parse, unlike the reference's spend-then-refund-on-failure pattern (there's no async dispatch here to fail after the spend).

**Monetization is a deliberate deviation from the reference's pure per-op credit model in its ceiling, not its mechanism**: unlike the reference's 1 free credit, this product grants 15 (a single free credit wouldn't demonstrate the product's value — logging is the whole point, and one log proves nothing). Beyond the free tier, `profiles.plan`/`subscription_expires_at` gates continued logging and all weekly cards, same subscription-over-credits pattern as ParcelPick and DoseLark in this batch. Manual UPI top-up day 1, operator sets `plan='active'` after confirming payment.

## Integrations and launch gates

**WhatsApp — lighter gate than DoseLark's.** Inbound logging and its confirmation reply both happen inside the 24-hour session window opened by the user's own message, so **no template approval is required to ship the core logging flow.** Only the proactive weekly progress-card push needs an approved template, and that's a lower-stakes, non-time-critical send (unlike DoseLark's medication reminders, a delayed or missing weekly card degrades the product without any safety implication) — so this product can plausibly launch its core flow while template approval for the weekly card is still pending, shipping that feature as soon as the template clears. Business verification is still required for a production WhatsApp Business Account regardless of template status; start it early even though it isn't blocking day-one logging.

## Security notes

- RLS scopes `link_codes`, `workouts`, `personal_records`, and `weekly_cards` to `auth.uid() = user_id`; the Worker uses the service-role key for writes.
- `/api/whatsapp/webhook` is authenticated by Meta's `X-Hub-Signature-256` HMAC, never a user JWT.
- Account linking requires the code to arrive **from the WhatsApp number being linked**, be unused, and be within its 15-minute expiry — this proves the requester controls that number before binding `profiles.whatsapp_number`.
- Voice notes are transcribed and then discarded — the raw audio is not retained beyond the transcription call; only the resulting text is stored in `workouts.raw_message`.
- Secrets (`ANTHROPIC_API_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `WHATSAPP_ACCESS_TOKEN`, `WHATSAPP_APP_SECRET`) via `wrangler secret put`.

## Out of scope for v1

- Nutrition or calorie tracking of any kind — this is a permanent product boundary, not a v1 gap.
- Apple Health / Google Fit / wearable sync.
- Coaching, training plans, or programmed workouts.
- Multi-user leaderboards or social features.
- Any channel besides WhatsApp (no native app, no SMS fallback).
- Razorpay-automated billing — v1 is manual UPI + admin-set `plan`.
