# AI Profile Draft Handoff

Date: 2026-05-22

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
npm run build
```

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
