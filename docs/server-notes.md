# Server Notes

- Production static assets now served from `client/dist` (Vite build) with SPA fallback to `index.html`.
- Removed duplicate `userReviewRoutes` mount and extraneous `imageUploadRoutes` catch-all to avoid double handlers and unintended paths.
- CORS is open (`cors()` default); tighten allowed origins for production if needed.
- Consider adding security middleware (helmet, rate limiting) for auth-sensitive endpoints.
