# Sprint: AI Profile Draft Assistant

## Status

Implemented in the Node API and existing React profile edit flow.

Last context update: 2026-05-29.

Post-implementation status:

- AI profile drafting is wired through `POST /api/profile/ai-draft`.
- The OpenAI/LangChain integration lives in the API only.
- Generated values are previewed in `ProfileEditView` before they are applied.
- Applying a draft updates local form state only; saving still goes through the existing `PUT /api/profile` update flow.
- A persistence issue where applied draft values appeared in the UI but old values returned after refresh was fixed by saving an explicit applied-draft payload.
- The AI draft rate limiter now returns the same `{ success: false, message }` error shape as the rest of this feature contract.
- Image generation was discussed and intentionally deferred; it remains out of scope for this sprint.
- Follow-up public/member page redesign work was completed after the AI sprint validation. This is adjacent context, not part of the AI draft endpoint.
- Additional public-site polish was completed after the redesign: home search-first refinements, sector landing page alignment, shared `BodyVantage` inline token fixes, background grid removal, footer redesign, BV favicon replacement, large-screen spacing, tighter hero rhythm, intro treatment cleanup, decorative divider removal, and tighter mobile spacing so forms/content appear sooner.
- Follow-up form fixes were completed after the public/member polish: dark autofill backgrounds now hold after users fill/autofill fields, registration checkout submit now fires, and all shared `Button` components inside forms were audited for explicit `type` attributes.

## Goal

Help registered users create a professional BodyVantage profile faster by allowing them to describe their background in natural language, then using OpenAI through LangChain in the Node API to generate a structured profile draft that fills the existing profile form.

The AI output must be treated as a draft. The user must review, edit, and save the profile through the existing profile update flow.

## Product Decision

The AI integration belongs in the API, not the React client.

Reasons:

- Keep `OPENAI_API_KEY` server-side.
- Keep prompts, schema rules, validation, rate limiting, and audit logging centralized.
- Reuse existing Express auth, Joi validation, and profile save behavior.
- Prevent AI from writing admin-only or verification fields.
- Keep the frontend focused on collecting natural-language input, previewing the generated draft, and applying it to the form.

## References

- OpenAI Structured Outputs: https://developers.openai.com/api/docs/guides/structured-outputs
- LangChain JS models structured output: https://docs.langchain.com/oss/javascript/langchain/models
- LangChain OpenAI integration: https://docs.langchain.com/oss/javascript/integrations/chat/openai
- LangChain structured output: https://docs.langchain.com/oss/javascript/langchain/structured-output

## Scope

In scope:

- Add a protected backend endpoint for AI profile draft generation.
- Add OpenAI + LangChain dependencies to the API package.
- Add backend service code for prompt construction, structured output, and result normalization.
- Add strict field allowlisting for profile draft fields.
- Add request validation, output validation, and rate limiting.
- Add frontend UI in the existing profile create/edit flow.
- Add Redux action/reducer state for draft generation.
- Add tests for backend validation/service behavior and frontend draft application behavior.
- Update environment documentation.

Out of scope for this sprint:

- Automatically saving AI output to MongoDB.
- Admin-generated profiles on behalf of users.
- AI verification of qualifications.
- AI document/image processing.
- AI-generated subscription, admin, review, or qualification status values.
- Chatbot-style multi-turn profile coaching.
- Streaming responses.

## User Story

As a registered BodyVantage professional, I want to describe my experience, services, location, specialisms, and qualifications in my own words, so that BodyVantage can draft the profile form for me and reduce the time needed to create my profile.

## User Flow

1. User opens the profile create/edit page.
2. User enters a natural-language description into an AI assistant panel.
3. User clicks `Generate draft`.
4. Frontend sends the text to the API.
5. API authenticates the user, validates the request, rate-limits the action, and sends the input to OpenAI through LangChain.
6. API receives structured output, normalizes it, strips disallowed fields, validates it, and returns it to the client.
7. Frontend shows a preview of generated fields.
8. User clicks `Apply draft to form`.
9. Existing form fields are populated.
10. User reviews, edits, uploads any documents/images separately, and saves through the current profile save flow.

## Backend Contract

Add route:

```txt
POST /api/profile/ai-draft
```

Auth:

- Required: normal user auth token.
- Admin auth is not required.

