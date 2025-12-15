# Backend User Routes

Auth and profile endpoints exposed under `/api` (see `api/routes/userRoutes.js`).

- `POST /api/users/login` — authenticate, returns JWT and profile basics.
- `POST /api/users` — register; sends verification email via `MAILER_LOCAL_URL`.
- `GET /api/users/profile` — current user profile (auth required).
- `PUT /api/users/profile` — update current user profile (auth required).
- `POST /api/user-forgot-password` — issue 15-minute reset token, sends email using `RESET_PASSWORD_LOCAL_URL`.
- `PUT /api/user-update-password` — reset password using valid token; clears stored reset token after use.
- `GET /api/user/profile/:id` — public profile lookup by id.
- Admin-only:
  - `GET /api/users` — list all users.
  - `DELETE /api/users/:id` — delete user by id.
  - `PUT /api/user/profile/:id` — toggle `isAdmin` (expects `val` boolean).

Verification:
- `GET /api/verify?token=...` (legacy: `/api/verify/token=:id`) — confirm user email; redirects to frontend origin.
- `GET /api/verifyReviewer?token=...` (legacy: `/api/verifyReviewer/token=:id`) — confirm reviewer email; redirects to frontend origin.

Contact:
- `POST /api/send` — submit contact form (name, email, message); emails the user and responds 201 on success.

Profiles:
- `GET /api/profiles` — list all profiles (public).
- `GET /api/profile/:id` — profile by id (public).
- `GET /api/profile` — current user profile (auth).
- `POST /api/profiles` — create profile for current user (auth).
- `PUT /api/profile/:id` — update profile for user id (auth).
- `PUT /api/profile-clicks` — increment profile click counter (public).
- Admin: `GET/DELETE/PUT /api/profiles/admin/:id` (list/delete profile, verify qualifications); `DELETE /api/profile/review/admin/:id` (remove review).

Reviewer Accounts:
- `POST /api/users-review` — register reviewer; sends verification email.
- `POST /api/users-review/login` — reviewer login.
- `GET /api/reviewer/public/:id` — public reviewer by id.
- Admin: `GET /api/reviewers/admin/:id` (all reviewers), `DELETE /api/reviewer/admin/:id` (delete reviewer).
