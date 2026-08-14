# Implementation Plan: Verifolio MVP

## Overview

Full-stack implementation of Verifolio using Next.js 14 App Router, Prisma + Neon PostgreSQL, NextAuth.js v5 (Credentials), OpenAI GPT-4o (JSON mode), pdf-parse, GitHub REST API (unauthenticated), Tailwind CSS + shadcn/ui, and Vercel deployment. Tasks are ordered by build dependency; each task targets under 2 hours of focused work.

Tasks marked `*` are optional (testing / polish) and can be skipped for a faster MVP path.

---

## Tasks

### Phase 1 — Project Setup

- [x] 1. Initialize Next.js 14 project with TypeScript and App Router
  - Run `npx create-next-app@latest verifolio --typescript --app --tailwind --eslint`
  - Confirm `app/` directory structure, `tsconfig.json`, and `tailwind.config.ts` exist
  - _Requirements: all (foundation for every requirement)_

- [-] 2. Install and configure shadcn/ui component library
  - Run `npx shadcn-ui@latest init`; select default style, Tailwind CSS, TypeScript
  - Add baseline components: `button`, `card`, `input`, `label`, `badge`, `textarea`, `alert`, `separator`
  - _Requirements: all UI requirements (2–7)_

- [ ] 3. Configure environment variables and project structure
  - Create `.env.local` with placeholders: `DATABASE_URL`, `NEXTAUTH_SECRET`, `OPENAI_API_KEY`, `NEXTAUTH_URL`
  - Add `.env.local` to `.gitignore`
  - Create directory skeleton: `lib/ai/`, `lib/github/`, `lib/db.ts`, `components/cv/`, `components/repos/`, `components/skills/`, `components/evidence/`, `components/jobs/`, `components/public/`
  - _Requirements: all (foundational structure)_

- [ ] 4. Initialize Prisma and connect to Neon PostgreSQL
  - Run `npm install prisma @prisma/client` and `npx prisma init`
  - Set `DATABASE_URL` in `.env.local` to Neon connection string (pooled)
  - Create `lib/db.ts` as the Prisma client singleton (use global guard to avoid hot-reload issues in dev)
  - _Requirements: 1, 2, 3, 4, 5, 6, 7_

- [ ] 5. Configure Vitest and testing infrastructure
  - Run `npm install -D vitest @vitest/coverage-v8 fast-check @testing-library/react @testing-library/jest-dom jsdom`
  - Create `vitest.config.ts` in project root with `environment: 'jsdom'`, path aliases mirroring `tsconfig.json`
  - Add `tests/unit/` and `tests/integration/` directory structure
  - Add `"test": "vitest --run"` and `"test:watch": "vitest"` scripts to `package.json`
  - _Requirements: all (testing infrastructure)_

---

### Phase 2 — Database Schema

- [ ] 6. Define complete Prisma schema with all models and enums
  - Write `prisma/schema.prisma` with: `User`, `Profile`, `Skill`, `EvidenceItem`, `Repository`, `JobMatch`, `MatchResult` models exactly as specified in design.md
  - Define enums: `SkillOrigin`, `SkillState`, `SkillStrength`, `RepoStatus`, `MatchCategory`
  - Set `@@unique([profileId, name])` on `Skill` and `@@unique([userId, ownerRepo])` on `Repository`
  - _Requirements: 1, 2, 3, 4, 5, 6, 7_

- [ ] 7. Run initial Prisma migration and generate client
  - Run `npx prisma migrate dev --name init` against the Neon dev database
  - Verify generated `@prisma/client` types compile without errors
  - Confirm all tables and enums are created in Neon via `npx prisma studio`
  - _Requirements: 1, 2, 3, 4, 5, 6, 7_

---

### Phase 3 — Authentication

- [ ] 8. Install NextAuth.js v5 and configure Credentials provider
  - Run `npm install next-auth@beta bcryptjs` and `npm install -D @types/bcryptjs`
  - Create `app/api/auth/[...nextauth]/route.ts` with Credentials provider
  - Implement `authorize` callback: look up `User` by email, compare password with `bcrypt.compare`, return user object or `null`
  - Configure JWT session strategy and `NEXTAUTH_SECRET` in `.env.local`
  - _Requirements: 1.3, 1.4_

