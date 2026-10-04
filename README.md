# IssueHub

IssueHub is a full-stack campus issue reporting app. Students can submit and track reports; administrators can review reports and update their status. The UI uses a charcoal palette with a restrained purple accent.

## Features

- Student signup and login with bcrypt-hashed passwords and JWTs in HTTP-only cookies.
- Session restoration with `GET /api/auth/me`; no application data or tokens are stored in `localStorage`.
- Student dashboard counts, report submission, report history, report details, and profile.
- Admin dashboard counts from stored reports, report filtering, status changes, deletion confirmation, and profile.
- Explainable duplicate-report warnings and confirmed issue groups; students can submit anyway, and admins review uncertain matches.
- Empty, loading, and error states; no seeded reports or invented dashboard metrics.
- Role checks in both frontend routing and backend middleware; signup always creates a student.

## Tech stack

- Frontend: React, Vite, React Router, Tailwind CSS, Axios.
- Backend: Node.js, Express, Mongoose, MongoDB, bcryptjs, JWT, HTTP-only cookies.

## Project structure

```text
IssueHub/
├── frontend/
│   └── src/
│       ├── components/       # Shared UI
│       ├── context/          # API-backed authentication state
│       ├── hooks/
│       ├── layouts/          # Separate student and admin layouts
│       ├── pages/
│       │   ├── public/
│       │   ├── student/
│       │   │   └── CampusProblems.jsx
│       │   └── admin/
│       ├── routes/
│       ├── services/         # Axios, auth and report API calls
│       └── utils/
├── backend/
│   └── src/
│       ├── config/
│       ├── controllers/
│       ├── middleware/
│       ├── models/
│       │   └── UrgencyVote.js
│       ├── routes/
│       ├── services/            # ImageKit uploads
│       └── utils/
├── package.json              # Root convenience scripts
└── README.md
```

## Requirements and setup

Use Node.js 20.19+ and npm. Start a local MongoDB server, or use a MongoDB Atlas URI you control.

1. Install dependencies:

   ```bash
   npm install
   cd frontend && npm install
   cd ../backend && npm install
   ```

2. Copy `frontend/.env.example` to `frontend/.env` and `backend/.env.example` to `backend/.env`.
3. Set `MONGO_URI` to your MongoDB connection string and replace `JWT_SECRET` with a random secret of at least 32 characters. Set `CLIENT_URL` to the frontend origin. Do not commit `.env` files.
4. Add your ImageKit private key as `IMAGEKIT_PRIVATE_KEY` in `backend/.env`. Keep it on the server only.
5. Run the app from the repository root:

   ```bash
   npm run dev
   ```

   Or start each part separately:

   ```bash
   npm run frontend
   npm run backend
   ```

The backend API defaults to port 3000. The backend intentionally refuses to start without a MongoDB URI and JWT secret. It does not silently switch to in-memory or mock data.

## Production deployment

The repository includes a Render Blueprint at `render.yaml` for the Express API and Vercel SPA fallback configuration at `frontend/vercel.json`.

1. In Render, create a Blueprint from this repository and enter the private `MONGO_URI`, `CLIENT_URL`, `IMAGEKIT_PRIVATE_KEY` (optional), and `ADMIN_SIGNUP_KEY` values when prompted. Render generates `JWT_SECRET`.
2. Deploy the frontend on Vercel with the project root set to `frontend`. Set `VITE_API_URL` to the deployed API origin plus `/api`, for example `https://issuehub-api.onrender.com/api`.
3. Set Render's `CLIENT_URL` to the exact Vercel production origin (for example, `https://issuehub.vercel.app`) and redeploy the API.
4. The Render Blueprint sets `AUTH_COOKIE_SAME_SITE=none` for cross-site cookie authentication; production cookies are secure and HTTP-only.

Never commit production secrets or put backend credentials in Vercel environment variables. A MongoDB Atlas database must be reachable from Render.

Issue photos are optional. The backend accepts one JPEG, PNG, or WebP image up to 5 MB, validates the image signature, uploads it to the existing reports folder in ImageKit, and stores the returned URL and ImageKit file ID with the report in MongoDB. Set `IMAGEKIT_PRIVATE_KEY` from your ImageKit account before using photo uploads; requests without a configured key receive a clear service-unavailable response. The key is never sent to the browser.

## Admin setup

