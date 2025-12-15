# Environment Variables

Client (Vite):
- `client/.env.development`: `VITE_API_BASE_URL` (e.g., `http://127.0.0.1:5000` for dev)
- `client/.env.production`: `VITE_API_BASE_URL` (e.g., `https://api-cllf.onrender.com`)

API (`api/.env`):
- `MONGO_URI` — database connection string
- `JWT_SECRET` — signing key for auth/reset tokens
- `MAILER_HOST`, `MAILER_USER`, `MAILER_PW` — SMTP settings
- `MAILER_LOCAL_URL` — API origin for verification links (e.g., `https://api-cllf.onrender.com/`)
- `RESET_PASSWORD_LOCAL_URL` — frontend origin for password reset links (e.g., `http://localhost:5173` for dev, `https://bodyvantage.co.uk` for prod)
- `FRONTEND_URL` — allowed CORS origin for the deployed frontend (e.g., `https://bodyvantage.co.uk`)
- `PORT`, `NODE_ENV` — usual runtime settings
- `CLOUDINARY_*` — media storage creds