- [ ] 9. Implement user registration API route
  - Create `app/api/auth/register/route.ts` (POST handler)
  - Validate email format and password length ≥ 8 characters; return 400 on violation
  - Check for duplicate email; return 409 if already registered
  - Hash password with `bcrypt.hash(password, 10)`
  - Generate `username` slug: `{normalized-display-name}-{4-char-random}` (e.g., `hadia-khan-a3f2`); use `crypto.randomBytes(2).toString('hex')` for the suffix
  - Create `User` + empty `Profile` in a Prisma transaction; return 201
  - _Requirements: 1.1, 1.2, 7.1_

- [ ] 10. Build register and login UI pages
  - Create `app/(auth)/register/page.tsx` — form with email, password fields; calls `/api/auth/register`, then `signIn` on success
  - Create `app/(auth)/login/page.tsx` — form that calls `signIn("credentials", ...)` from NextAuth; displays error on failure
  - Use shadcn/ui `Card`, `Input`, `Label`, `Button` components throughout
  - _Requirements: 1.1, 1.2, 1.3, 1.4_

- [ ] 11. Implement middleware for protected routes and logout
  - Create `middleware.ts` at project root: protect all `/dashboard/*` and `/api/*` paths (except `/api/auth/*` and `/p/*`); redirect unauthenticated requests to `/login`
  - Add logout button to dashboard layout that calls `signOut()` and redirects to `/login`
  - _Requirements: 1.5_

---

### Phase 4 — CV Upload Pipeline

- [ ] 12. Create CV upload route handler with file validation
  - Create `app/api/cv/upload/route.ts` (POST, multipart form)
  - Install `npm install pdf-parse` and `npm install -D @types/pdf-parse`
  - Validate MIME type: accept only `application/pdf` and `text/plain`; return 400 with accepted-types message for anything else
  - Enforce 10MB file size limit; return 400 if exceeded
  - Extract text: call `pdf-parse` for PDF, `buffer.toString('utf-8')` for TXT; return 422 on corrupt PDF
  - _Requirements: 2.1, 2.2_

- [ ] 13. Implement CV Extractor (`lib/ai/cv-extractor.ts`)
  - Install `npm install openai`
  - Write `extractProfile(cvText: string): Promise<ExtractedProfile>` function
  - Call GPT-4o with `response_format: { type: "json_object" }` and the system prompt from design.md specifying the exact `ExtractedProfile` schema
  - Truncate `cvText` to 15,000 tokens before sending
  - Post-process: validate all required top-level fields (`name`, `education`, `workExperience`, `declaredSkills`, `extractionWarnings`) exist; default missing fields to empty values; retry once with stricter prompt on malformed JSON; return 500 on second failure
  - _Requirements: 2.3, 2.4_

- [ ] 14. Persist extracted profile and declared skills to database
  - In the `/api/cv/upload` route handler, after extraction: upsert `Profile` (`displayName`, `education`, `workExperience`, `cvUploadedAt`) for the authenticated user
  - Delete all existing `Skill` records with `origin=Declared` for the profile (CV re-upload replaces profile per Req 2.6)
  - Create new `Skill` records for each entry in `declaredSkills` with `origin=Declared`, `state=Confirmed`
  - Return the updated profile JSON including extraction warnings; surface warnings in the UI
  - _Requirements: 2.3, 2.4, 2.6_

- [ ] 15. Build CVUploadCard component and wire to dashboard
  - Create `components/cv/CVUploadCard.tsx` — drag-and-drop + click-to-select for PDF/TXT
  - Show upload progress spinner and extraction status messages
  - Display extraction warnings (from `extractionWarnings` array) as dismissible alerts
  - Integrate into `app/dashboard/page.tsx`
  - _Requirements: 2.1, 2.2, 2.3, 2.4_

---

### Phase 5 — GitHub Fetcher

- [ ] 16. Implement GitHub URL parser and repository info fetcher
  - Create `lib/github/fetcher.ts`
  - Write `parseGitHubUrl(url: string): { owner: string; repo: string } | null` — accept `https://github.com/{owner}/{repo}` format only; return `null` for invalid URLs
  - Fetch `GET /repos/{owner}/{repo}` to verify accessibility and retrieve `default_branch`; throw typed errors for 404 (private/not found) and 429 (rate limit)
  - _Requirements: 3.1, 3.2_

