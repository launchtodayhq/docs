# Launch Boilerplate Quality Assessment (Dec 2025)

Goal recap (from you): make this the best “world class” Expo + tRPC backend boilerplate for startups—**modular building blocks**, **professional/security-minded**, and **high trust + educational docs**.

This document is a candid assessment of the current state vs that goal, based on reading `Launch/docs/` and representative code paths in `apps/mobile` + `apps/api`.

---

## Current level (high-level scorecard)

These are directional, not absolute—meant to help prioritize.

| Area                                                                              |                                Current level | Why                                                                                                                                                                                 |
| --------------------------------------------------------------------------------- | -------------------------------------------: | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Mobile architecture & feature separation                                          | **Good foundations, inconsistent execution** | You have `features/chat` and `lib/*` modules, but the rest of the app is not consistently structured as “removable blocks”.                                                         |
| API architecture & separation                                                     |                         **Good foundations** | Koa routing + tRPC routers + `services/` is a solid baseline.                                                                                                                       |
| End-to-end type safety (tRPC promise)                                             |                                  **Not yet** | Web imports `AppRouter` types, mobile uses `any` and casts `trpc as any` in multiple places.                                                                                        |
| Security posture                                                                  |                  **Mixed / needs hardening** | Strong pieces exist (SecureStore, input validation), but there are critical trust breakers (hardcoded OAuth secrets in config; overly-permissive dev CORS behaviors; auth logging). |
| Docs accuracy / trust                                                             |                                  **Big gap** | Multiple docs are out of sync with actual code (stack mismatch, wrong file paths, features claimed that aren’t currently wired).                                                    |
| “Swap/remove features like Lego”                                                  |                                  **Not yet** | Some modules are swappable (payments provider selection), but app composition isn’t designed as a feature registry with clean contracts and deletion guides.                        |
| Production readiness cues (logging, env validation, rate limiting, observability) |                                    **Early** | Some logging exists, but not “production-grade by default” yet. Feature flag mentions exist without enforcement (e.g., rate limiting).                                              |

---

## What’s already strong (keep and build on)

### Mobile: real modular intent exists

- **AI + Chat is reasonably layered**:
  - UI/state orchestration: `apps/mobile/features/chat/context/ChatContext.tsx`
  - Streaming generation: `apps/mobile/features/chat/hooks/useChat.ts`
  - Persistence/history: `apps/mobile/features/chat/hooks/useChatHistory.ts`, `useChatPersistence.ts`
  - Provider abstraction: `apps/mobile/lib/ai/providers/*`
  - Streaming transport: `apps/mobile/lib/api/streaming.ts` (SSE client)

This is very close to the kind of “teachable, modular architecture” you want.

### API: routing split is clean

- `apps/api/src/app.ts` clearly dispatches:

  - Better Auth endpoints under `/api/auth/*`
  - tRPC under `/trpc`
  - webhooks under `/webhooks/*` (raw body preserved)
  - AI streaming under `/api/ai/stream`

- `apps/api/src/lib/trpc.ts` creates a consistent auth context and `protectedProcedure`.

### Payments: the “swap provider” concept is already present

The “unified payment provider” pattern is the right direction:

- `apps/mobile/lib/payments/provider.tsx` chooses between Stripe / RevenueCat / Superwall.

This can become a flagship example of “feature building blocks” with some refactoring (see gaps below).

---

## Biggest trust breakers (must fix for “world class”)

These are the items that undermine confidence the fastest—especially if you’re positioning this as secure and production-grade.

### 1) Docs and code disagree in important places

Examples:

- `Launch/docs/backend/index.mdx` describes **Hono + Drizzle** and a folder `apps/server/`, but the actual backend is **Koa + Prisma** in `apps/api/`.
- `Launch/docs/mobile/file-structure.mdx` describes a component folder structure (`components/auth`, `components/ui`, etc.) and a `launch.config.ts` copy system that **does not exist** in the repo structure currently shown.
- `Launch/docs/ai-features/overview.mdx` claims a fully-featured chat screen at `/ai-chat` with prompt suggestions/history sheet, but `apps/mobile/app/ai-chat.tsx` is currently a **keyboard/composer layout test** that does not call the AI API.
- `Launch/docs/README.md` is still the **generic Mintlify starter kit readme**.

