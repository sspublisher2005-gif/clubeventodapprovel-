# Campus OD

College event and On-Duty request workflow. The interface is plain HTML, CSS, and JavaScript; the API is Node.js, Express, and MongoDB.

## Run the backend

From `back-end`, copy `.env.example` to `.env`, set `MONGODB_URI` and a unique `JWT_SECRET` of at least 32 characters, then run:

```powershell
npm install
npm run create-admin
npm run dev
```

The API listens at `http://localhost:5000`. The first admin creates student and staff accounts. Reviewer department and class assignments are used by the API to route requests.

## Run the website

Serve this project folder from a local web server (for example, VS Code Live Server). The default API address in `js/api.js` is `http://localhost:5000/api`; the API's `CLIENT_ORIGIN` must match the web server origin. Set `window.CAMPUS_OD_API` before `js/api.js` if the API is hosted elsewhere.

Sign in with an account created by the admin to use live backend data. The separate **Preview** option uses sample data saved only in the browser; it does not create real requests or verify an identity.

## Workflows

- **Inter-college:** Coordinator publishes event → eligible student registers → student uploads certificate and missed sessions → Class In-charge → Department Coordinator → Director → DEO records attendance.
- **Outside-college:** Student submits event details, missed sessions, and registration proof → Class In-charge → Department Coordinator → student uploads certificate to the same request → Director → DEO records attendance.

Rejections stop the review chain. Correction returns keep the request open and preserve earlier decisions. Attendance becomes recorded only after the DEO saves the selected sessions.

## Project map

- `index.html`, `styles.css`, `js/`: website and browser API client.
- `back-end/src/`: API server, MongoDB models, access control, uploads, and workflow rules.
- `back-end/scripts/create-admin.js`: creates the initial administrator.
- `back-end/README.md`: endpoint reference, payload formats, and deployment notes.
