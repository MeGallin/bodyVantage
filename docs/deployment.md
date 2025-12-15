# Deployment Guide

## API (Render or similar Node host)
- Install dependencies: `npm install` in `api/`.
- Required env (see `docs/environment.md`): `MONGO_URI`, `JWT_SECRET`, SMTP vars, `MAILER_LOCAL_URL`, `RESET_PASSWORD_LOCAL_URL`, `CLOUDINARY_*`, `NODE_ENV=production`.
- Start command: `node server.js` (uses `PORT` if provided).
- Static assets: server now serves `client/dist` in production; if API and frontend are deployed separately, disable static serving or keep `client/dist` empty.
- CORS: currently open (`cors()` default). For production, configure allowed origins explicitly.
- Health: `/` returns a simple string in non-production; consider adding `/health` if your host needs a probe.

## Frontend (Vite build)
- Install dependencies: `npm install` in `client/`.
- Build: `npm run build` (outputs to `client/dist`).
- Env:
  - Dev: `VITE_API_BASE_URL=http://127.0.0.1:5000`
  - Prod: `VITE_API_BASE_URL=https://api-cllf.onrender.com` (no trailing slash)
- Serve: Any static host. Ensure an SPA fallback so routes like `/reset-password/:token` and `/fullProfile/:id` load `index.html`.
  - Apache: use the provided `.htaccess` rewrite pattern (see 404 fix notes).
  - Nginx: `try_files $uri /index.html;`
  - Render Static: rewrite `/*` → `/index.html`.

## Password Reset & Verification Links
- API sends verification links via `MAILER_LOCAL_URL` (should point to API origin).
- Password reset links use `RESET_PASSWORD_LOCAL_URL` (should point to frontend origin).
- Reset tokens expire after 15 minutes and are cleared on use.

## Local Development
- Run API: `npm run dev` if using nodemon (or `node server.js`) from `api/`, ensure `MONGO_URI` is set.
- Run client: `npm run dev` from `client/`; Vite dev server proxies `/api` to `http://127.0.0.1:5000`.