Impact:

- This breaks the “core value prop is what you learn + trust” immediately.
- Engineers will waste time chasing paths that don’t exist, and then assume the rest is similarly unreliable.

### 2) Hardcoded OAuth secrets / IDs in server config

In `apps/api/src/config/app.ts`, Google config has fallbacks that include what look like real IDs/secrets.

Impact:

- Even if those are “demo”, it trains bad habits, looks unprofessional, and can create real security risk.
- It conflicts with your docs’ own security checklist (“No secrets in code”).

### 3) Mobile tRPC is not actually type-safe

The web app imports `AppRouter` types (`apps/web/lib/trpc.ts`), but mobile does:

- `apps/mobile/lib/trpc/client.ts`: `type AppRouter = any`
- `apps/mobile/features/chat/hooks/useChatHistory.ts` and `useChatPersistence.ts`: `const trpcAny = trpc as any`

Impact:

- The “tRPC promise” is a core differentiator of this stack; not having it on mobile is a big quality gap.
- It reduces teachability: “how to ship safely at speed” is largely about types preventing regressions.

---

## Architectural gaps vs “building blocks”

### 1) No explicit feature registry / composition layer

Right now, features are “present in folders”, but the app isn’t composed from a single place where features are registered/enabled/disabled with predictable contracts.

Example:

- `apps/mobile/app/_layout.tsx` directly wires `SessionProvider`, `PaymentProvider`, etc.
- Removing a feature means hunting for providers, routes, UI entry points, and docs across multiple locations.

What “world class” typically has:

- A `features/registry.ts` (or similar) that is the single place you can:
  - enable/disable features
  - see required env vars
  - see providers that must be mounted
  - see routes/screens the feature contributes
  - see permissions it needs (push notifications, camera roll, etc.)

### 2) Mixed transport story for AI chat (SSE REST + tRPC + DB)

Mobile chat generation uses:

- **SSE REST** endpoint: `POST /api/ai/stream` (mobile uses `lib/api/streaming.ts`)
- Persistence uses **tRPC** endpoints: `chat.*` procedures (mobile uses `useChatHistory` / `useChatPersistence`)

This is not “wrong”, but as a boilerplate, it needs to be extremely clearly documented as a deliberate tradeoff:

- Why SSE is chosen (better streaming support)
- How auth is attached (cookies from Better Auth)
- How to swap SSE out for “tRPC-only” or “WebSocket” in a clean way

Right now, that modular “swap story” exists partly in code but isn’t made explicit as a building block contract.

### 3) Payments provider selection uses conditional hooks

In `apps/mobile/lib/payments/provider.tsx`, `usePayment()` conditionally calls hooks and disables the React hook rules.

Impact:

- This is a “looks fine until it bites you” pattern.
- It makes the code less teachable, because it normalizes breaking fundamental React rules.

What “world class” would do instead:

- Always render a single `PaymentContext.Provider` whose value is provided by the selected provider implementation.
- Providers implement a shared interface and set the context value.
- `usePayment()` simply reads from context (no conditional hooks).

---

## Security / production hardening gaps

### 1) Rate limiting is referenced but not enforced

`apps/api/src/config/app.ts` has `features.enableRateLimiting`, but I didn’t see an actual rate limiting middleware wired into `apps/api/src/app.ts`.

For a “boilerplate for startups”, this should be:

- enabled and safe-by-default in production
- documented with a clear extension point
- at least covering:
  - auth endpoints
  - AI streaming endpoint
  - any public webhooks (with signature verification)

### 2) CORS behavior is permissive in dev and ambiguous for RN

`apps/api/src/middleware/cors.ts` returns `"*"` when there is no Origin header (common for some RN traffic), and also echoes origin for all origins in development.

This is understandable for DX, but “world class” needs:

- very explicit documentation of what to set in dev vs prod
- a clearer “approved origins” strategy
- no confusing wildcard-with-credentials situations

### 3) Auth logging contains risky patterns

`apps/api/src/routes/auth/auth.ts` includes:

- a debug log printing cookies for `/get-session`

Even if intended for development, boilerplates should model clean “dev logging” patterns that cannot accidentally leak secrets, and should be gated behind a strict debug flag.

