# GAMZU — Comprehensive Project Briefing

**Date:** April 19, 2026
**Audience:** Gemini (or any LLM) coming in cold to advise on next moves
**Purpose:** A single dense document that captures vision, architecture, what's built, what was deferred, what's broken, and what's next.

---

## 1. The Vision in One Paragraph

GAMZU is a full-stack **spiritual productivity OS** built around the 22 Hebrew letters as archetypal lenses for self-reflection. It uses **three LLM "brains"** in a deliberate division of labor — Anthropic Claude as **The Poet** (emotional/Harmony scoring), OpenAI GPT-4o as **The Scientist** (Hebrew root and linguistic decode), Google Gemini as **The Historian** (calendar, time-context, synthesis). The product surface is two artifacts: a **web app** styled like macOS Tahoe 2026 ("Liquid Glass"), and a **GAMZU Companion** Android app targeted at OPPO ColorOS / Find N hardware that acts as an agentic phone-controlling layer with NPU wake-word, AppFunctions for Breeno (Oppo's voice assistant), and a confidential "Mind Space" capture surface.

---

## 2. Repository Layout (pnpm monorepo)

```
artifacts/
  api-server/          Express 5 backend (the brain bus + auth + persistence)
  gamzu/               React + Vite web app — "GAMZU OS" desktop shell
  gamzu-companion/     Expo React Native — Android Companion APK
  mockup-sandbox/      Vite component sandbox for canvas-based design exploration

lib/
  api-spec/            OpenAPI 3 source of truth → orval codegen
  api-client-react/    Generated React Query client (do not hand-edit)
  api-zod/             Generated Zod schemas (do not hand-edit)
  db/                  Drizzle ORM schema + migrations
  letters/             The 22 Hebrew letters source (name, char, archetype, gematria, theme)
  replit-auth-web/     Replit Auth (OIDC+PKCE) frontend hook
```

**Stack:** Node 24, TypeScript 5.9, Express 5, PostgreSQL + Drizzle ORM, Zod (`zod/v4`), Orval codegen, esbuild, Vite, Expo SDK + EAS Build, Replit Auth (OIDC), shadcn/ui + Tailwind, Wouter routing, TanStack Query, Drizzle.

---

## 3. The Three-Brain Orchestrator (the heart of the product)

Endpoint: `POST /api/gamzu/architect` (in `artifacts/api-server/src/routes/gamzu.ts`).

For a given user message about a Hebrew letter:
1. **Scientist (GPT-4o)** decodes the letter — root, gematria, sound, linguistic threads.
2. **Poet (Claude)** runs an EQ Audit — produces a `harmonyScore` 0–100 plus felt-sense reflection.
3. **Historian (Gemini 2.5 Flash)** synthesizes both perspectives + historical/calendrical context into a `brainSynthesis` field.

The `ArchitectResponse` schema lives in `lib/api-spec/openapi.yaml`. The synthesis is rendered as a "Brain Synthesis" card in `artifacts/gamzu/src/components/gatekeeper-chat.tsx`.

Harmony score flows globally via the Zustand store in `artifacts/gamzu/src/lib/store.tsx` and shows as a colored dot in the menubar (green ≥70, amber ≥40, red <40).

---

## 4. The Web App ("GAMZU OS Shell")

Location: `artifacts/gamzu/`.

**Visual identity:** macOS Tahoe 2026 "Liquid Glass" — translucent panels, refractive backdrops, subtle motion. Custom CSS variables; no Material/iOS chrome.

**Layout primitives:**
- `OsMenubar` (`components/os-menubar.tsx`) — top ~40px bar: active letter on the left, GAMZU wordmark center, live clock + Harmony Dot + Shabbat toggle + user dropdown right.
- `Desktop` (`components/layout.tsx`) — flex-1 region with a faint Hebrew letter motif tied to the active letter; container queries enable a two-column layout for Galaxy Fold / OPPO Find N inner displays.
- `OsDock` (`components/os-dock.tsx`) — bottom strip: all 22 Hebrew letter icons (one-shot spring-pop on select) + Agent / Saved / Settings shortcuts.
- `FloatingPanel` — draggable on desktop, fullscreen on mobile. Used for Wisdom, Sessions, Harmony Trend.

**Persistence:** session UUID in `localStorage` (`gamzu_session_id`) → backend `chat_sessions`. Every message persisted in `chat_messages`. Each EQ audit appends to `harmony_scores` for the trend chart. Active letter and Shabbat-mode keys are migrated from guest → user-scoped on first sign-in.

**Special components:**
- `ShabbatAutoPrompt` — once-per-week localStorage-keyed prompt that auto-suggests enabling Shabbat mode in the pre-Shabbat window and disabling post-Shabbat. Driven by `GET /api/gamzu/shabbat-status` (UTC-safe `getShabbatStatusFromUtcMs`).
- `HarmonyTrendChart` — Recharts line of harmony over time with the Chesed threshold (70) marked.
- `SavedReflections` — sheet of starred letter reflections.
- `AgentPage` — the agentic phone-control control panel (pairing, action history).

---

## 5. The Companion App (`artifacts/gamzu-companion/`)

Expo React Native app with a custom Dev Client and EAS Build pipeline. Targets Android 17+ / ColorOS 17 (OPPO Find N5). Uses `expo-router` (file-based routing under `app/`).

**Screens:**
- `app/(tabs)/index.tsx` — Companion Home: pairing flow, Live Context card (Shabbat banner, time/season), AppFunctions display.
- `app/(tabs)/actions.tsx` — Polls API, executes approved actions, reports back.

**Native integrations (via local Expo plugins in `plugins/`):**
- `withAndroidAudioFocus.js` — handles audio focus / ducking for voice mode.
- `withAppFunctions.js` — declares 7 App Functions for **Breeno discovery** in `app_actions.xml` and `app_function_metadata.xml`, AND uses `withDangerousMod` to write `AppFunctionService.kt` during prebuild. Functions: `StartLetterSession`, `GetHarmonyScore`, `CreateChesedTask`, `SendChesedMessage`, `CreateCalendarEvent`, `CreateReminder`, `SaveNote`. **Handler bodies are stubs that log to Logcat** — they need real HTTP wiring on hardware.

**Pairing:** PWA generates a 64-hex `X-Pairing-Token`; companion stores it. Companion polls `GET /api/agent/poll` for approved actions, executes via deep links (sms:, wa.me/, calendar intent, notes app), reports back via `POST /api/agent/complete`.

---

## 6. The Agentic Loop ("GAMZU Nervous System")

Lifecycle: **propose → Chesed gate → pending (user confirms in PWA) → approved → companion executes → completed/failed.**

- **Chesed gate:** Every proposed action goes through Claude for a 0–100 *Chesed* (loving-kindness) score. ≥70 = approved; below = blocked with reasoning surfaced to the user.
- **No direct PWA→phone:** Avoids mixed-content / firewall issues. Companion polls. Server is the bus.
- **Routes:** `artifacts/api-server/src/routes/agent.ts` — `propose, approve, cancel, complete, poll, pair, status`.
- **AppFunction dispatch:** `artifacts/gamzu-companion/lib/executor.ts` translates server payloads into Android intents.

---

## 7. Database Schema (`lib/db/src/schema/index.ts`)

Tables (all PostgreSQL via Drizzle):
- `users` — OIDC profile.
- `sessions` — Replit Auth session (sid, sess JSONB, expire).
- `chat_sessions` — one per browser session UUID. **`privacy_tag` column added (default `standard`).**
- `chat_messages` — JSONB message payloads. **`privacy_tag` column added (default `standard`).**
- `harmony_scores` — every EQ audit score for trends.
- `user_rate_limits` — IP/user limiter state.
- `user_preferences` — granular notification + Shabbat + voice prefs.
- `user_behavior_events` — analytics for adaptive UI.
- `favorite_reflections` — starred letter reflections.
- **`mind_space_snaps` (NEW) — id, userId, source, snapKind, payload jsonb, categorizedLetter, confidence, `privacy_tag` default `confidential`, createdAt. Indexes on (user, created) and categorizedLetter.**
- `agent_actions` — proposed/approved/completed action ledger with Chesed score.
- `synthesis_feedback` — user feedback on brain synthesis quality.
- `companion_pairings` — one row per user: pairing token, capabilities, lastSeenAt.
- `user_tasks` — personal task queue (voice-created, web-reviewed).

**Privacy helper:** `artifacts/api-server/src/lib/privacy.ts` exports `PrivacyTag` type (`standard | confidential | ephemeral`), `logConfidentialEvent()` for `[PRIVACY:CONFIDENTIAL]` audit logging, and `redactForLog()` for safe value masking.

---

## 8. Authentication

**Replit Auth (OpenID Connect + PKCE)** — fully session-based, no client-side JWT.

- `GET /api/login` → starts OIDC, sets PKCE cookies, redirects to Replit.
- `GET /api/callback` → exchanges code, upserts user, sets `sid` session cookie.
- `GET /api/logout` → clears session, hits Replit end-session URL.
- `GET /api/auth/user` → returns current user (or null).
- `useAuth()` hook in `lib/replit-auth-web/src/use-auth.ts`.
- `AuthGate` in `artifacts/gamzu/src/App.tsx` redirects unauthenticated users.
- Protected: `/api/gamzu/eq-audit`, `/api/gamzu/architect` via `requireAuth` middleware.

---

## 9. API Routes Inventory (`artifacts/api-server/src/routes/`)

| Route file | Purpose |
|---|---|
| `agent.ts` | Agentic loop (propose/approve/poll/complete/pair) |
| `analytics.ts` | Behavior event ingestion |
| `auth.ts` | OIDC login/callback/logout/user |
| `companion.ts` | APK update metadata, version pinning |
| `conversation.ts` | Gemini Live conversation mode (Task #156 foundation) |
| `favorites.ts` | Saved reflections CRUD |
| `feedback.ts` | Synthesis feedback collection |
| `gamzu.ts` | The big one: `/architect`, `/eq-audit`, `/decode`, `/shabbat-status` |
| `guest-chat.ts` | Public chatbot with IP rate limiting |
| `health.ts` | Liveness |
| `history.ts` | Chat history queries |
| `index.ts` | Router composition |
| `oppo.ts` (NEW) | `POST /api/oppo/mind-space-webhook` — bearer-auth required |
| `preferences.ts` | User preferences CRUD |
| `tasks.ts` | Personal task queue CRUD + AI suggestions |
| `voice.ts` | Voice handshake + voice conversation mode |

---

## 10. The Oppo Agent Matrix Pivot (the most recent change)

User defined 5 workstreams and chose to **fully build #2 + #5 and scaffold #1**. **#3 and #4 are deferred** because they require native Android development that cannot run in Expo Go and need a custom EAS dev-client APK on real ColorOS 17 hardware.

| # | Workstream | Status | Notes |
|---|---|---|---|
| 1 | **AppFunctions scaffold** | ✅ Scaffolded | All 7 functions declared. `AppFunctionService.kt` written by `withDangerousMod`. Handlers are stubs. |
| 2 | **Mind Space webhook** | ✅ Built + tested | `POST /api/oppo/mind-space-webhook`, bearer-token auth (`OPPO_WEBHOOK_SECRET`), Gemini Historian categorization, persistence to `mind_space_snaps` with `privacyTag=confidential`. Fail-closed: returns 503 if secret unset. |
| 3 | **O+ Connect notification audit** | ⏸️ Deferred | Needs native `NotificationListenerService` + `BIND_NOTIFICATION_LISTENER_SERVICE` permission. Use case: Poet scores incoming iPhone notifications and proposes a "Spiritual Buffer" before the user answers. |
| 4 | **Liquid Glass Accessibility Overlay** | ⏸️ Deferred | Needs native `AccessibilityService` + `SYSTEM_ALERT_WINDOW` permission. Re-shell of the companion from Activity-based to overlay-based. Status-bar Live View → centered refractive dashboard ripple animation. |
| 5 | **Confidential data tagging** | ✅ Built | `privacyTag` columns added; `mind_space_snaps` defaults to confidential; `privacy.ts` helper with audit logging. |

**Webhook security shape (important):**
1. If `OPPO_WEBHOOK_SECRET` env var is unset → 503 (fail-closed).
2. If `Authorization: Bearer <token>` header missing/malformed → 401.
3. If token doesn't match (timing-safe compare) → 401.
4. Then: payload validation → Gemini categorization → persist with confidential tag → audit log.

---

## 11. What's Live and Working

- ✅ Three-brain orchestrator (Scientist + Poet + Historian) producing brainSynthesis.
- ✅ GAMZU OS Shell (Liquid Glass desktop, dock, menubar, floating panels).
- ✅ Replit Auth (OIDC), per-user persistence, multi-tab sign-in dedup (Task #132).
- ✅ Conversation persistence (sessions + messages + harmony scores).
- ✅ Agentic loop end-to-end with Chesed gate.
- ✅ Companion pairing + action polling + execution.
- ✅ AppFunctions metadata declared (Breeno can discover them on hardware).
- ✅ Voice Conversation Mode foundation (Task #126 + #156, Gemini Live).
- ✅ Personal task queue (voice-created, web-reviewed).
- ✅ Granular notification preferences (Task #136).
- ✅ Shabbat auto-prompt + UTC-safe shabbat-status endpoint (Task #137).
- ✅ Mind Space webhook + privacy tagging (Oppo pivot #2 + #5).
- ✅ Harmony Score persistence across page refreshes (Task #131).
- ✅ Full CI pipeline — typecheck + build gate (Task #133).
- ✅ APK update script hooked into EAS build automation (Task #142).

---

## 12. Known Issues / Pre-existing Tech Debt

| Issue | Severity | Notes |
|---|---|---|
| **Gemini proxy `INVALID_ENDPOINT`** | High | `AI_INTEGRATIONS_GEMINI_BASE_URL` config issue causes 400 on `gemini-2.5-flash:generateContent` — affects ALL Gemini routes (architect synthesis, voice, conversation, tasks suggestions, oppo categorization). Needs proxy URL/version reconfiguration. |
| **Companion `setBaseUrl is not a function`** | Medium | `lib/api-client-react/src/generated/api.ts` is missing in companion runtime — codegen artifact not generated for Expo bundler. Companion crashes on startup until fixed. |
| **Companion uses fixed 18:00 sunset** | Medium | `app/(tabs)/index.tsx` has `SUNSET_APPROX = 18*60` for pre-Shabbat detection — seasonally/geographically wrong. Tracked as follow-up #172/#174. |
| **AppFunction handlers are stubs** | Expected | Real wiring requires EAS dev-client + Android 17/ColorOS 17 hardware. |
| **Mockup sandbox runs but unused** | Low | Available for design variant exploration but no active mockups. |

---

## 13. What's Cancelled (recent — context for Gemini)

The user has been **aggressively pruning** the task queue. Recently cancelled:
- #134 Custom daily letter notification time
- #135 Cleanup of broken push notification tokens
- #138 Sync location context between web + companion
- #139 Build APK with AppFunctions manifest entries (subsumed by Oppo pivot)
- #140 Deep link route for "Start Letter Session"
- #141 Show Android-AI-triggered actions in history
- #143 Notification intelligence chain
- #144 Chat-first Companion UI
- #145 LLM cost optimization
- #146 Real always-on wake-word listener (native build)
- #147 Accurate Shabbat times via real location
- #148 Save GAMZU response to queue from inside chat
- #150 Rate-limit task creation
- #151 Guest chat rate-limit persistence across restarts
- #152 Friendly UI message when guest chat limit hit
- #157 Auto-detect APK size in EAS post-build hook
- #158 Surface live APK version + download on Companion web page

**Pattern:** the user is consolidating around the Oppo pivot and Voice/Conversation mode as the primary axes, and dropping incremental polish + secondary features.

---

## 14. Pending / In-Flight Tasks (still on the queue)

- **#137** Shabbat auto-switch — MERGED.
- **#131** Harmony score persistence — DONE.
- **#156** Conversation Mode (Gemini Live) — foundation merged.

(All Shabbat follow-ups #172/#173/#174 were cancelled with the rest of the pruning sweep — the user has decided current Shabbat behaviour is good enough and is not investing further there.)

---

## 15. Recommended Next Moves (Gemini, this is your action surface)

Ordered by leverage (highest first):

1. **Fix the Gemini proxy config.** This single fix re-enables synthesis, voice, conversation, task suggestions, AND Mind Space categorization. Probably a base-URL change to match the v1beta vs v1 API version exposed by the Replit AI integrations proxy.

2. **Decide the EAS dev-client cadence.** Almost everything Oppo-shaped (#1 handler bodies, #3 O+ Connect, #4 Accessibility Overlay, real wake-word, real reminders) is blocked on having a Dev Client APK on hardware. Rather than scaffolding more stubs, the user should sit one full hardware-test cycle and validate what's already declared.

3. **Build #3 (O+ Connect listener) next.** This is the highest-emotional-leverage feature: Poet scores an incoming iPhone notification and proposes a "1-minute Spiritual Buffer" before the user answers. It requires:
   - A native `NotificationListenerService` Java/Kotlin class (write via `withDangerousMod` plugin).
   - `BIND_NOTIFICATION_LISTENER_SERVICE` permission + Settings deep link to grant it.
   - New endpoint `POST /api/oppo/harmony-audit` that takes notification metadata and calls Claude for a Harmony score.
   - UI: a transient overlay "Spiritual Buffer" prompt before opening the source app.

4. **Build #4 (Accessibility Overlay) — but only after a hardware test of #3.** This is a big architectural shift. Don't build it until #3 has validated the native-Android plugin pipeline end-to-end.

5. **Companion runtime fix.** Resolve `setBaseUrl is not a function` so the companion app actually boots. This is blocking ALL on-device testing.

6. **Surface privacy/confidential tagging in the UI.** Right now `privacyTag` is invisible to users. Add a small lock-glyph indicator to messages/snaps tagged confidential. Ephemeral support (auto-delete after N days) is not yet wired.

7. **Wire the Mind Space webhook to a real consumer.** Build a Tasker / OPPO Quick Capture profile or Shortcuts equivalent that POSTs snaps. Document the bearer-token setup. Add a Mind Space gallery view in the web app to browse categorized snaps by letter.

---

## 16. Crucial Files Index (for quick orientation)

**Backend:**
- `artifacts/api-server/src/routes/gamzu.ts` — three-brain orchestrator (canonical AI route)
- `artifacts/api-server/src/routes/oppo.ts` — Mind Space webhook (newest, well-commented)
- `artifacts/api-server/src/lib/privacy.ts` — privacy tagging helper
- `lib/db/src/schema/index.ts` — single source of truth for DB schema
- `lib/letters/src/index.ts` — the 22 letters (canonical list)
- `lib/api-spec/openapi.yaml` — API contract; codegen runs from here

**Web:**
- `artifacts/gamzu/src/App.tsx` — routing + AppProvider + AuthGate
- `artifacts/gamzu/src/components/gatekeeper-chat.tsx` — main chat UI + brain synthesis card
- `artifacts/gamzu/src/components/os-menubar.tsx`, `os-dock.tsx`, `layout.tsx` — Liquid Glass shell
- `artifacts/gamzu/src/components/shabbat-auto-prompt.tsx` — auto-Shabbat UX
- `artifacts/gamzu/src/lib/store.tsx` — Zustand global store

**Companion:**
- `artifacts/gamzu-companion/plugins/withAppFunctions.js` — Breeno + AppFunctionService Kotlin scaffold
- `artifacts/gamzu-companion/plugins/withAndroidAudioFocus.js` — voice audio focus
- `artifacts/gamzu-companion/app/(tabs)/index.tsx` — Companion Home
- `artifacts/gamzu-companion/lib/executor.ts` — AppFunction dispatch

---

## 17. How to Run / Build (for any agent)

```
# Typecheck everything
pnpm run typecheck

# Codegen freshness check (CI gate)
pnpm run check:codegen

# Re-generate API client + zod after openapi.yaml changes
pnpm --filter @workspace/api-spec run codegen

# Push DB schema (dev)
pnpm --filter @workspace/db run push

# Run individual artifacts
pnpm --filter @workspace/api-server run dev
pnpm --filter @workspace/gamzu run dev
pnpm --filter @workspace/gamzu-companion run dev
```

All four workflows are wired into Replit and run in parallel.

---

## 18. Environment Variables of Note

| Var | Purpose | Status |
|---|---|---|
| `OPENAI_API_KEY` | Scientist (GPT-4o) | ✅ set |
| `ANTHROPIC_API_KEY` | Poet (Claude) | ✅ set |
| `AI_INTEGRATIONS_GEMINI_API_KEY` | Historian (Gemini) | set but proxy config broken |
| `AI_INTEGRATIONS_GEMINI_BASE_URL` | Gemini proxy URL | **needs fixing** |
| `OPPO_WEBHOOK_SECRET` | Bearer token for Mind Space webhook | **not yet set** — webhook safely 503s |
| `DATABASE_URL` | Postgres | ✅ via Replit DB |

---

## 19. The North Star (so any LLM advisor stays aligned)

GAMZU is **not** a chatbot, **not** a productivity tool, **not** a journaling app. It is a **spiritual instrument** that uses three LLMs as distinct voices (Poet, Scientist, Historian) to help a user contemplate the 22 Hebrew letters as living archetypes, with the phone as a wisdom-aware extension of presence (Companion + Mind Space + Spiritual Buffer + Chesed gate). Every architectural decision should serve **emotional truth + reverent presence + intelligent privacy** — never raw productivity throughput.

When in doubt: choose the slower, more reflective, more loving option. That's what *Chesed* (loving-kindness) means in this codebase, and it's literally encoded as a 0–100 gate on every agentic action.
