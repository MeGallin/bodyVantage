# AI Profile Draft Handoff

Date: 2026-05-22

Last updated: 2026-05-29

## Feature Context

The admin/profile edit area now includes an AI-assisted profile draft feature. A user can enter natural language about their professional background, services, qualifications, location, contact details, and specialisms. The API uses OpenAI through LangChain to convert that text into structured profile form fields.

Primary implementation plan:

- Sprint document: `docs/ai-profile-draft-sprint.md`
- Backend endpoint: `POST /api/profile/ai-draft`
- Frontend UI: `client/src/views/profileEditView/ProfileEditView.jsx`
- Model setting: `OPENAI_MODEL=gpt-5.5`, with optional `OPENAI_PROFILE_DRAFT_MODEL` override

## Backend Implementation

Key files:

- `api/services/profileDraftAIService.js`
- `api/controllers/profileDraftController.js`
- `api/routes/profileRoutes.js`
- `api/validators/profileValidator.js`
- `api/middleware/rateLimitMiddleware.js`
- `api/utils/auditLogger.js`
- `api/config/validateEnv.js`

Important env vars:

- `AI_PROFILE_DRAFT_ENABLED=true`
- `OPENAI_API_KEY=...`
- `OPENAI_MODEL=gpt-5.5`
- Optional: `OPENAI_PROFILE_DRAFT_MODEL`
- Optional: `AI_PROFILE_DRAFT_MAX_INPUT_CHARS`
- Optional: `AI_PROFILE_DRAFT_RATE_LIMIT_PER_HOUR`

The API returns an allowlisted draft object only. It does not send or return user IDs, tokens, passwords, documents, images, ratings, reviews, onboarding fields, or admin fields.

## Frontend Implementation

Key files:

- `client/src/views/profileEditView/ProfileEditView.jsx`
- `client/src/views/profileEditView/ProfileEditView.scss`
- `client/src/store/actions/profileActions.js`
- `client/src/store/reducers/profileReducers.js`
- `client/src/store/constants/profileConstants.js`
- `client/src/store/store.js`
- `client/src/components/button/Button.jsx`

User flow:

1. Open Profile Draft Assistant.
2. Enter natural language profile information.
3. Click `Generate draft`.
4. Review generated fields.
5. Click `Apply draft to form`.
6. Click `Save draft changes`.
7. Confirm `Profile updated successfully`.
8. Refreshing the profile edit screen should now show the saved data.

## Latest Fix

Problem observed:

- The AI draft generated and applied to the visible form.
- Refreshing the profile edit screen showed old data.

Root cause:

- Applying the draft changed local form state.
- The save path needed to persist the applied draft payload reliably through `PUT /api/profile`.

Fix applied:

- `Save draft changes` now calls an explicit `handleSaveAppliedProfileDraft`.
- That handler builds the update payload directly from the current draft plus existing form values.
- It sends the normal `profileUpdateAction` to `PUT /api/profile`.
- The Redux profile caches are updated on `PROFILE_UPDATE_SUCCESS` for:
  - logged-in profile
  - public profile by ID
  - public profile list
  - admin profile list

The user confirmed the fix worked.

## Adjacent UI Redesign Follow-up

After the AI profile draft validation, several public/member-facing views were redesigned for visual consistency. This work is adjacent context and does not change the AI draft API contract.

Touched views:

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

Matching SCSS files were updated for each view.

Design direction:

- Retain the BodyVantage dark/gold palette, Bebas Neue heading style, Comfortaa body style, and existing page copy.
- Replace old `fieldset`/`legend` page wrappers with semantic page structure.
- Remove the large restrictive outer page frame after review.
- Keep structure only where useful: forms, accordions, pricing/plan cards, timeline blocks, CTA blocks, and highlighted content panels.
- Use `client/src/assets/logo/logo.svg` as the decorative hero mark instead of text-only `BV`.
- Reduce hero headline size, tighten the hero gap between headline and logo mark, cap the home hero height, and keep the hero mark out of normal mobile flow so it does not wrap beneath headings or push task content down.
- Use shared page-shell tokens in `client/src/styles/_theme.scss` for redesigned page wrappers: `1680px` max width, responsive large-screen gutters, tighter vertical padding, hero spacing, desktop/mobile hero mark sizing, hero title sizing, and intro-block spacing.
- Home is now search-first: the search input is the primary CTA, the headline is reduced, the `Professional verification` kicker was removed, the search input background was made more solid, and result cards were widened so profile text does not collapse.
- Background grid overlays were removed from refreshed pages; dark surfaces now rely on restrained gold glow/accent treatments only.
- Sector landing pages, including `/personal-trainers`, now share the same open hero, numbered sections, and CTA treatment as the other refreshed pages.
- The shared `BodyVantage` inline token was updated for consistent baseline, scale, no-wrap behavior, and gold-accent styling. Section-heading selectors were scoped to direct child spans to avoid styling nested `BodyVantage` spans.
- The footer now uses the same open-width dark/gold system with logo, navigation links, build details, live date/time, and copyright.
- The React favicon was replaced by a BV SVG favicon at `client/public/public/favicon.svg`; active and legacy HTML plus the web manifest now reference BodyVantage branding.
- Hero bottom borders and other decorative horizontal divider lines were removed from refreshed public/member pages and the footer, leaving spacing and content grouping to define sections.
- Detached left-rule intro blocks on About, Pre-registration, and sector landing pages were replaced with aligned editorial summary bands that match the page grid.
- Contact, Login, Registration, FAQ, and Forgot Password context/support copy no longer uses the old skinny left-rule or top-rule callout treatment.
- Mobile spacing was tightened across refreshed pages: wrapper padding, hero/content gaps, hero bottom padding, first content spacing, and context/intro padding were reduced so forms/content appear sooner without excessive scrolling.