Request body:

```json
{
  "input": "I am a Level 3 personal trainer in Manchester...",
  "currentProfile": {
    "location": "Manchester",
    "specialisationOne": "Strength training"
  }
}
```

Response body:

```json
{
  "draft": {
    "location": "Manchester",
    "telephoneNumber": null,
    "websiteUrl": null,
    "faceBook": null,
    "instagram": null,
    "description": "Professional public profile copy...",
    "specialisation": "Strength training and weight management.",
    "specialisationOne": "Strength training",
    "specialisationTwo": "Weight loss",
    "specialisationThree": null,
    "specialisationFour": null,
    "qualifications": "Level 3 Personal Training",
    "keywords": ["personal trainer", "strength training", "weight loss"]
  },
  "missingFields": ["telephoneNumber"],
  "warnings": [
    "Please verify all qualification and experience claims before saving."
  ]
}
```

Error examples:

```json
{
  "success": false,
  "message": "Profile draft input must be between 40 and 4000 characters."
}
```

```json
{
  "success": false,
  "message": "AI profile drafting is temporarily unavailable."
}
```

## Allowed Draft Fields

The AI service may return only these profile-facing fields:

- `location`
- `telephoneNumber`
- `websiteUrl`
- `faceBook`
- `instagram`
- `description`
- `specialisation`
- `specialisationOne`
- `specialisationTwo`
- `specialisationThree`
- `specialisationFour`
- `qualifications`
- `keywords`
- `missingFields`
- `warnings`

The AI service must not return or mutate:

- `user`
- `email`
- `isAdmin`
- `isQualificationsVerified`
- `qualificationVerificationStatus`
- `qualificationStatusUpdatedAt`
- `subscriptionStatus`
- `stripeCustomerId`
- `stripeSubscriptionId`
- `profileImage`
- `cloudinaryId`
- qualification document records
- reviews
- ratings
- click counters
- onboarding state

## AI Service Design

Add service:

```txt
api/services/profileDraftAIService.js
```

Recommended dependencies:

```txt
@langchain/openai
@langchain/core
langchain
zod
```

Implementation notes:

- Use `ChatOpenAI` from `@langchain/openai`.
- Use a Zod schema with LangChain structured output.
- Use `strict: true` where supported.
- Prefer `nullable()` fields over optional fields for generated profile values, so missing information is explicit.
- Keep the model configurable via environment variable.
- Do not pass passwords, JWTs, full user records, uploaded documents, or images to the model.
- Pass only the user's natural-language input and a small allowlisted `currentProfile` object when useful.

Suggested environment variables:

```txt
OPENAI_API_KEY=
OPENAI_MODEL=gpt-5.5
OPENAI_PROFILE_DRAFT_MODEL=
AI_PROFILE_DRAFT_ENABLED=true
AI_PROFILE_DRAFT_MAX_INPUT_CHARS=4000
AI_PROFILE_DRAFT_RATE_LIMIT_PER_HOUR=5
```

## Prompt Rules

System behavior:

- You draft public BodyVantage professional profile fields from user-provided text.
- Extract only information present in the user's text.
- Do not invent qualifications, experience length, location, services, phone numbers, websites, or social links.
- If a field is not present, return `null`.
- Write in clear, professional UK English.
- Avoid exaggerated claims and medical/regulatory claims unless clearly stated by the user.
- Keep copy suitable for a public professional directory.
- Warn the user to verify qualification and experience claims before saving.
- Never return admin, verification, subscription, document, image, review, or internal system fields.

## Backend Work Items

- [x] Add API dependencies.
- [x] Add environment validation for OpenAI and AI feature flag.
- [x] Add `profileDraftValidator.js` or extend existing profile validators with AI draft request schema.
- [x] Add AI rate limiter in `rateLimitMiddleware.js`.
- [x] Add `profileDraftAIService.js`.
- [x] Add controller method, likely in `profileController.js` or a dedicated `profileDraftController.js`.
- [x] Add protected route under existing profile routes.
- [x] Normalize AI output with an allowlist before responding.
- [x] Add server-side logging for draft attempts, failures, and rate-limit events without logging full user input.
- [x] Add backend tests for validation, allowlisting, disabled feature behavior, and malformed AI output.
- [x] Update `docs/environment.md`.
- [x] Update `api/docs/ROUTES.md`.