- [ ] 17. Implement file tree fetch, filtering, scoring, and content sampling
  - In `lib/github/fetcher.ts`, implement `fetchRepoSample(owner: string, repo: string): Promise<RepoSample>`
  - Fetch `GET /repos/{owner}/{repo}/git/trees/{sha}?recursive=1` to retrieve full file tree
  - Filter: exclude `node_modules`, `.git`, lock files, images, binaries; use extension allowlist: `.ts`, `.tsx`, `.js`, `.jsx`, `.py`, `.go`, `.java`, `.rs`, `.rb`, `.cs`, `.cpp`, `.c`, `.md`
  - Score and select ≤15 files: `README.md` first, then root-level source files, then files with names matching `index`, `main`, `app`, `server`, `routes`, `models`, `schema`
  - Fetch each selected file via `GET /repos/{owner}/{repo}/contents/{path}`; decode base64 content; truncate to 2KB; cap total payload at ~30KB
  - Return `RepoSample` object as defined in design.md
  - _Requirements: 3.1, 3.3_

---

### Phase 6 — AI Repository Analyzer

- [ ] 18. Implement AI Analyzer (`lib/ai/repo-analyzer.ts`)
  - Write `analyzeRepo(sample: RepoSample): Promise<AnalysisResult>` function
  - Call GPT-4o with `response_format: { type: "json_object" }` using the system prompt from design.md (skills, strength, dimensions, evidenceItems schema)
  - Post-process: validate JSON schema; downgrade `Strong` → `Partial` if `evidenceItems.length < 3`; downgrade `Partial` → `Inferred` if `evidenceItems.length < 1`; retry once on malformed JSON
  - Return validated `AnalysisResult`
  - _Requirements: 3.3, 3.5, 5.3_

- [ ] 19. Create `/api/repos/analyze` route handler with repo count enforcement
  - Create `app/api/repos/analyze/route.ts` (POST)
  - Validate GitHub URL format; return 400 for invalid URLs
  - Enforce max 3 repositories per user: count existing `Repository` records; return 400 if already at 3
  - Upsert `Repository` record (set `analysisStatus=Analyzing`); call `fetchRepoSample`, then `analyzeRepo`
  - On success: set `analysisStatus=Done`, `analyzedAt`; on failure: set `analysisStatus=Error`, `errorMessage`
  - _Requirements: 3.1, 3.2, 3.3, 3.4_

---

### Phase 7 — Skill Classification & Weak Claim Detection

- [ ] 20. Implement skill classification and DB persistence after analysis
  - After `analyzeRepo` returns, iterate over `skills` in `AnalysisResult`
  - For each skill: check if `name` (case-insensitive) exists in the candidate's `declaredSkills` set
    - If match found → add `EvidenceItem` records to the existing `Declared` skill; update `strength`
    - If no match → create new `Skill` with `origin=Discovered`, `state=Pending`, `strength` from AI output; create `EvidenceItem` records
  - Enforce Property 10: do NOT persist a `Discovered` skill with zero evidence items
  - _Requirements: 4.1, 4.5_

- [ ] 21. Implement Weak Claim detection logic
  - After all skill + evidence persistence is complete for an analysis run, query all `Skill` records with `origin=Declared` for the profile
  - For each declared skill: count associated `EvidenceItem` records; if count === 0, set `state=WeakClaim`
  - For any declared skill with ≥1 evidence item that was previously `WeakClaim`, reset `state=Confirmed`
  - _Requirements: 4.4_

- [ ] 22. Implement accept/dismiss Server Actions for Discovered skills
  - Create server actions in `app/dashboard/skills/actions.ts`
  - `acceptSkill(skillId: string)`: set `state=Confirmed` for a `Pending` skill; `revalidatePath('/dashboard/skills')`
  - `dismissSkill(skillId: string)`: set `state=Dismissed`; `revalidatePath('/dashboard/skills')`
  - `toggleSkillVisibility(skillId: string)`: flip `isPublic`; `revalidatePath('/dashboard/skills')` and `revalidatePath('/p/[username]')`
  - Validate that the skill belongs to the authenticated user before mutation
  - _Requirements: 4.2, 4.3, 7.5_

---

### Phase 8 — Evidence Reports

- [ ] 23. Implement evidence report query and page
  - Create `app/dashboard/evidence/[skillId]/page.tsx` as a Server Component
  - Query `Skill` by `skillId` (verify ownership), include `evidenceItems` joined with `Repository`
  - Pass serialized data to `EvidenceReport` component
  - _Requirements: 5.1, 5.2_