### 4) Environment validation is not strict enough

Some places use `process.env.X || default`.

For a high-trust boilerplate, you want:

- a single env schema (zod) per app
- no hardcoded secrets
- predictable behavior when env is missing (fail-fast with actionable errors)

---

## Error handling & observability (state of the art)

This is one of the biggest “world class” differentiators. A boilerplate should teach:

- **How errors are represented** (consistent error shapes/codes)
- **How errors are handled** (user-facing UX, retries, offline)
- **How errors are surfaced** (Sentry/Datadog with correlation IDs and sane noise controls)
- **How to avoid leaking secrets/PII** in logs and error reports

### What you have today (good + gaps)

- **Good**:

  - API request logging has **sanitization** helpers (`apps/api/src/middleware/logging.ts`) that strip common sensitive fields and headers.
  - tRPC has a clean `protectedProcedure` pattern that throws `UNAUTHORIZED` (`apps/api/src/lib/trpc.ts`).

- **Gaps**:
  - API does not appear to have a single “top-level error middleware” that:
    - converts thrown errors into a consistent response shape
    - tags errors with a request/trace ID
    - prevents accidental leaking of internal stack traces or sensitive error messages
  - Logging is mostly `console.log` formatted output (`apps/api/src/utils/logger.ts`) rather than structured JSON logs that work well in Datadog.
  - Auth route code contains risky debug logging (e.g., printing cookies in `apps/api/src/routes/auth/auth.ts`), which is exactly the kind of thing that leaks secrets into log aggregators.
  - On mobile, the chat feature handles streaming errors with a generic “Something went wrong”, but there isn’t a clear app-wide standard for:
    - error boundaries
    - mapping error codes → user messages
    - capturing and correlating errors with backend logs

### What “world class” looks like (recommended targets)

#### 1) Consistent error taxonomy (API + mobile)

Define and document a small, stable set of error codes/categories, e.g.:

- `UNAUTHORIZED`, `FORBIDDEN`
- `VALIDATION_ERROR`
- `RATE_LIMITED`
- `UPSTREAM_TIMEOUT` (OpenAI/Anthropic slow)
- `UPSTREAM_ERROR` (provider 5xx)
- `NETWORK_ERROR` (client side)
- `INTERNAL_ERROR` (unexpected)

Then ensure:

- **API responses include**: `code`, `message` (safe), `requestId`/`traceId`
- **Client error handling** branches on `code` (not string matching)

tRPC already has a place to standardize this via its error formatting; Koa REST endpoints should follow the same contract.

#### 2) Correlation IDs everywhere (the “how do I debug this?” story)

Make it a first-class guarantee that:

- every API request has a `traceId`/`requestId`
- the API returns it in a response header (`X-Request-Id`) and/or JSON body on errors
- the mobile app attaches:
  - `X-Request-Id` (generated client-side for the request), or
  - accepts `X-Request-Id` from server and uses it in Sentry breadcrumbs

This is what makes Sentry ↔ Datadog ↔ backend logs line up.

You already generate a `traceId` in `apps/api/src/utils/tracing.ts` usage in logging middleware—what’s missing is making it part of the external contract and ensuring it’s used consistently across all handlers (tRPC + REST + webhooks).

#### 3) Structured logging by default (Datadog-friendly)

Plain `console.log` strings are human-readable, but don’t scale well for:

- dashboards
- searching by fields (userId, traceId, route, errorCode)
- alerting

Target:

- JSON logs with stable keys: `level`, `msg`, `traceId`, `route`, `status`, `durationMs`, `errorCode`, `userId?`
- never log secrets (cookies, tokens, idTokens), even in development, unless behind an explicit local-only debug switch

#### 4) “Safe by default” error reporting (Sentry/Datadog)

World-class boilerplates clearly show:

- **What gets captured** (and what never should)
- **How to reduce noise** (ignore known non-actionables like aborted requests)
- **How to attach context safely**:
  - device info (OK)
  - user id/email (maybe OK, depends on privacy posture)
  - never attach tokens/cookies

Suggested minimal integrations (docs should teach setup, not just mention it):

- Mobile:
  - Sentry React Native SDK (captures JS exceptions + native crashes)
  - optional Datadog RUM (navigation timing, app hangs, network tracing)