## Frontend Work Items

- [x] Add Redux constants for AI draft request/success/fail/reset.
- [x] Add Redux action to call `POST /api/profile/ai-draft`.
- [x] Add reducer state for draft generation.
- [x] Add an assistant panel to `ProfileEditView`.
- [x] Add textarea, generate button, loading state, error state, and generated draft preview.
- [x] Add `Apply draft to form` behavior that merges generated fields into existing local form state.
- [x] Do not automatically submit the profile after applying the draft.
- [x] Preserve user-entered values unless the user explicitly applies generated replacements.
- [x] Add accessible labels, status messages, and focus handling for success/error states.
- [x] Add frontend tests for generate, preview, apply, error, and no-auto-save behavior.
- [x] Add an explicit `Save draft changes` path after draft application so applied draft values persist through the existing profile update endpoint.
- [x] Update profile Redux caches on `PROFILE_UPDATE_SUCCESS` so the logged-in profile, public profile detail, public list, and admin list do not hold stale profile data after saving.

## Implementation Notes

Primary backend files:

- `api/services/profileDraftAIService.js`
- `api/controllers/profileDraftController.js`
- `api/routes/profileRoutes.js`
- `api/validators/profileValidator.js`
- `api/middleware/rateLimitMiddleware.js`
- `api/utils/auditLogger.js`
- `api/config/validateEnv.js`

Primary frontend files:

- `client/src/views/profileEditView/ProfileEditView.jsx`
- `client/src/views/profileEditView/ProfileEditView.scss`
- `client/src/store/actions/profileActions.js`
- `client/src/store/reducers/profileReducers.js`
- `client/src/store/constants/profileConstants.js`
- `client/src/store/store.js`
- `client/src/components/button/Button.jsx`

Important implementation details:

- `profileDraftAIService.js` uses `ChatOpenAI` and `withStructuredOutput(...)` with a strict Zod schema.
- `normalizeProfileDraft(...)` strips disallowed fields and returns only `draft`, `missingFields`, and `warnings`.
- The API sends only the user's natural-language input and an allowlisted `currentProfile` context to the model.
- The service defaults to `OPENAI_PROFILE_DRAFT_MODEL`, then `OPENAI_MODEL`, then `gpt-5.5`.
- The frontend converts generated rich text fields into Quill-compatible paragraph HTML before applying or saving.
- `ProfileEditView` keeps draft application and profile save as separate user actions.
- `profileDraftLimiter` is keyed by authenticated user ID where available and falls back to IP.

## Latest Fixes And Context

Applied draft persistence fix:

- Problem: the AI draft generated and applied to the visible form, but refreshing the profile edit screen showed old data.
- Root cause: applying the draft updated local form state, but the save path needed to persist the applied draft payload reliably through `PUT /api/profile`.
- Fix: `Save draft changes` calls `handleSaveAppliedProfileDraft`, which builds an update payload directly from the current draft plus existing form values, then dispatches `profileUpdateAction`.
- Redux profile caches are updated on `PROFILE_UPDATE_SUCCESS` for logged-in profile, public profile by ID, public profile list, and admin profile list.

Rate-limit response contract fix:

- Problem: the AI draft rate limiter returned `{ error: "..." }`, while the frontend action and sprint contract expect `message`.
- Fix: `profileDraftLimiter` now returns `{ success: false, message: "Too many profile draft requests. Please try again after an hour." }`.
- Regression coverage: `client/src/store/actions/profileActions.test.js` verifies the AI draft action surfaces the API message.

Adjacent login regression fixed during post-sprint validation:

- Problem: after logout, clicking member `Login` gave no response.
- Root cause: the shared `Button` component defaults to `type="button"`, and `LoginFormView` did not set `type="submit"`.
- Fix: the member login button now explicitly uses `type="submit"`.
- Regression coverage: `client/src/views/loginFormView/LoginFormView.test.jsx` verifies valid credentials dispatch `loginAction` when the Login button is clicked.

Autofill background fixed during form QA:

- Problem: filled/autofilled inputs could appear with a white browser autofill background on the dark UI.
- Root cause: the WebKit/Firefox autofill overrides used transparent inset paint, which did not cover the browser's native light autofill surface.
- Fix: global and component-specific autofill rules now paint a dark inset background and preserve the expected light text and caret colors.
- Touched files:
  - `client/src/index.scss`
  - `client/src/components/inputField/InputField.scss`
  - `client/src/views/contactFormView/ContactFormView.scss`
  - `client/src/views/reviewerLoginView/ReviewerLoginView.scss`

Registration checkout submit fixed:

- Problem: at `/registration`, valid form details enabled `Subscribe Now`, but clicking the button did nothing.
- Root cause: the shared `Button` component defaults to `type="button"`, and `RegistrationView` did not set `type="submit"`.
- Fix: the registration checkout button now explicitly uses `type="submit"`.
- Regression coverage: `client/src/views/registrationView/RegistrationView.test.jsx` verifies valid details dispatch `createCheckoutSessionAction`.

Form button audit completed:

- A follow-up review checked all `<Button>` components inside `<form>` elements.
- Any submit action that relied on the shared default now explicitly sets `type="submit"`.
- Existing secondary actions inside forms, such as cancel, draft generation, and file-upload helpers, keep explicit `type="button"` where appropriate.
- Additional files updated:
  - `client/src/views/forgotPassword/ForgotPassword.jsx`
  - `client/src/views/resetPassword/ResetPassword.jsx`
  - `client/src/views/reviewerForgotPassword/ReviewerForgotPassword.jsx`
  - `client/src/views/reviewerResetPassword/ReviewerResetPassword.jsx`
  - `client/src/views/reviewerRegisterView/ReviewerRegisterView.jsx`
  - `client/src/views/reviewerLoginView/ReviewerLoginView.jsx`

### Adjacent public/member page redesign follow-up

- Goal: make the public/member support views visually consistent using the BodyVantage dark/gold brand language while retaining existing fonts, copy, routes, and behavior.
- Redesigned views:
  - `client/src/views/HomeView/HomeView.jsx`
  - `client/src/views/aboutView/AboutView.jsx`
  - `client/src/views/contactFormView/ContactFormView.jsx`
  - `client/src/views/loginFormView/LoginFormView.jsx`
  - `client/src/views/registrationView/RegistrationView.jsx`
  - `client/src/views/forgotPassword/ForgotPassword.jsx`
  - `client/src/views/preRegistrationView/PreRegistrationView.jsx`
  - `client/src/views/faqsView/FaqsView.jsx`
  - `client/src/views/sectorLandingView/SectorLandingView.jsx`
  - `client/src/components/footer/Footer.jsx`
  - `client/src/components/bodyVantage/BodyVantage.jsx`
- Matching SCSS files were updated for the same views.
- Design decisions:
  - Retain BodyVantage dark background, gold accent, Bebas Neue headings, Comfortaa body text, and existing page wording.
  - Replace old `fieldset`/`legend` page wrappers with semantic `main`, `article`, `header`, and `section` structure.
  - Remove the restrictive outer bordered page frame after review; keep structure in inner form panels, accordion rows, plan cards, timeline blocks, CTA blocks, and highlighted content panels.
  - Use the real header logo asset, `client/src/assets/logo/logo.svg`, as the decorative hero mark instead of text-only `BV`.
  - Reduce hero headline scale, tighten the gap between headline text and the decorative logo mark, cap the home hero height, and keep the hero mark out of normal mobile flow so it does not wrap beneath headings or push task content down.
  - Expand page wrapper width from `1280px` to shared shell tokens based on `1680px` max width with responsive large-screen gutters, while keeping paragraph measure caps for readability.
  - Keep redesigned view styling page-scoped so global/shared components are not unexpectedly restyled elsewhere.
  - Home page now treats the search input as the primary call to action. The hero headline is reduced, the `Professional verification` kicker was removed, the search input surface was made more opaque, and search result cards were constrained so profile copy does not collapse into narrow columns.
  - Background grid overlays were removed from the refreshed pages after review; only restrained dark/gold glow and structural accents remain.
  - Sector landing pages, including `/personal-trainers`, now use the same open hero, section-heading, numbered section, and CTA patterns as the other refreshed informational views.
  - The shared `BodyVantage` component was tightened into an inline brand token. Section-heading SCSS selectors now use direct-child spans so nested brand-token spans are not accidentally styled as section numbers.
  - The footer was redesigned to match the same dark/gold, open-width visual system.
  - The legacy React favicon was replaced with a BV SVG favicon at `client/public/public/favicon.svg`; `client/index.html`, `client/public/public/index.html`, and `client/public/public/manifest.json` were updated accordingly.
  - Shared page-shell spacing tokens were added in `client/src/styles/_theme.scss` for max width, large-screen gutters, vertical shell padding, hero spacing, desktop/mobile hero mark size, hero title size, and intro-block spacing.
  - Hero bottom borders were removed from refreshed pages; the design now relies on spacing, alignment, and content grouping rather than horizontal divider lines.
  - Detached left-rule intro blocks on About, Pre-registration, and sector landing pages were replaced with aligned two-column editorial summary bands.
  - Form/support context copy on Contact, Login, Registration, FAQ, and Forgot Password was simplified by removing decorative top/left rule treatments.
  - Decorative horizontal separators were removed from refreshed page intro bands, content sections, support/context blocks, FAQ answers, registration plan section, contact social-links section, and the footer top edge.
  - Mobile spacing was tightened across refreshed pages: wrapper padding, hero/content gaps, hero bottom padding, first content spacing, and context/intro padding were reduced so lower-page elements are visible sooner without excessive scrolling.