- [ ] 24. Build EvidenceReport component
  - Create `components/evidence/EvidenceReport.tsx`
  - Display: skill name, strength badge (colour-coded: Strong=green, Partial=yellow, Inferred=grey), list of evidence items each showing `repoName`, `filePath`, and `description`
  - Handle empty state (no evidence items)
  - _Requirements: 5.1, 5.2, 5.3_

---

### Phase 9 — Job Description Matching

- [ ] 25. Implement Job Matcher (`lib/ai/job-matcher.ts`)
  - Write `matchJob(input: JobMatchInput): Promise<JobMatchOutput>` function
  - Call GPT-4o with `response_format: { type: "json_object" }` using the system prompt from design.md
  - Post-process: validate JSON schema; verify every skill in `extractedRequiredSkills` appears in `matchResults`; add `Gap` entries for any omitted; verify count invariant (`strongMatches + partialMatches + gaps === extractedRequiredSkills.length`); retry once on malformed JSON
  - _Requirements: 6.2, 6.3_

- [ ] 26. Create `/api/jobs/match` route handler and persist results
  - Create `app/api/jobs/match/route.ts` (POST)
  - Validate JD text: non-empty, ≥10 characters (return 400); ≤50,000 characters (return 400)
  - Verify candidate has at least one `Confirmed` skill (return 422 if none)
  - Load `CandidateSkillSummary[]` from DB (confirmed skills + evidence count); call `matchJob`
  - Persist `JobMatch` + `MatchResult[]` records; return match output JSON
  - _Requirements: 6.1, 6.2, 6.3, 6.6_

- [ ] 27. Build JobMatchResults component and jobs page
  - Create `components/jobs/JobMatchResults.tsx` — three-column layout: Strong Match / Partial Match / Gap with summary counts header
  - Strong Match and Partial Match items: clickable, navigate to `/dashboard/evidence/[skillId]`
  - Gap items: display skill name + `gapGuidance` text
  - Create `app/dashboard/jobs/page.tsx` — JD textarea + submit button + results
  - _Requirements: 6.1, 6.3, 6.4, 6.5, 6.6_

---

### Phase 10 — Public Profile

- [ ] 28. Build `/p/[username]` Server Component
  - Create `app/p/[username]/page.tsx` as a Server Component (no auth check)
  - Query `User` by `username`; 404 if not found
  - Fetch `Profile` with `skills` filtered to `state=Confirmed AND isPublic=true`, include `evidenceItems`
  - Pass to `PublicProfileView` component; use `cache: 'no-store'` fetch option to ensure freshness
  - Ensure response does NOT include `email`, `passwordHash`, `education`, or `workExperience`
  - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.6_

- [ ] 29. Build PublicProfileView component
  - Create `components/public/PublicProfileView.tsx`
  - Display: candidate display name, confirmed public skills grouped by `SkillStrength` (Strong → Partial → Inferred), evidence summary for each skill (count of evidence items + repo names)
  - Include copy-to-clipboard button for the profile URL
  - _Requirements: 7.2, 7.3, 7.4_

---

### Phase 11 — Dashboard UI

- [ ] 30. Build dashboard layout and profile overview page
  - Create `app/dashboard/layout.tsx` with sidebar navigation links to: Profile, Repositories, Skills, Jobs; include logout button
  - Create `app/dashboard/page.tsx` — profile overview showing `displayName`, CV upload CTA (if no CV yet), link to public profile URL
  - _Requirements: 1.3, 2.5, 7.1_

- [ ] 31. Build profile edit page with Server Actions
  - Create `app/dashboard/profile/page.tsx` — Server Component that loads current profile data
  - Add editable fields for `displayName`; display read-only `education` and `workExperience` extracted from CV
  - Implement `updateProfile` Server Action in `app/dashboard/profile/actions.ts`; call `revalidatePath`
  - _Requirements: 2.5_

- [ ] 32. Build repositories page with RepoInputList component
  - Create `components/repos/RepoInputList.tsx` — up to 3 URL inputs (enforced by disabling "Add repo" when 3 exist); per-repo status badge using `RepoStatus` enum values
  - Create `app/dashboard/repos/page.tsx` — shows existing repos with status, URL input, "Analyze" button that calls `/api/repos/analyze`
  - Show inline errors for invalid URLs, private repos, rate limit hits
  - _Requirements: 3.1, 3.2, 3.4_

