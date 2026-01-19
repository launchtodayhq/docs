 # Documentation Quality Assessment & Upgrade Plan
 
 ## Purpose
 Capture the current documentation quality assessment and a concrete plan to
 upgrade Launch's docs to best-in-class for a React Native + Expo boilerplate.
 
 ---
 
 ## Assessment (Key Gaps)
 
 - **Trust breakers from doc/code mismatches**
   - Docs reference files or patterns that do not exist in the repo.
   - Claims about features and architecture do not align with current code.
 
 - **Broken or placeholder navigation**
   - `docs.json` links to pages that do not exist, causing dead ends.
 
 - **Critical “coming soon” pages**
   - Backend environment variables and deployment pages are stubs.
 
 - **Quickstart and setup inconsistency**
   - Requirements differ across pages (e.g., Node version).
   - Missing verification steps and clear success checkpoints.
 
 - **API Reference not tied to Launch**
   - Example OpenAPI content is unrelated to Launch.
 
 - **Internal tooling docs in public nav**
   - Mintlify authoring/AI tools docs show up as customer-facing content.
 
 - **Learning path lacks progressive guidance**
   - Docs are feature-indexed, not a guided onboarding funnel.
 
 ---
 
 ## Upgrade Plan
 
 ### Phase 0 — Restore Trust (1–3 days)
 - Fix doc/code mismatches (paths, claims, stack details).
 - Remove or hide missing pages from `docs.json`.
 - Replace or hide placeholder API reference content.
 - Normalize prerequisites (Node, pnpm, Expo CLI) across all pages.
 - Move internal authoring/tooling docs out of the main nav.
 
 ### Phase 1 — Improve New-User Journey (3–5 days)
 - Rewrite `quickstart.mdx` into a true “from zero” path with checkpoints.
 - Add a “Start here” guide that introduces Expo, monorepo structure, and the
   feature registry concept.
 - Make step order explicit: Clone → Install → DB → API env → API running →
   Mobile env → App running → First login.
 - Add a “What next?” section that maps to auth, payments, AI, uploads.
 
 ### Phase 2 — Feature Modules as Building Blocks (1–2 weeks)
 For each feature (Auth, Payments, AI, Uploads, Push):
 - Standardize doc structure:
   - Overview → Setup → How it works → Test checklist → Troubleshooting → Remove.
 - Tie docs to the feature registry:
   - required env vars, routes added, providers used, removal steps.
 - Ensure examples use real file paths and actual code behavior.
 
 ### Phase 3 — Production-Ready Documentation (2–4 weeks)
 - Full deployment docs (Railway/Docker for backend, EAS build/submit for mobile).
 - Expand app store submission docs (iOS + Android) with checklists.
 - Security/observability docs that reflect actual defaults in the repo.
 - Troubleshooting split into topic pages mapped to real failure points.
 
 ### Phase 4 — Docs QA & Maintenance (ongoing)
 - Add a docs QA checklist (paths valid, commands run, links verified).
 - Enforce docs updates for code changes.
 - Schedule regular quickstart verification (CI or release checklist).
 
 ---
 
 ## Next Optional Deliverables
 - A page-by-page change list mapped to specific docs.
 - A redesigned navigation tree aligned with the new learning path.