- Behavior kept intact:
  - Home search URL synchronization, debounced profile fetching, profile cards, keyword highlighting, and pagination.
  - Contact form dispatch, validation, and messages.
  - Member login dispatch and validation.
  - Registration checkout/session dispatch, plan selection, validation, and password strength display.
  - Forgot-password reset-link dispatch and messages.
  - FAQ accordion state and data.
  - Pre-registration CTAs and support links.
  - Sector landing CTAs and SEO metadata.
- Consistency sweep completed across the touched views:
  - Shared page width and padding.
  - Shared large-screen gutters and consistent shell padding across desktop, tablet, and mobile.
  - Shared compact mobile spacing for wrappers, hero/content gaps, and first content blocks.
  - Shared hero grid, logo mark, headline sizing, spacing, and tighter vertical rhythm.
  - Shared section-heading pattern.
  - Shared open-page layout with page-level frame and decorative divider lines removed.
  - Shared intro treatment for overview/proposition copy where needed.
  - Consistent SCSS imports.
  - Build verification was rerun after the latest polish; the existing large chunk warning remains.

### Useful manual QA input

```txt
I am a Level 4 sports massage therapist and personal trainer based in Bristol, working with runners, cyclists, and desk-based professionals. I specialise in injury prevention, mobility, strength conditioning, and postural correction. I hold Level 3 Personal Training, Level 4 Sports Massage Therapy, and first aid qualifications. Clients usually come to me for lower back pain, tight hips, running niggles, strength training plans, and recovery support. My phone number is 07123 456789 and my website is bristolperformance.co.uk.
```

## Suggested UI Copy

Panel heading:

```txt
Profile draft assistant
```

Textarea label:

```txt
Describe your professional background, services, location, qualifications, and specialisms.
```

Button:

```txt
Generate draft
```

Preview action:

```txt
Apply draft to form
```

Warning text:

```txt
Review all generated details before saving. Qualification and experience claims must be accurate.
```

## Testing Plan

Backend:

- Valid request returns allowed draft fields only.
- Short input is rejected.
- Excessively long input is rejected.
- Unauthenticated request is rejected.
- Disabled feature flag returns a clear unavailable message.
- AI output containing disallowed fields is stripped.
- AI output with invalid field types is rejected or normalized safely.
- Rate limiter blocks repeated draft requests.
- Service handles OpenAI/LangChain errors without leaking internals.

Frontend:

- User can enter text and request a draft.
- Loading state appears while request is pending.
- Error state appears on failed request.
- Preview appears on successful request.
- Applying the draft updates the form.
- Applying the draft does not save the profile.
- Existing required validation still runs on normal save.
- Keyboard and screen reader behavior is acceptable.
- Saved applied draft data remains visible after refresh.
- Login still submits after logout and return to `/login`.
- Registration checkout submits after valid details are entered and `Subscribe Now` is clicked.
- Shared `Button` submit controls inside forms use explicit `type="submit"`.
- Filled/autofilled form fields retain the dark UI treatment rather than showing browser-default white autofill backgrounds.

Manual QA examples:

