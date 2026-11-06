# Club Event OD Approval API

REST backend for the college event and On-Duty approval application. Built with Node.js, Express, Mongoose, and MongoDB.

## Requirements and setup

- Node.js 20 or newer
- MongoDB running locally, or a MongoDB Atlas connection string

```powershell
Copy-Item .env.example .env
# Set MONGODB_URI, JWT_SECRET (32+ characters), and CLIENT_ORIGIN in .env
npm install
npm run create-admin
npm run dev
```

The API starts on `http://localhost:5000`. Health check: `GET /api/health`.

The first admin is created only by the local setup script. The admin creates all student and staff accounts through the authenticated user API; public signup cannot grant or choose a privileged role. Create one account per staff reviewer assignment. A Class In-charge account must have a department and `classSection` matching the students they review. Reviewer assignment currently selects one active Class In-charge for a department/class and one Department Coordinator per department. Director accounts are department-scoped. DEOs can access eligible attendance queue items across departments.

## Authentication and response format

Login with `POST /api/auth/login`. Send the returned token as `Authorization: Bearer <token>` on protected requests. Responses use `{ "data": ... }`; errors use `{ "error": { "message": ... } }`. Dates are ISO-8601 values. IDs are MongoDB ObjectId strings.

## Main endpoints

| Method | Path | Role | Purpose |
|---|---|---|---|
| POST | `/api/auth/login` | Public | Login |
| GET | `/api/auth/me` | Any signed-in user | Current account |
| GET, POST | `/api/auth/users` | Admin | List/create accounts |
| PATCH | `/api/auth/users/:id` | Admin | Update account or deactivate it (`active:false`) |
| GET, POST | `/api/events` | Signed-in / Event Coordinator or Admin | List eligible events / create event |
| GET, PATCH, DELETE | `/api/events/:id` | Scoped / event owner or Admin | Read/update/delete event; events linked to OD requests cannot be deleted |
| POST | `/api/events/:id/register` | Student | Register and create the linked OD request |
| GET | `/api/events/:id/requests` | Event owner, Admin, DEO | View registrations and request progress |
| GET | `/api/requests` | Any signed-in user | Role-scoped request list; reviewers use `?queue=true` for their pending queue |
| POST | `/api/requests/outside` | Student | Submit outside-college request plus registration proof |
| GET, PATCH | `/api/requests/:id` | Scoped / request owner | View request / edit and resubmit a returned request |
| POST | `/api/requests/:id/reviews` | Class In-charge, Department Coordinator, Director | Approve, reject, or (outside-college only) return for correction |
| POST | `/api/requests/:id/files` | Student owner | Upload certificate or permitted replacement registration proof |
| GET | `/api/requests/:id/files/:fileId` | Scoped | Download private evidence |
| POST | `/api/requests/:id/attendance` | DEO | Record attendance after all required approvals |
| GET | `/api/requests/attendance` | Scoped | View recorded attendance |

### Request payloads

Create an outside-college request with `multipart/form-data`. Include a JSON field named `request` and a file field named `file`:

```json
{
  "eventName": "Regional Robotics Meet",
  "organizer": "Example Institute",
  "startAt": "2026-11-15T00:00:00.000Z",
  "endAt": "2026-11-15T23:59:00.000Z",
  "venue": "Example city",
  "isOnline": false,
  "missedSessions": [
    { "date": "2026-11-15T00:00:00.000Z", "courseCode": "CS201", "courseName": "Algorithms", "period": "1" }
  ]
}
```

For certificate upload, send `kind=certificate`, `file`, and optional `missedSessions` as JSON text. Inter-college and outside-college certificates are accepted only in their certificate-waiting stage. Allowed uploads are PDF, JPEG, PNG, and WEBP up to `MAX_UPLOAD_MB`; files are stored outside public web paths and served only through the authenticated request file endpoint.

Review JSON:

```json
{ "action": "approved" }
{ "action": "rejected", "comment": "Reason visible to the student" }
{ "action": "returned_for_correction", "comment": "Upload a clearer registration confirmation" }
```

Rejection and correction require a comment. Correction return is limited to outside-college requests and the Class In-charge or Department Coordinator stages.

DEO attendance JSON:

```json
{ "sessionIds": ["<missed-session-object-id>"], "note": "Verified against the certificate" }
```

## Workflow rules enforced by the API

- Inter-college: event registration → certificate → Class In-charge → Department Coordinator → Director → DEO → attendance record.
- Outside-college: registration proof → Class In-charge → Department Coordinator → certificate → Director → DEO → attendance record.
- The outside-college certificate remains attached to its original request. It is not a second request.
- Reviewers can act only at their current stage and within their assigned department/class. Rejected requests stop. Returned requests retain prior decisions and go back to the reviewer who returned them.
- Attendance cannot be recorded before the request reaches `pending_deo`. Saving attendance changes the request to `attendance_recorded` and records the DEO, time, and selected sessions.
- Each request stores an append-only application history for submissions, uploads, reviewer decisions, resubmissions, and attendance recording. Decision and history entries are not exposed to client-side update operations.
- Student and role-scoped request retrieval prevents users from reading unrelated requests. Event Coordinators can see only events and requests they own.

## Frontend connection

Set `CLIENT_ORIGIN` to the frontend origin (for example `http://localhost:5173`). The browser should send the JWT bearer token on protected API calls. Use `FormData` for uploads; do not set its `Content-Type` header manually because the browser must add the multipart boundary.

## Deployment notes

Use a managed MongoDB deployment or replica set, HTTPS, a strong unique JWT secret, a persistent private upload volume or object storage, and a backup/retention plan. Restrict the API origin to the deployed frontend. The local-disk upload adapter is for development/single-server deployments; multiple backend instances should use private object storage with short-lived authorized downloads. Connect to the college attendance/SIS system only after its integration contract and attendance policies are defined.
