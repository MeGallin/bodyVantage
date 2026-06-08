# Subscription Login and Checkout Handoff

Last updated: 2026-06-08

## Current Rule

Login is authentication only. A user who enters valid credentials should be sent to
`/user-profile-edit` and must not be redirected to `/subscribe` by the login page.

Subscription/payment is enforced when the user tries to use paid professional
profile features. The API returns `402` with:

```text
An active subscription is required to edit your professional profile.
```

## Fixed Regression

Reported issue: paid users were logging in successfully, then immediately landing
on `/subscribe`.

Root cause: `LoginFormView` was making a client-side subscription decision from
the login payload. Stale or legacy subscription fields could therefore send paid
users to the payment page.

Resolution:

- `client/src/views/loginFormView/LoginFormView.jsx` no longer checks
  subscription status.
- Successful login always navigates to `/user-profile-edit`.
- Backend middleware still enforces subscription access on paid actions.

## Checkout Flow

Checkout creation:

- `POST /api/checkout-session`
- Creates or reuses the user and returns only the Stripe hosted checkout URL.
- It does not hydrate client login state before payment succeeds.

Checkout success:

- Stripe redirects to `/subscribe/success?session_id=...`.
- `SubscriptionSuccess.jsx` dispatches `verifyCheckoutSessionAction(sessionId)`.
- Client calls `GET /api/checkout-session/:sessionId`.
- API retrieves the Stripe checkout session, finds the user by
  `stripeCustomerId`, syncs subscription state, and returns a login payload.
- Client stores the returned user in Redux login state and `localStorage`.

## Subscription Reconciliation

`api/services/subscriptionStatusService.js` is the shared sync point.

It can reconcile a stale Mongo record when at least one of these fields exists:

- `stripeCustomerId`
- `stripeSubscriptionId`

The normal checkout path should save `stripeCustomerId` before redirecting to
Stripe. If a historic paid user has neither Stripe ID in Mongo, the app cannot
automatically infer the Stripe account safely during login. Those records need
manual audit/backfill, likely by matching Stripe customer email to Mongo user
email.

## Access Rules

Allowed subscription statuses:

- `active`
- `trialing`
- legacy-compatible `pending` only when `isSubscribed === true`
- missing payment status only when `isSubscribed === true`

Rejected statuses:

- `canceled`
- `failed`
- `incomplete`
- `incomplete_expired`
- `past_due`
- `paused`
- `unpaid`

Expired `currentPeriodEnd` also blocks access.

Admins bypass subscription checks.

## Key Files

API:

- `api/controllers/userController.js`
- `api/middleware/authMiddleware.js`
- `api/routes/stripeRoutes.js`
- `api/services/subscriptionStatusService.js`
- `api/utils/subscriptionStatus.js`
- `api/tests/authMiddleware.test.js`
- `api/tests/subscriptionStatusService.test.js`

Client:

- `client/src/views/loginFormView/LoginFormView.jsx`
- `client/src/views/loginFormView/LoginFormView.test.jsx`
- `client/src/views/subscriptionView/SubscriptionSuccess.jsx`
- `client/src/store/actions/stripeActions.js`
- `client/src/store/actions/stripeActions.test.js`
- `client/src/store/reducers/stripeReducers.js`
- `client/src/store/constants/stripeConstants.js`

## Verification Commands

API:

```bash
cd api
npm test
node -c services/subscriptionStatusService.js
node -c utils/subscriptionStatus.js
node -c controllers/userController.js
node -c routes/stripeRoutes.js
node -c middleware/authMiddleware.js
node -c config/stripe.js
```

Client:

```bash
cd client
npm test -- src/store/actions/stripeActions.test.js src/views/loginFormView/LoginFormView.test.jsx --run
npm test -- --run
npx vite build
```

## Production Follow-Up

Run a subscription audit/backfill for historic users:

```bash
cd api
node scripts/auditSubscriptions.js
```

Look especially for paid Stripe customers with Mongo users missing both
`stripeCustomerId` and `stripeSubscriptionId`.
