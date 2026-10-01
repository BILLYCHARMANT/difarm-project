# DiFarm

Farm management app for livestock farms: cattle, milk/meat production and sales, stock and suppliers, vaccinations, inseminations, waste logs and activity logs, with role-based access across multiple farms.

- **Frontend:** Next.js 15 (Pages Router), React 18, Tailwind
- **API:** Express + Prisma in `backend/`, served by Next.js at `/api/v1`
- **Database:** PostgreSQL, schema in `prisma/schema.prisma`

Everything runs from this repo root; there is no separate backend checkout.

## Getting started

Prerequisites: Node.js 18.18+ (required by Next.js 15) and a PostgreSQL database.

1. Create `.env` in the repo root (see [Environment](#environment)). At minimum:

   ```env
   DATABASE_URL=postgresql://user:password@localhost:5432/difarm
   JWT_SECRET=change-me
   JWT_VERIF_SECRET=change-me-too
   ```

2. Install, create the tables, seed the default users, and start:

   ```bash
   npm run setup      # first time only: installs root + backend deps, generates Prisma client
   npm run db:push    # create/update tables from prisma/schema.prisma (stop dev servers first)
   npm run seed       # optional: default logins + a demo farm
   npm run dev
   ```

3. Open **http://localhost:3003**. The API answers on the same port at `/api/v1`; check `http://localhost:3003/api/v1/health`.

## Dev modes

| Command | What runs | Ports |
|---------|-----------|-------|
| `npm run dev` | One Next.js process. The Express API runs inside it via `src/pages/api/v1/[[...path]].ts`. This is how it runs on Vercel. | UI + API on `3003` |
| `npm run dev:all` | Next.js **and** a standalone Express server (`backend/server.ts`, hot reload with nodemon) | UI `3003`, API `4000` |
| `npm run dev:api` | Standalone Express server only | API `4000` |

The UI port comes from `FRONTEND_URL` and the standalone API port from `PORT`. Each script frees its port before starting.

> **Important:** the browser sends API calls to `NEXT_PUBLIC_SERVER_URL` when it is set. With `npm run dev`, leave it **unset** so the UI calls its own `/api/v1`. Set it to `http://localhost:4000` only with `npm run dev:all` if you want the UI to use the standalone server.

`npm run dev:api` regenerates `backend/.env` from the root `.env` on every start, so edit the root file, not `backend/.env`.

## Environment

All settings live in the root `.env` (git-ignored).

| Variable | Required | Default | Purpose |
|----------|----------|---------|---------|
| `DATABASE_URL` | yes | — | PostgreSQL connection string |
| `JWT_SECRET` | yes | — | Signs login tokens (also the session secret) |
| `JWT_VERIF_SECRET` | yes | — | Signs email-verification tokens |
| `EXPIRE_TIME` | no | `7d` | Login token lifetime |
| `EXPIRE_VERIF_TIME` | no | `24h` | Verification token lifetime |
| `FRONTEND_URL` | no | `http://localhost:3003` | Sets the dev UI port; allowed CORS origin |
| `PORT` | no | `4000` | Standalone API port (`dev:all` / `dev:api`) |
| `NEXT_PUBLIC_SERVER_URL` | no | same origin | API host for the browser — see [Dev modes](#dev-modes) |
| `EMAIL_USERNAME`, `EMAIL_PASS`, `EMAIL` | no | — | Gmail account (nodemailer) for password-reset and account emails |
| `ALLOW_SUPER_REGISTER` | no | — | `true` allows `POST /api/v1/auth/register/super` even after a super admin exists |

## Default logins (after `npm run seed`)

All seeded accounts use the password `Difarm123`. You can log in with the email, username or phone.

| Email | Username | Role |
|-------|----------|------|
| `superadmin@difarm.com` | `superadmin` | Super Admin |
| `admin@difarm.com` | `admin` | Farm Admin |
| `manager@difarm.com` | `manager` | Farm Manager |

The seed also creates an active **Demo Farm** owned by the super admin. The admin and manager accounts start with no farm, so after logging in the admin is sent to `/register-farm`. Re-running the seed resets these accounts' passwords.

Without seeding, you can create the first super admin with `POST /api/v1/auth/register/super`. It only works while no super admin exists, unless `ALLOW_SUPER_REGISTER=true`.

## Roles

| Role | Access |
|------|--------|
| Super Admin | Every farm, plus an "all farms" view; activates accounts and newly registered farms |
| Farm Admin | Farms they own; creates managers and veterinarians for those farms |
| Farm Manager | Day-to-day records on the farms they are assigned to |
| Veterinarian | Cattle, vaccinations and inseminations on their assigned farm |

Farms registered by a farm admin, and the managers and veterinarians a farm admin creates, stay inactive until a super admin activates them. Anything the super admin creates is active immediately. After login, everyone except the super admin picks a farm on `/choose-farm`, and the dashboard then works on that farm.

## Pages

| Path | Page |
|------|------|
| `/` | Redirects to `/home` |
| `/home` | Landing page |
| `/login` | Login |
| `/choose-farm` | Farm selection after login |
| `/register-farm` | Farm onboarding for farm admins |
| `/account` | Dashboard overview |
| `/account/farms`, `/account/farms/[farmId]` | Farms |
| `/account/users`, `/account/users/detail/[userId]` | Users |
| `/account/cattle`, `/account/cattle/detail/[cattleId]` | Cattle |
| `/account/production`, `/account/production_totals`, `/account/production_transactions` | Production, totals and sales |
| `/account/stock`, `/account/stock_transactions` | Stock, suppliers and stock movements |
| `/account/health` | Vaccinations and inseminations |
| `/account/waste-logs` | Waste logs |
| `/account/activity-logs` | Activity logs |
| `/account/profile` | Profile |
| `/stock` | Stock overview |

Any other path redirects to `/home`.

## API

The base path is `/api/v1`, and calls need the header `Authorization: Bearer <token>` (the token comes from `POST /api/v1/auth/login`). Resources: `auth`, `users`, `farms`, `cattles`, `productions`, `production-totals`, `production-transaction`, `waste-logs`, `stocks`, `stock-transactions`, `suppliers`, `vaccinations`, `veterinarians`, `inserminations`, `activity-logs`. Routes are defined in `backend/src/router/`.

Health checks that don't touch the database: `/api/health` and `/api/v1/health`. Both report whether `DATABASE_URL`, `JWT_SECRET` and `JWT_VERIF_SECRET` are set.

## Scripts

| Script | Description |
|--------|-------------|
| `npm run dev` | Next.js with the API in-process |
| `npm run dev:all` | Next.js + standalone API |
| `npm run dev:api` | Standalone API only |
| `npm run setup` | Install all dependencies and generate the Prisma client |
| `npm run build` | `prisma generate` + `next build` (fails on type or lint errors) |
| `npm run start` | Serve the production build |
| `npm run lint` | ESLint |
| `npm run db:push` | Sync the Prisma schema to the database |
| `npm run db:generate` | Regenerate the Prisma client |
| `npm run seed` | Seed default users and the demo farm |

To type-check the frontend and backend together, run `npx tsc --noEmit`.

## Project layout

```
├── backend/          # Express API: routes → middleware → controllers → services
│   ├── server.ts     # standalone entry (dev:api)
│   └── src/createApp.ts   # builds the Express app (used by Next and server.ts)
├── prisma/           # schema.prisma + seed.ts
├── scripts/          # dev/setup helpers used by npm scripts
├── src/
│   ├── pages/        # Next.js routes (thin wrappers) + pages/api (API entry points)
│   ├── app/          # page components
│   ├── components/   # shared UI, dashboard layout, RoleGuard
│   ├── hooks/api/    # data hooks over the shared axios client
│   ├── lib/          # router-compat (react-router-style hooks on next/router)
│   ├── utils/        # permissions, farm selection, token storage
│   └── store/        # Redux (theme settings)
├── public/           # static assets
├── next.config.ts
└── vercel.json
```

## Deploying to Vercel

The app deploys as a single Next.js project; the API runs as Vercel functions.

1. Set `DATABASE_URL`, `JWT_SECRET` and `JWT_VERIF_SECRET` in **Vercel → Settings → Environment Variables**, plus the email variables if you need email. Do **not** set `NEXT_PUBLIC_SERVER_URL` to a localhost URL.
2. Create the tables by running `npm run db:push` locally against the production `DATABASE_URL`.
3. Deploy, then open `/api/v1/health` to confirm the variables are present.

On Vercel, uploaded files (e.g. vaccination scans) are written to `/tmp`, which is temporary storage, so they won't survive between function instances.

## Notes

- npm installs use `legacy-peer-deps` (set in `.npmrc`) because of peer-dependency conflicts in the UI libraries.
- Image optimization is disabled (`images.unoptimized`) so the existing `<img>` tags keep working.