Behavior preserved:

- Home search URL synchronization, debounced profile fetching, profile cards, keyword highlighting, and pagination.
- Contact form validation, dispatch, and success/error messages.
- Member login validation and dispatch; login button remains `type="submit"`.
- Registration validation, password strength display, monthly/annual plan selection, and checkout dispatch.
- Forgot-password reset-link dispatch and messages.
- FAQ accordion state and FAQ data.
- Pre-registration CTAs and support links.
- Sector page CTAs and SEO metadata.

Consistency sweep:

- Shared page width and padding.
- Shared large-screen gutters and shell padding across desktop, tablet, and mobile.
- Shared compact mobile spacing for wrappers, hero/content gaps, and first content blocks.
- Shared hero grid, logo mark, headline sizing, spacing, and tighter vertical rhythm.
- Shared section-heading treatment.
- Shared open-page layout with no page-level border/frame and no decorative horizontal divider lines.
- Shared intro treatment for overview/proposition copy where pages need it.
- Consistent SCSS imports across touched views.
- Build verification was rerun after the latest public-site polish; the existing large chunk warning remains.

## Latest Form Follow-up

Two public/member form regressions were investigated and fixed after the UI polish.

Input autofill background:

- Problem: filled/autofilled form controls could show a white browser background against the dark BodyVantage UI.
- Root cause: the WebKit/Firefox autofill override used transparent inset paint, allowing the browser's native light autofill surface to show through.
- Fix: global and component-specific autofill rules now paint a dark inset background and preserve light text/caret colors.
- Touched files:
  - `client/src/index.scss`
  - `client/src/components/inputField/InputField.scss`
  - `client/src/views/contactFormView/ContactFormView.scss`
  - `client/src/views/reviewerLoginView/ReviewerLoginView.scss`

Form submit buttons:

- Problem: the member registration form at `/registration` passed validation but clicking `Subscribe Now` did nothing.
- Root cause: the shared `Button` component defaults to `type="button"`, and the registration submit button did not explicitly set `type="submit"`.
- Fix: `client/src/views/registrationView/RegistrationView.jsx` now sets `type="submit"` on the checkout submit button.
- Regression coverage: `client/src/views/registrationView/RegistrationView.test.jsx` verifies valid registration details dispatch `createCheckoutSessionAction`.
- Follow-up review: all `<Button>` components inside `<form>` elements were audited. Remaining submit buttons now have explicit `type="submit"` and secondary in-form actions keep explicit `type="button"` where needed.
- Additional touched files:
  - `client/src/views/forgotPassword/ForgotPassword.jsx`
  - `client/src/views/resetPassword/ResetPassword.jsx`
  - `client/src/views/reviewerForgotPassword/ReviewerForgotPassword.jsx`
  - `client/src/views/reviewerResetPassword/ReviewerResetPassword.jsx`
  - `client/src/views/reviewerRegisterView/ReviewerRegisterView.jsx`
  - `client/src/views/reviewerLoginView/ReviewerLoginView.jsx`

## Useful Test Input

```text
I am a Level 4 sports massage therapist and personal trainer based in Bristol, working with runners, cyclists, and desk-based professionals. I specialise in injury prevention, mobility, strength conditioning, and postural correction. I hold Level 3 Personal Training, Level 4 Sports Massage Therapy, and first aid qualifications. Clients usually come to me for lower back pain, tight hips, running niggles, strength training plans, and recovery support. My phone number is 07123 456789 and my website is bristolperformance.co.uk.
```

## Verification Commands Run

Frontend:

```bash
cd client
npm test -- --run src/views/profileEditView/ProfileEditView.test.jsx
npm test -- --run src/store/reducers/profileReducers.test.js
npm test -- --run src/views/loginFormView/LoginFormView.test.jsx
npm run build
```

Additional latest verification:

- `client npm run build` passed after the public/member page redesign and consistency sweep.
- `client npm test -- HomeView --run` passed after home search refinements and shared `BodyVantage` styling changes.
- `client npm run build` passed after sector landing alignment, background grid removal, and favicon replacement.
- `client npm run build` passed after the large-screen spacing, hero rhythm, intro treatment, and divider-removal UI pass.
- `client npm run build` passed after the mobile hero mark flow and mobile spacing pass.
- `client npm run build` passed after the autofill background fix.
- `client npm test -- --run src/views/registrationView/RegistrationView.test.jsx` passed after the registration submit fix.
- `client npm test -- --run src/views/loginFormView/LoginFormView.test.jsx src/views/registrationView/RegistrationView.test.jsx` passed after the form button audit.
- `client npm run build` passed after the form button audit.
- Build still reports the existing large chunk warning for the main JS bundle and large media/image assets.

Backend:

```bash
cd api
node --test tests/profileDraftAIService.test.js
npm test
```

Known lint status:

- `client npm run lint` still has pre-existing project-wide lint failures unrelated to the AI draft work.

## Local Runtime Notes

During debugging, these ports were in use:

- Frontend Vite: `http://127.0.0.1:5174/`
- API: `http://127.0.0.1:5000/`

If the API crashes after reboot or port conflicts occur:

```bash
lsof -iTCP:5000 -sTCP:LISTEN -n -P
```

Then stop the stale process and restart the API from `api/`.

## Security Note

The OpenAI API key was discussed during debugging. Do not paste or commit secrets. Keep secrets only in local `.env` files.
