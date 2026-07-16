# SweatDash
Log workouts by WhatsApp text or voice note, get weekly progress cards and PRs — no app to open.

**Status: planned — not yet built (50-SaaS challenge #55)**

## The problem
Fitness enthusiasts who already track workouts informally ("bench 4x8 @60kg, ran 5k") don't want to fill in a structured app form mid-set. They'll send a WhatsApp message to a friend or a notes app instead. SweatDash captures that same freeform habit — text or a voice note — and turns it into structured history, personal records, and a weekly recap, with zero app install.

## Target buyer
Fitness enthusiasts who log workouts casually and have tried (and abandoned) at least one fitness-tracking app because the data entry was too much friction.

## Pricing hypothesis
Rs149/month subscription. Free tier: 15 logged workouts, then a subscription is required for continued logging and for weekly progress cards.

## Stack
- Cloudflare Worker (TypeScript, Workers Assets) serving the web dashboard and `/api/*`.
- Supabase: magic-link auth (linked to a WhatsApp number), Postgres + RLS.
- WhatsApp Cloud API for inbound logging (text and voice notes); Workers AI Whisper for voice transcription (same pattern already proven in the ai-caption-studio project); Claude (`claude-sonnet-4-6`) parses freeform text into a structured workout schema and drafts the weekly progress card.

## How to continue this build
Read `docs/LLD.md` for architecture, data model, the LLM parsing strategy, and the WhatsApp integration notes, then `docs/PLAN.md` for the ordered TDD task list. `CLAUDE.md` points back to the reference implementation.

## Risks / constraints
- **The LLM parses freeform logs into a structured schema** (exercise, sets/reps/weight or distance/duration) — it is a best-effort parser, not a source of truth; low-confidence parses are flagged for the user to confirm rather than silently guessed.
- **Wellbeing guardrail: no diet or calorie prescriptions, ever.** SweatDash logs activity only. The parsing and progress-card prompts are hard-constrained to never estimate calories burned, suggest a diet, or comment on body composition — this is a product boundary, not a nice-to-have, and applies to every LLM call in this repo.
- WhatsApp inbound logging (replies within the user's own 24-hour session) needs no pre-approved message template, but the proactive weekly progress-card push does — same Meta approval process as any other WhatsApp-first product in this challenge, though lighter-weight here since the core logging flow works without waiting on template approval.
