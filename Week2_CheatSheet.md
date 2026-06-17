# Week 2 — Cheatsheet
**Internship Week 2**

---

## Day 1 — MVC
`Request → Controller → Model (DB/logic) → View (response)`

- **Model** → data + business logic. No UI knowledge.
- **View** → what user sees. No DB knowledge.
- **Controller** → middleman. Thin — logic goes in services, not here.

---

## Day 1 — Node.js Project Structure
```
src/
├── controllers/   # handle req/res
├── routes/        # URL → controller
├── models/        # DB schemas
├── middleware/    # auth, logging, errors
├── services/      # actual business logic
└── utils/         # helpers
app.js             # express setup
server.js          # entry point
```

---

## Day 1 — Middleware
Pipeline that runs before the route handler. Must call `next()` or request hangs.

```js
// Global — runs on every request
app.use(express.json());
app.use(authMiddleware);

// Route-specific — runs only here
router.get('/admin', checkAdmin, handler);

// Custom
function myMiddleware(req, res, next) {
    // do stuff
    next();
}
```
- Global → logging, CORS, body parsing, auth
- Route-specific → role checks, per-endpoint validation

---

## Day 2 — Async Patterns
I/O is slow → don't block the thread.

```js
// Callbacks (old, messy)
fs.readFile('f.txt', (err, data) => { ... });

// Promises
fetch(url).then(r => r.json()).catch(console.error);

// async/await (use this)
async function get() {
    try {
        const data = await fetch(url).then(r => r.json());
        return data;
    } catch (err) { console.error(err); }
}
```
Always `try/catch` around `await`. Unhandled rejections crash the app.

---

## Day 2 — Microservices vs Monolith
| | Monolith | Microservices |
|---|---|---|
| Deploy | One unit | Each service independent |
| Scale | Whole app | Only the bottleneck |
| Debug | Easy | Hard (cross-service tracing) |
| Start with | ✅ Yes | When monolith hits limits |

Split into services when a specific part becomes the bottleneck. Put an API Gateway in front.

---

## Day 2 — Logging (winston)
Not `console.log`. Levels: `DEBUG → INFO → WARN → ERROR → FATAL`

```js
const logger = winston.createLogger({
    level: 'info',
    transports: [new winston.transports.Console(),
                 new winston.transports.File({ filename: 'app.log' })]
});
logger.info('Server started');
logger.error('DB failed', { err });
```
Log what happened, not what function ran. `"User 42 login failed"` > `"entering login fn"`.

---

## Day 3 — Auth basics
- **Authentication** → who are you? (login)
- **Authorization** → what can you do? (permissions)

**Session** → state on server, DB lookup every request. Good for web apps.
**JWT** → stateless, client holds token, server just verifies signature. Good for APIs.

---

## Day 3 — JWT
`header.payload.signature` — three base64 parts, dot-separated.

```js
jwt.sign({ userId: 42, role: 'admin' }, SECRET, { expiresIn: '1h' });
jwt.verify(token, SECRET);  // throws if invalid/expired
```
Payload is **NOT encrypted** — don't put passwords in it.

---

## Day 3 — OAuth 2.0
"Login with Google" flow — I don't handle passwords, Google does.
```
App → redirect to Google → user approves
Google → sends auth code → App exchanges for access token
App → uses token to get user info
```

---

## Day 3 — RBAC & SSO
**RBAC** → assign roles to users, permissions to roles. Never direct user → permission.
```js
function checkRole(role) {
    return (req, res, next) => {
        if (req.user.role !== role) return res.status(403).json({ error: 'Forbidden' });
        next();
    };
}
```
`user` → read | `editor` → read+write | `admin` → full

**SSO** → login once, access all company apps. Central IdP handles auth, all apps trust it.

---

## Day 3 — API Design
```
GET    /users          list
GET    /users/:id      get one
POST   /users          create
PUT    /users/:id      full update
PATCH  /users/:id      partial update
DELETE /users/:id      delete
```
- **Pagination** → `?page=2&limit=20` (offset) or `?cursor=xyz` (better for scale)
- **Filtering** → `?category=shoes&sort=price&order=asc`
- **Rate limiting** → hard cap → 429. Throttling → slow down + queue.
- **GraphQL vs REST** → REST = fixed endpoints. GraphQL = one endpoint, client picks fields. Use GraphQL when clients need flexible/nested queries.

---

## Quick Ref
```
MVC:           Request → Controller → Model → View
Middleware:    global vs route-specific | always next()
Async:         callbacks → promises → async/await | try/catch always
Microservices: monolith first → split at bottleneck | API Gateway in front
Auth:          authn = identity | authz = permissions
JWT:           header.payload.sig | stateless | payload NOT encrypted
OAuth:         auth code flow → access token → user info
RBAC:          roles → permissions, not users → permissions
API:           REST verbs | paginate | rate limit | GraphQL for flexibility
```
