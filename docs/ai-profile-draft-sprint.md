# Sprint: AI Profile Draft Assistant

## Status

Implemented in the Node API and existing React profile edit flow.

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

## Known Risks

- AI may produce confident wording that overstates qualifications or regulated services.
- Users may paste sensitive personal information into the assistant.
- Structured output can still fail if model/provider configuration changes.
- Existing `ProfileEditView` is large, so frontend changes should be kept tightly scoped.
- The current backend search validation likely strips `search` from profile list queries; avoid mixing that fix into this sprint unless explicitly approved.

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