- Personal trainer with full details.
- Beauty professional with partial details.
- User includes qualifications but no location.
- User includes location and services but no qualifications.
- User writes unsupported or exaggerated claims.
- User pastes contact details.
- User enters irrelevant text.

## Acceptance Criteria

- AI provider access exists only in the Node API.
- The client never receives or uses `OPENAI_API_KEY`.
- The endpoint requires authenticated user access.
- The endpoint is rate-limited.
- AI output is structured, allowlisted, and validated.
- AI output cannot set verification, subscription, admin, document, image, review, or internal fields.
- Generated content is previewed before being applied.
- Applying generated content fills the form but does not save it.
- Existing profile save behavior remains the source of truth.
- Environment and route documentation are updated.
- Relevant backend and frontend tests are added or documented if blocked.
- AI draft rate-limit failures display a clear API message in the frontend.
- Applied draft values persist after clicking `Save draft changes` and refreshing the profile edit screen.
- Registration checkout action dispatches when valid form details are submitted.
- All shared `Button` components inside forms have explicit `type` attributes, with submit and secondary actions differentiated.
- Browser autofill styling remains consistent with the dark BodyVantage form design.

## Known Risks

- AI may produce confident wording that overstates qualifications or regulated services.
- Users may paste sensitive personal information into the assistant.
- Structured output can still fail if model/provider configuration changes.
- Existing `ProfileEditView` is large, so frontend changes should be kept tightly scoped.
- The current backend search validation likely strips `search` from profile list queries; avoid mixing that fix into this sprint unless explicitly approved.
- OpenAI image generation is not implemented. If revisited, prefer a separate sprint with explicit product, moderation, storage, cost, and public-profile review decisions.
- Build still reports the existing large chunk warning for the main JS bundle and large media/image assets.
- Test output still includes existing Sass legacy JS API deprecation warnings and React Router future flag warnings.

## Implementation Order

1. Backend dependencies and environment documentation.
2. Backend validator, rate limiter, and route/controller skeleton.
3. LangChain/OpenAI service with structured schema and allowlist normalization.
4. Backend tests.
5. Frontend Redux action/reducer.
6. `ProfileEditView` assistant panel and draft application behavior.
7. Frontend tests.
8. Manual QA.
9. Final documentation update and implementation summary.

## Definition of Done

- Feature works locally behind `AI_PROFILE_DRAFT_ENABLED=true`.
- Feature fails gracefully when disabled or when OpenAI is unavailable.
- No secrets are exposed to the frontend.
- Draft generation never writes to the database directly.
- User can apply the draft and then save using the existing profile form.
- Tests or manual verification notes cover the main success and failure paths.

## Verification Commands

Backend:

```bash
cd api
node --test tests/profileDraftAIService.test.js
npm test
```

Frontend:

```bash
cd client
npm test -- --run src/views/profileEditView/ProfileEditView.test.jsx
npm test -- --run src/store/reducers/profileReducers.test.js
npm test -- --run src/store/actions/profileActions.test.js
npm test -- --run src/views/loginFormView/LoginFormView.test.jsx
npm test -- --run src/views/registrationView/RegistrationView.test.jsx
npm test -- --run src/views/loginFormView/LoginFormView.test.jsx src/views/registrationView/RegistrationView.test.jsx
npm test -- --run
npm run build
```

Latest verification notes:

- `api npm test` passed.
- `client npm test -- --run` passed before the final login test was added.
- `client npm test -- --run src/views/loginFormView/LoginFormView.test.jsx` passed after the login submit fix.
- `client npm run build` passed.
- `client npm run build` also passed after the public/member page redesign and consistency sweep.
- `client npm run build` passed after the large-screen spacing, hero rhythm, intro treatment, and divider-removal UI pass.
- `client npm run build` passed after the mobile hero mark flow and mobile spacing pass.
- `client npm run build` passed after the autofill background fix.
- `client npm test -- --run src/views/registrationView/RegistrationView.test.jsx` passed after the registration submit fix.
- `client npm test -- --run src/views/loginFormView/LoginFormView.test.jsx src/views/registrationView/RegistrationView.test.jsx` passed after the form button audit.
- `client npm run build` passed after the form button audit.
- Build still reports the existing large chunk warning for the main JS bundle and large media/image assets.