- [ ] 33. Build skills management page with SkillCard and WeakClaimBanner
  - Create `components/skills/SkillCard.tsx` — displays skill name, strength badge, origin tag (Declared / Discovered); show Accept/Dismiss buttons for `state=Pending` skills; show visibility toggle for `state=Confirmed` skills
  - Create `components/skills/WeakClaimBanner.tsx` — warning list of skills with `state=WeakClaim`
  - Create `app/dashboard/skills/page.tsx` — Server Component grouping skills by state: Confirmed, Pending (Discovered), WeakClaims, Dismissed
  - _Requirements: 4.1, 4.2, 4.3, 4.4, 7.5_

---

### Phase 12 — Property-Based Tests (Optional)

- [ ]* 34. Write property test P1: Password minimum-length validation
  - File: `tests/unit/auth/password-validation.property.test.ts`
  - **Property 1: Password Minimum-Length Validation**
  - Arbitrary: `fc.string()` of varying length; assert validation function rejects length < 8, accepts length ≥ 8
  - **Validates: Requirements 1.1**

- [ ]* 35. Write property test P2: Authentication round-trip correctness
  - File: `tests/unit/auth/auth-roundtrip.property.test.ts`
  - **Property 2: Authentication Round-Trip Correctness**
  - Arbitrary: `fc.emailAddress()`, `fc.string({ minLength: 8 })`; mock Prisma; assert valid credentials → session, wrong password → no session
  - **Validates: Requirements 1.3, 1.4**

- [ ]* 36. Write property test P3: CV file-type validation
  - File: `tests/unit/cv/file-type-validation.property.test.ts`
  - **Property 3: CV File-Type Validation**
  - Arbitrary: `fc.string()` as MIME type; assert `application/pdf` and `text/plain` accepted, all others rejected
  - **Validates: Requirements 2.1, 2.2**

- [ ]* 37. Write property test P4: CV extractor output schema completeness
  - File: `tests/unit/ai/cv-extractor-schema.property.test.ts`
  - **Property 4: CV Extractor Output Schema Completeness**
  - Arbitrary: `fc.string({ minLength: 10 })` as CV text; mock OpenAI client; assert returned object always has all five top-level fields
  - **Validates: Requirements 2.3, 2.4**

- [ ]* 38. Write property test P5: CV re-upload replaces profile
  - File: `tests/unit/cv/cv-replace.property.test.ts`
  - **Property 5: CV Re-Upload Replaces Profile**
  - Arbitrary: two `fc.record(...)` `ExtractedProfile` objects; mock Prisma; assert second upload's data completely replaces first
  - **Validates: Requirements 2.6**

- [ ]* 39. Write property test P6: Repository count bounds
  - File: `tests/unit/repos/repo-count.property.test.ts`
  - **Property 6: Repository Count Bounds**
  - Arbitrary: `fc.integer({ min: 0, max: 10 })` as existing repo count; assert add succeeds iff count < 3, fails with 400 if count === 3
  - **Validates: Requirements 3.1**

- [ ]* 40. Write property test P7: AI Analyzer output schema invariant
  - File: `tests/unit/ai/analyzer-schema.property.test.ts`
  - **Property 7: AI Analyzer Output Schema Invariant**
  - Arbitrary: arbitrary `AnalysisResult` JSON; mock OpenAI; assert every skill has non-empty `name`, valid `strength`, non-empty `dimensions`, and valid `evidenceItems`
  - **Validates: Requirements 3.3, 3.5**

- [ ]* 41. Write property test P8: Discovered skill classification
  - File: `tests/unit/skills/skill-classification.property.test.ts`
  - **Property 8: Discovered Skill Classification**
  - Arbitrary: `fc.array(fc.string())` for AI skills, `fc.array(fc.string())` for declared skills; assert non-matching skills → Discovered/Pending, matching skills → update existing Declared
  - **Validates: Requirements 4.1**

- [ ]* 42. Write property test P9: Weak claim detection
  - File: `tests/unit/skills/weak-claims.property.test.ts`
  - **Property 9: Weak Claim Detection**
  - Arbitrary: profiles with `fc.integer({ min: 0, max: 10 })` evidence item counts per declared skill; assert zero-evidence → WeakClaim, ≥1 evidence → not WeakClaim
  - **Validates: Requirements 4.4**