- API:
  - Datadog APM (request traces) or OpenTelemetry
  - Sentry Node (uncaught exceptions + performance traces)

#### 5) UX: error handling patterns that teach professionals

Examples of what the boilerplate should demonstrate consistently:

- **Global error boundary** for unexpected UI crashes (show a friendly recovery screen + “Restart” / “Report issue”).
- **Network-aware errors**:
  - offline vs server down vs auth expired
  - “retry” patterns with backoff where appropriate
- **Streaming cancellation as non-error**:
  - aborted streaming should be treated as expected (already partially done in `apps/mobile/lib/api/streaming.ts`)
- **Form validation errors** show inline messages (not generic toast)

### “Gap to close” checklist (high-priority)

- Add a consistent error response contract across:
  - tRPC errors
  - REST endpoints (`/api/ai/stream`, `/api/auth/*`)
- Add request/trace IDs to:
  - API responses
  - mobile error reports/breadcrumbs
- Replace debug logging that prints sensitive headers/cookies with safe, gated debug logs.
- Switch API logging to structured JSON, and document the expected log fields for Datadog/Sentry correlation.

---

## Code quality & consistency gaps

### 1) A lot of “docs mention patterns that code doesn’t implement”

The docs talk about:

- centralized copy system
- directory structure that isn’t reflected in `apps/mobile/components`
- backend stack mismatch

If the product’s value is “what you learn”, the code must be the source of truth—and docs must match it precisely.

### 2) Logging is not structured

`apps/api/src/utils/logger.ts` logs formatted strings and then pretty-prints body JSON; this is readable but not “production observability ready”.

“World class” boilerplate would:

- emit structured JSON logs (pino-style)
- include request IDs / trace IDs consistently
- avoid logging request bodies by default (only in debug, and always sanitized)

---

## How far are we from “world class”?

### Where you are right now

You have:

- a solid monorepo and modern stack
- real features implemented (payments, uploads, AI streaming, auth)
- a _directionally correct_ modular architecture in some areas (notably chat)

But:

- docs trust is not there yet (mismatches)
- type safety is incomplete (mobile)
- security hardening defaults are not consistent (hardcoded config, logging, rate limiting)
- “remove/replace features” is not yet a first-class product experience

---

## Onboarding funnel: intro → paywall → signup (should this be baked in?)

This “3-step funnel” is a pattern you see in many top-grossing consumer apps:

1. **Onboarding intro** (value prop + product education)
2. **Hard paywall** (before deep engagement)
3. **Sign up** (only after purchase intent is established)

### Recommendation

Yes, Launch should include this—but **not as the default always-on flow**.

Instead: ship it as a **first-class optional feature module** that can be enabled with one toggle and cleanly removed without hunting through the app.

Why:

- It’s “world class” because it teaches a proven growth/monetization pattern.
- But it has real integration dependencies (Stripe / RevenueCat / Superwall, entitlements, webhooks, restore purchases) that not every developer wants on day one.
- It also needs platform-aware guidance (App Store / Play policies, restore flows, messaging), which belongs in docs alongside the code.

### What “world class” would look like in this template

#### 1) A feature module, not a scattered set of screens

Make this its own building block (example shape):

- `features/funnel/`
  - `screens/` (Intro, Paywall, Signup)
  - `routing/` (guards + route groups)
  - `state/` (where the funnel stores completion state)
  - `contracts/` (what the rest of the app can rely on: “isPaywalled?”, “hasActiveEntitlement?”, “isAuthed?”)
  - `remove-guide.md` (exact steps to delete)

#### 2) Provider-agnostic paywall UI

The paywall screen should not “be Stripe” or “be RevenueCat”. It should depend on a small interface:

- “list available offerings”
- “purchase offering”
- “restore purchases”
- “get subscription/entitlement status”

You already have the start of this concept in `apps/mobile/lib/payments/*`. The key improvement is to make the paywall consume only the unified contract, so swapping payment providers doesn’t require rewriting the UI.

#### 3) Enable mode vs “requires setup”

To avoid a frustrating first-run experience, the feature should have clear modes:

- **Disabled (default)**: template uses the current onboarding/auth flow.
- **Enabled**: funnel is active and enforced.
- **Enabled but misconfigured**: the app should fail fast in development with a clear error (and docs point to setup), rather than silently skipping or behaving unpredictably.

