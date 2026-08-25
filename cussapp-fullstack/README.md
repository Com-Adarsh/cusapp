# CUSSApp — Full-Stack JavaScript Prototype

A runnable full-stack JavaScript version of CUSSApp.

## Stack
- Node.js
- Express 5
- Vanilla JavaScript frontend
- JSON persistence for zero-config local development
- JWT authentication
- bcrypt password hashing
- REST API

## Run
1. Install Node.js 18+.
2. In this folder run:

```bash
npm install
npm start
```

3. Open http://localhost:3000

For development:

```bash
npm run dev
```

## API
- `POST /api/auth/register`
- `POST /api/auth/login`
- `GET /api/me`
- `GET /api/home`
- `GET /api/events`
- `GET /api/opportunities`
- `GET /api/polls`
- `POST /api/polls/:id/vote`
- `GET /api/campus`
- `POST /api/grievances`
- `GET /api/grievances`
- `POST /api/ask`

## Production next steps
Replace `data/db.json` with PostgreSQL, use university SSO/OAuth, move JWT secret to environment variables, add admin RBAC, verified document ingestion for Ask CUSAT, rate limiting, audit logs, and real CUSAT integrations.