- [ ]* 43. Write property test P10: Evidence item cardinality for discovered skills
  - File: `tests/unit/skills/evidence-cardinality.property.test.ts`
  - **Property 10: Evidence Item Cardinality for Discovered Skills**
  - Arbitrary: `AnalysisResult` with Discovered skills; assert skills with zero evidence items are never persisted
  - **Validates: Requirements 4.5**

- [ ]* 44. Write property test P11: Evidence report structural completeness
  - File: `tests/unit/evidence/report-structure.property.test.ts`
  - **Property 11: Evidence Report Structural Completeness**
  - Arbitrary: `Skill` + `fc.array(EvidenceItem)` with length ≥ 1; assert report always includes `skillName`, `strength`, and complete evidence item fields
  - **Validates: Requirements 5.1, 5.2**

- [ ]* 45. Write property test P12: Skill strength assignment rules
  - File: `tests/unit/skills/skill-strength.property.test.ts`
  - **Property 12: Skill Strength Assignment Rules**
  - Arbitrary: `fc.integer({ min: 0, max: 20 })` as evidence count; assert count ≥ 3 → Strong, 1–2 → Partial, 0 → Inferred
  - **Validates: Requirements 5.3**

- [ ]* 46. Write property test P13: Evidence report monotonic growth
  - File: `tests/unit/evidence/monotonic-growth.property.test.ts`
  - **Property 13: Evidence Report Is Monotonically Growing**
  - Arbitrary: before/after `fc.array(EvidenceItem)` sets; assert after-set is superset of before-set
  - **Validates: Requirements 5.4**

- [ ]* 47. Write property test P14: Job match categorization completeness
  - File: `tests/unit/ai/job-match-completeness.property.test.ts`
  - **Property 14: Job Match Categorization Completeness**
  - Arbitrary: `fc.array(fc.string())` for extracted required skills; mock GPT-4o response; assert every required skill appears in `matchResults`
  - **Validates: Requirements 6.2**

- [ ]* 48. Write property test P15: Match result count invariant
  - File: `tests/unit/ai/match-count-invariant.property.test.ts`
  - **Property 15: Match Result Count Invariant**
  - Arbitrary: arbitrary `MatchResult[]`; assert `|strongMatches| + |partialMatches| + |gaps| === |extractedRequiredSkills|`
  - **Validates: Requirements 6.3**

- [ ]* 49. Write property test P16: Job match does not mutate analysis state
  - File: `tests/unit/jobs/match-idempotency.property.test.ts`
  - **Property 16: Job Match Does Not Mutate Analysis State**
  - Arbitrary: profiles + multiple `fc.string()` JD texts; mock Prisma; assert `Repository`, `Skill`, `EvidenceItem` records unchanged after each match; only `JobMatch`/`MatchResult` records created
  - **Validates: Requirements 6.6**

- [ ]* 50. Write property test P17: Public profile URL uniqueness
  - File: `tests/unit/public/url-uniqueness.property.test.ts`
  - **Property 17: Public Profile URL Uniqueness**
  - Arbitrary: `fc.array(fc.string(), { minLength: 2 })` as display names; assert slug generation produces unique usernames across all inputs
  - **Validates: Requirements 7.1**

- [ ]* 51. Write property test P18: Public profile data contract
  - File: `tests/unit/public/profile-data-contract.property.test.ts`
  - **Property 18: Public Profile Data Contract**
  - Arbitrary: full profile objects generated with `fc.record(...)`; assert public profile response contains confirmed public skills but never `email`, `passwordHash`, `education`, `workExperience`
  - **Validates: Requirements 7.2, 7.3, 7.4**

- [ ]* 52. Write property test P19: Public profile reflects latest state (integration)
  - File: `tests/integration/public/profile-consistency.test.ts`
  - **Property 19: Public Profile Reflects Latest Profile State**
  - DB-level integration test using `DATABASE_URL_TEST`; update skills/evidence in DB; assert `/p/{username}` response on next load is consistent with DB state — no stale data
  - **Validates: Requirements 7.6**

---

### Phase 13 — Integration Tests (Optional)

- [ ]* 53. Write integration test: CV upload flow
  - File: `tests/integration/cv/upload-flow.test.ts`
  - Mock OpenAI and pdf-parse; use real Prisma against `DATABASE_URL_TEST`
  - Assert: valid PDF upload → `Profile` created in DB with correct declared skills; invalid MIME → 400; re-upload → profile replaced
  - _Requirements: 2.1, 2.3, 2.6_

