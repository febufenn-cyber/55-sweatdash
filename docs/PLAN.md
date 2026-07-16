# SweatDash — Build Plan

TDD throughout: write the test first, watch it fail for the right reason, then implement. Each task is sized for one focused agent session.

## T1 — Supabase schema and RLS
Files: `supabase/migrations/0001_init.sql`.
Interfaces: none (schema only).
Test first: RLS check — a second user's JWT cannot `select` another user's `workouts`/`personal_records`/`weekly_cards` rows.
Done when: migration applies cleanly; RLS verified with two test JWTs; `weekly_cards` unique constraint on `(user_id, week_start)` confirmed via a duplicate-insert test.

## T2 — Worker skeleton and static shell
Files: `wrangler.jsonc` (weekly-card cron trigger), `src/worker.ts`, `src/router.ts`, `public/index.html`, `public/config.js` route.
Interfaces: `router(req: Request, env: Env): Promise<Response>`; `scheduled(event, env)` export for the weekly-card cron.
Test first: `test/router.test.ts` — unknown route 404, `/api/me` without auth 401.
Done when: `wrangler dev` serves the shell and stub routes locally.

## T3 — Supa client
Files: `src/supa.ts`.
Interfaces: `class Supa { getUser(jwt); createLinkCode(userId): Promise<string>; resolveLinkCode(code, fromNumber): Promise<string|null>; insertWorkout(row); listWorkouts(userId); updateWorkout(id, userId, patch); upsertPersonalRecord(row); getWeekWorkouts(userId, weekStart): Promise<WorkoutRow[]>; upsertWeeklyCard(row); spendCredit(userId); getProfile(userId); }` — injectable `fetchFn`, mirrors the reference `Supa` shape.
Test first: `test/supa.test.ts` with mocked fetch asserting URLs/headers per method.
Done when: all methods pass against mocked fetch.

## T4 — Account linking flow
Files: `src/handlers.ts` (add `handleCreateLink`), `src/linking.ts`.
Interfaces: `resolveLink(code: string, fromNumber: string, supa: Pick<Supa,'resolveLinkCode'>): Promise<{userId: string} | {error: string}>`.
Test first: an expired code is rejected; a code sent from a number that doesn't match is still bound to the number that texted it (not spoofable from the web side); a used code cannot be reused.
Done when: all three cases pass.

## T5 — Voice transcription module (Workers AI Whisper)
Files: `src/transcribe.ts`.
Interfaces: `transcribeVoiceNote(audioBytes: Uint8Array, env: {AI: Ai}): Promise<string>` — 30-second chunking for anything longer, matching the ai-caption-studio pattern.
Test first: a mocked Workers AI binding returns concatenated text across chunk boundaries in order.
Done when: chunking and ordering are correct against a mocked `env.AI.run`.

## T6 — Claude parse module with the wellbeing guardrail
Files: `src/parse.ts`, `src/prompt/parse-system.ts`, `src/guard.ts`.
Interfaces: `parseWorkout(opts: {text: string, env: ReviewEnv, fetchFn?: typeof fetch}): Promise<{workout: ParsedWorkout, model: string}>`; `containsNutritionGuidance(text: string): boolean` — a keyword/pattern guard checked against the raw model output before it's ever stored.
Test first: a mocked Claude response with a calorie estimate or diet suggestion is caught by `containsNutritionGuidance` and the parse is rejected as `needs_review`; a clean activity-only response passes through; malformed JSON raises the same `ParseError` taxonomy as the reference's `ReviewError`.
Done when: the guardrail test, the malformed-JSON test, and a happy-path strength/cardio parse all pass.

## T7 — WhatsApp webhook: link-or-parse dispatch, credit spend, confirmation reply
Files: `src/whatsapp.ts`, `src/handlers.ts` (add `handleWhatsappVerify`, `handleWhatsappWebhook`).
Interfaces: `sendMessage(opts: {to: string, body: string, env: WhatsappEnv, fetchFn?: typeof fetch}): Promise<void>` (free-form reply, no template needed — inside the 24h session window); `verifyWebhookSignature(rawBody, signatureHeader, appSecret): boolean`.
Test first: a bad signature is rejected with 401 before touching the database; an unlinked number's message triggers the link flow, not the parse flow; a linked number with zero credits and `plan='free'` gets a plain-text upgrade prompt instead of a parse attempt.
Done when: signature verification, link/parse dispatch, and the credit gate all pass with mocked payloads.

## T8 — Workouts list/correct handlers and PR detection
Files: `src/handlers.ts` (add `handleListWorkouts`, `handleUpdateWorkout`), `src/personal-records.ts`.
Interfaces: `detectPersonalRecord(workout: ParsedWorkout, existing: PersonalRecordRow[]): PersonalRecordRow | null` — pure function, no LLM.
Test first: a strength set that beats the stored `max_weight` for that exercise produces a new PR row; one that doesn't, doesn't.
Done when: PR detection is deterministic and covered for both strength and cardio metrics.

## T9 — Weekly card generation and cron push
Files: `src/cron/weekly-card.ts`.
Interfaces: `generateWeeklyCard(opts: {counts: WorkoutRow[], prs: PersonalRecordRow[], env: ReviewEnv}): Promise<{content: WeeklyCardContent}>` reusing the T6 guardrail.
Test first: the unique constraint prevents a duplicate card if the cron fires twice for the same week; a free-tier user (no `plan='active'`) is skipped entirely, not just un-sent.
Done when: both cases pass.

## T10 — Frontend: dashboard, linking flow, and progress cards
Files: `public/app.js`, `public/link.js`, `public/cards.js`, `public/styles.css`.
Interfaces: none new (calls `/api/*` from T3-T9).
Test first: a simple string-match check that the "activity logging only, no diet or calorie guidance" statement appears on the landing page and pricing page.
Done when: a user can generate a link code, see it reflected once a simulated WhatsApp message links their number, browse their workout history and PRs, and view the latest weekly card, against a local `wrangler dev` + real Supabase dev project.

## T11 — Deploy, live smoke test, launch checklist
Files: `scripts/smoke.ts`, deployment config.
Interfaces: `smoke.ts` runs against the deployed URL: creates a link code, simulates a text-log webhook call, asserts a `workouts` row and (if it beats nothing) no spurious PR.
Done when: `wrangler deploy` succeeds; `scripts/smoke.ts` passes against production; pricing page shows Rs149/mo + 15 free logs and the "activity logging only" guardrail statement; secrets set via `wrangler secret put`; WhatsApp Business verification confirmed (needed for the weekly-card template even though core logging doesn't require one).