This is a docs + code contract issue: “world class” templates never leave developers guessing why a paywall is blank.

### Docs needed to make this a flagship feature

If you include this funnel, the docs should include:

- Why this funnel exists and when to use it (startups vs enterprise/internal apps)
- A setup checklist per provider (Stripe vs RevenueCat vs Superwall)
- A minimal entitlement model (what “Pro” means) and where it’s enforced
- A “remove/replace the paywall” guide (how to swap it for a softer gate)
- App review notes (copy suggestions, restore purchases UX, avoiding misleading claims)

### What it would take (practical, staged)

#### Phase 0 (1–3 days): restore trust fast

- Make docs match the code **exactly** (even if shorter).
- Remove any hardcoded sensitive values from server config.
- Add “Current State” notes for AI chat screen vs keyboard test screen.

#### Phase 1 (1–2 weeks): make “building blocks” real

- Introduce a **feature registry** in mobile:
  - `features/*` each exports: `FeatureProvider?`, `routes?`, `requiredEnv?`, `docsLink`, `removeGuide`.
  - Root `_layout.tsx` becomes a thin composition layer.
- Standardize each feature as:
  - `features/<name>/{components,hooks,context,api,types,index.ts}`
- Provide a “How to remove feature X” doc for each top feature (AI, payments, uploads, auth).

#### Phase 2 (2–4 weeks): fulfill the tRPC promise on mobile

- Create a shared types package (or publish types from `apps/api`) so mobile can import `AppRouter` without `any`.
- Remove `trpc as any` usage and fix typing in chat persistence hooks.
- Add a “type-safe endpoint tutorial” doc page.

#### Phase 3 (1–2 months): production-grade defaults

- Rate limiting middleware + docs
- Environment validation schemas + docs
- Structured logging + docs
- Security checklist expanded into “what we ship by default vs what you must configure”
- Minimal CI checks (typecheck/lint/build) to reinforce quality and trust

---

## Specific “building blocks” design recommendation (mobile)

If you want the core experience to be “swap and ship”, the boilerplate should teach a consistent pattern:

### Proposed contract

Each feature exports an object like:

- `id`, `name`, `description`
- `providers`: React providers to mount (or a single provider component)
- `routes`: route segments or screens it contributes
- `requiredEnv`: env keys it needs
- `permissions`: camera, notifications, etc.
- `removeSteps`: bullet steps for removing it safely

Then the app root composes from that registry.

This creates:

- a single “table of contents” for features
- a single “toggle point” for enabling/disabling
- a natural structure for docs (docs mirror the registry)

---

## Key mismatches to fix in docs (short list)

- Replace the Mintlify starter `docs/README.md` with Launch-specific docs contributor guide.
- Fix backend docs to reflect: **Koa + Prisma** in `apps/api/`, not Hono/Drizzle/apps/server.
- Fix mobile file-structure docs to reflect the current directory layout (or refactor the code to match the docs).
- Update AI docs to match current `/ai-chat` reality (or wire the real chat screen back and note the keyboard composer work).

---

## Appendix: concrete file references observed

- Mobile chat layering:
  - `apps/mobile/features/chat/context/ChatContext.tsx`
  - `apps/mobile/features/chat/hooks/useChat.ts`
  - `apps/mobile/lib/api/streaming.ts`
  - `apps/mobile/lib/ai/providers/*`
  - `apps/mobile/features/chat/hooks/useChatHistory.ts` / `useChatPersistence.ts` (currently `any` tRPC usage)
- API routing:
  - `apps/api/src/app.ts`, `apps/api/src/router.ts`, `apps/api/src/lib/trpc.ts`
  - `apps/api/src/routes/ai-stream.ts` (SSE streaming)
  - `apps/api/src/routers/chat.ts` (DB persistence)
- Risk areas:
  - `apps/api/src/config/app.ts` (hardcoded OAuth fallbacks)
  - `apps/api/src/routes/auth/auth.ts` (cookie debug logging)
  - `apps/api/src/middleware/cors.ts` (dev permissiveness / wildcard behavior)
  - Mobile tRPC client: `apps/mobile/lib/trpc/client.ts` (`AppRouter = any`)