- [ ]* 54. Write integration test: Repo analyze flow
  - File: `tests/integration/repos/analyze-flow.test.ts`
  - Mock GitHub HTTP calls and OpenAI; use real Prisma against test DB
  - Assert: analyze → `Repository` status=Done, `Skill` and `EvidenceItem` records created; 4th repo → 400
  - _Requirements: 3.1, 3.3, 4.1, 4.4_

- [ ]* 55. Write integration test: JD match flow
  - File: `tests/integration/jobs/match-flow.test.ts`
  - Mock OpenAI; use real Prisma against test DB
  - Assert: match → `JobMatch` + `MatchResult[]` created; running match twice does not modify `Skill` or `EvidenceItem` records
  - _Requirements: 6.2, 6.6_

- [ ]* 56. Write integration test: Public profile page
  - File: `tests/integration/public/profile-page.test.ts`
  - Use real Prisma against test DB; render `/p/{username}` Server Component
  - Assert: returns confirmed public skills; does NOT return `email`, `passwordHash`, `education`, `workExperience`
  - _Requirements: 7.2, 7.3, 7.4_

---

### Phase 14 — Deployment

- [ ] 57. Configure Vercel project and environment variables
  - Create Vercel project linked to GitHub repository
  - Set production environment variables in Vercel dashboard: `DATABASE_URL` (Neon pooled connection string), `NEXTAUTH_SECRET`, `OPENAI_API_KEY`, `NEXTAUTH_URL` (production URL)
  - _Requirements: all (deployment)_

- [ ] 58. Run production database migration and deploy
  - Add `postinstall` script in `package.json`: `"postinstall": "prisma generate"`
  - Run `npx prisma migrate deploy` against the production Neon database (via Vercel build command or manually)
  - Push to main branch to trigger Vercel deployment; verify build succeeds
  - _Requirements: all (deployment)_

- [ ] 59. Smoke test deployed application end-to-end
  - Register a new account; verify public profile URL is created
  - Upload a PDF CV; verify profile is populated with extracted data
  - Add a public GitHub repo URL; trigger analysis; verify skills appear
  - Accept a Discovered skill; verify it shows on public profile
  - Paste a job description; verify match results appear with all three categories
  - _Requirements: 1, 2, 3, 4, 5, 6, 7_

---

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP delivery
- Each required task references specific requirements for traceability
- Phases 1–11 represent the core required build path (~35 required tasks, targeting 2–3 days solo)
- Property tests (P1–P19) and integration tests (Phase 13) add comprehensive correctness coverage but do not block the running application
- The Vitest single-run command is `npx vitest --run`; use `npx vitest` in your terminal for watch mode during development
- All AI calls use `response_format: { type: "json_object" }` and retry once on malformed JSON before returning an error response
- Weak Claim detection runs after every repository analysis completes — check all Declared skills against their current evidence count
- The username slug format `{normalized-name}-{4-char-random}` is generated once at registration and never changes
- Public profile only shows skills where `state=Confirmed AND isPublic=true`

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1", "5"] },
    { "id": 1, "tasks": ["2", "3", "4"] },
    { "id": 2, "tasks": ["6"] },
    { "id": 3, "tasks": ["7"] },
    { "id": 4, "tasks": ["8", "9", "16"] },
    { "id": 5, "tasks": ["10", "11", "13", "17"] },
    { "id": 6, "tasks": ["12", "18"] },
    { "id": 7, "tasks": ["14", "15", "19"] },
    { "id": 8, "tasks": ["20"] },
    { "id": 9, "tasks": ["21", "22", "23", "25"] },
    { "id": 10, "tasks": ["24", "26", "28"] },
    { "id": 11, "tasks": ["27", "29", "30"] },
    { "id": 12, "tasks": ["31", "32", "33"] },
    { "id": 13, "tasks": ["34", "35", "36", "37", "38", "39", "40", "41", "42", "43", "44", "45", "46", "47", "48", "49", "50", "51"] },
    { "id": 14, "tasks": ["52", "53", "54", "55", "56"] },
    { "id": 15, "tasks": ["57"] },
    { "id": 16, "tasks": ["58"] },
    { "id": 17, "tasks": ["59"] }
  ]
}
```