Student signup always creates a student; it never accepts a role from the browser. Admins have separate `/admin/login` and `/admin/signup` pages. To enable protected admin signup, set a private, high-entropy `ADMIN_SIGNUP_KEY` in `backend/.env` and restart the backend. Do not put this key in the frontend or commit it. Admin signup validates this key on the server, hashes the password, creates an admin in the same `User` collection, and issues the normal HTTP-only auth cookie. If the key is unset, the endpoint stays disabled.

Alternatively, provision an administrator from the backend using `ADMIN_NAME`, `ADMIN_EMAIL`, and `ADMIN_PASSWORD` in `backend/.env` (use a unique password of at least 12 characters), then run:

```bash
npm run seed-admin
```

The seed command creates or updates that one administrator account in the configured database. Keep its credentials private.

## Authentication and cookies

The backend hashes passwords with bcryptjs. On login/signup it issues a signed JWT in an HTTP-only, same-site cookie. Axios sends cookies with `withCredentials`. The frontend calls `/api/auth/me` on startup and keeps the user only in React memory. Logout clears the cookie server-side.

For local development, use `localhost` consistently for both frontend and backend URLs. In production, use HTTPS; the auth cookie is marked secure when `NODE_ENV=production`.

## API routes

| Method | Route | Access | Purpose |
| --- | --- | --- | --- |
| POST | `/api/auth/signup` | Public | Create a student account and sign in |
| POST | `/api/auth/login` | Public | Sign in as a student or admin |
| POST | `/api/auth/admin/login` | Public | Sign in only when the credentials belong to an admin |
| POST | `/api/auth/admin/signup` | Signup key | Create an admin account and sign in |
| POST | `/api/auth/logout` | Public | Clear auth cookie |
| GET | `/api/auth/me` | Signed in | Restore current session |
| POST | `/api/reports` | Student | Submit a report |
| GET | `/api/reports/my` | Signed in | List the current student’s reports |
| GET | `/api/reports` | Signed in | List/filter campus-wide reports with vote count and current user vote |
| GET | `/api/reports/:id` | Signed in | Read a campus report (reporter email is not exposed to students) |
| POST | `/api/reports/:id/urgency` | Student | Mark a report urgent (one vote per student) |
| DELETE | `/api/reports/:id/urgency` | Student | Remove the current student’s urgency vote |
| GET | `/api/admin/dashboard` | Admin | Read real report counts and recent reports |
| GET | `/api/admin/insights` | Admin | Read database-derived recurrence, backlog, and community urgency insights |
| PATCH | `/api/admin/duplicate-groups/:id/confirm` | Admin | Confirm a proposed related-report group |
| PATCH | `/api/admin/duplicate-groups/:id/reject` | Admin | Reject a proposed related-report group |
| GET | `/api/admin/reports` | Admin | List/filter reports |
| GET | `/api/admin/reports/:id` | Admin | Read a report |
| PATCH | `/api/admin/reports/:id/status` | Admin | Update status |
| DELETE | `/api/admin/reports/:id` | Admin | Delete a report |
| GET | `/api/users/profile` | Signed in | Read profile |
| PUT | `/api/users/profile` | Signed in | Update name and optional student ID |
| GET | `/api/health` | Public | Basic API health check |

## Student and admin flows

- Students sign up, then land on their dashboard; they can report issues, browse all campus problems, search/filter/sort the shared feed, vote urgent once per report (and remove their vote), view their own reports, and open any campus report details.
- Admins can use the separate authority login/signup pages and land on the admin dashboard. Dashboard statistics, vote counts, recurring issue groups, long-pending problems, and concern flags are derived from MongoDB; admins can filter all reports, change status, or delete a report after confirmation.
- Report submissions are compared with active reports in the same category using normalized location, title, and description text. Scores of 80 or higher are grouped automatically; matches from 65 to 79 are proposed for admin review. A student can inspect similar reports and explicitly submit anyway. Grouping never deletes or merges the original reports.
- Server-side middleware enforces authentication, role, and report ownership; frontend route guards are only a navigation aid.
- Dashboard totals are calculated from actual MongoDB reports. A new database shows zeros and the appropriate empty states.
- Urgency votes live in a separate MongoDB collection with a unique `(report, user)` index; only authenticated students can add or remove their own votes. The official report priority remains independent of community votes.
- Insight grouping matches normalized title, category, and location. `COMMUNITY_URGENCY_THRESHOLD` (default `10`) and `LONG_PENDING_DAYS` (default `7`) configure the high-concern and long-pending labels.

## Validation

```bash
npm run build
npm run lint
```

The API cannot complete database-backed flows until a MongoDB instance is configured and available.
