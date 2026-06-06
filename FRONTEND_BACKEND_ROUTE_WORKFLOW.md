# MediSense Frontend ↔ Backend Route Workflow

This document explains how navigation routes in the React frontend map to Express API routes in the backend, and how data flows through the system.

## 1) Base setup

- Frontend routes are defined in `client/src/App.jsx`.
- Backend route prefixes are mounted in `server/server.js`.
- Frontend API base URL comes from `VITE_API_URL` (client env).
- Backend secrets/config are read from `server/.env` (`DB_URL`, `JWT_SECRET`, etc.).

---

## 2) Frontend page routes (React Router)

Defined in `client/src/App.jsx`:

- `/` → `StartPage`
- `/home` → `Home`
- `/signin` → `Signin`
- `/signup` → `Signup`
- `/superadmin` → `SuperAdminLogin`
- `/superadmin/dashboard` → `SuperAdminDashboard`
- `/prescriptions` → `Prescriptions`
- `/medicines` → `Medicines`
- `/track-medicines` → `TrackMedicines`
- `/med-database` → `MedDatabase`
- `/profile` → `Profile`
- `/reports` → `Reports`

Header visibility is controlled by the `Layout` component (hidden on `/`, `/signin`, `/signup`, `/superadmin`).

---

## 3) Backend API groups (Express)

Mounted in `server/server.js`:

- `/api/auth` → `routes/auth.js`
- `/api/admin` → `routes/admin.js`
- `/api/prescriptions` → `routes/prescription.js`
- `/api/notifications` → `routes/notification.js`
- `/api/medicines` → `routes/medicine.js`
- `/api/med-database` → `routes/medDatabase.js`
- `/api/profile` → `routes/profile.js`
- `/api/ocr` → `routes/ocr.js`

Also available:

- `GET /` basic server status
- `GET /health` health + DB state

---

## 4) Page-to-API mapping

## Authentication screens

### `/signup` (Signup component)
- `POST /api/auth/signup`
- On success, token + user are saved in local storage, then navigates to `/home`.

### `/signin` (Signin component)
- `POST /api/auth/signin`
- On success, token + user are saved in local storage, then navigates to `/home`.

---

## Dashboard/home

### `/home` (Home component)
- Reads logged-in user from local storage.
- If missing, redirects to `/signin`.
- Provides navigation entry points to prescriptions, medicine database, profile, etc.

---

## Prescription workflow

### `/prescriptions` (Prescriptions component)
Main API calls:

- `GET /api/prescriptions/patient-prescriptions` (patient’s own prescriptions)
- `GET /api/prescriptions/search-patients?query=...` (organisation search)
- `GET /api/prescriptions/patient/:patientId` (organisation view of one patient)
- `POST /api/prescriptions/create` (organisation creates prescription + medicine records)
- `GET /api/med-database?search=...` (medicine search while creating prescription)
- `POST /api/ocr/extract` (OCR extraction from uploaded prescription image)

Backend behavior highlights (`routes/prescription.js`):
- Validates user token.
- Restricts creation/search-for-patients routes to `organisation` users.
- Creates one `Prescription` plus multiple linked `Medicine` docs.
- Creates initial notifications for newly prescribed medicines.

---

## Medicine tracking

### `/medicines` (Medicines component)
- `GET /api/medicines/today`
- `GET /api/medicines/stats`
- `PUT /api/medicines/:medicineId/take`
- `PUT /api/medicines/:medicineId/untake`

### `/track-medicines` (TrackMedicines component)
- `GET /api/medicines/my-medicines`
- `DELETE /api/medicines/:id`

### `/reports` (Reports component)
- `GET /api/medicines/medicine-counts`

Backend medicine route includes reminder/testing endpoints too (cron/notification support):
- `POST /api/medicines/trigger-reset`
- `POST /api/medicines/trigger-reminder`
- `POST /api/medicines/trigger-missed-check`
- `POST /api/medicines/trigger-expiry-check`
- `POST /api/medicines/trigger-emergency-contact-check`

---

## Medicine database

### `/med-database` (MedDatabase component)
- `GET /api/med-database`
- `POST /api/med-database` (auth + superadmin middleware in backend)
- `DELETE /api/med-database/:id`

Additional backend routes available:
- `GET /api/med-database/:id`
- `PUT /api/med-database/:id`
- `GET /api/med-database/categories/list`
- `GET /api/med-database/manufacturers/list`

---

## Profile

### `/profile` (Profile component)
- `GET /api/profile`
- `PUT /api/profile`
- `DELETE /api/profile`

All are protected routes requiring bearer token.

---

## Super admin

### `/superadmin` and `/superadmin/dashboard`
Used for user management UI:
- `GET /api/admin/users`
- `PUT /api/admin/users/:userId/usertype`

---

## 5) Notification flow

Notification APIs (`routes/notification.js`):
- `GET /api/notifications`
- `GET /api/notifications/unread-count`
- `GET /api/notifications/today`
- `GET /api/notifications/pending`
- `PUT /api/notifications/:id/read`
- `PUT /api/notifications/:id/sent`
- `PUT /api/notifications/read-all`
- `DELETE /api/notifications/:id`

Background behavior:
- Cron jobs are initialized from `services/cronService` once DB connects.
- Notification worker (`services/notificationWorker`) sends pending SMS notifications and marks them sent.

---

## 6) End-to-end request lifecycle

1. User opens frontend route (React Router).
2. Component checks local token/user where required.
3. Component calls `${VITE_API_URL}/api/...` and sends Authorization bearer token for protected routes.
4. Express route middleware validates JWT and role (where needed).
5. Route handler reads/writes MongoDB via Mongoose models.
6. Response JSON returns to frontend.
7. Frontend updates UI state (lists, badges, adherence statuses, etc.).

---

## 7) High-level role-based access pattern

- Public (no token): signup/signin, some med-database reads.
- Authenticated user: profile, medicine schedule/tracking, patient prescription view, notifications.
- Organisation user: create prescriptions, search patients, org patient prescription views.
- Super admin/admin endpoints: user-type management endpoints under `/api/admin`.
