# Finance Data Processing & Access Control Backend

A clean, well-structured REST API backend for a finance dashboard system with role-based access control, financial records management, and analytics.

---

## Tech Stack

| Layer | Choice | Reason |
|---|---|---|
| Runtime | Node.js (v18+) | Fast I/O, rich ecosystem |
| Framework | Express.js | Lightweight, composable |
| Database | sql.js (SQLite in-memory) | Zero setup, portable, full SQL |
| Validation | Zod | Type-safe schema validation |
| Auth | JWT (jsonwebtoken) | Stateless, simple to verify |
| Passwords | bcryptjs | Secure hashing |

---

## Project Structure

```
finance-backend/
├── src/
│   ├── app.js                  # Entry point, route mounting, middleware
│   ├── models/
│   │   └── database.js         # DB init, schema, query/run helpers
│   ├── routes/
│   │   ├── auth.js             # Login, register
│   │   ├── users.js            # User management (admin)
│   │   ├── records.js          # Financial records CRUD
│   │   └── dashboard.js        # Analytics & summary endpoints
│   └── middleware/
│       ├── auth.js             # JWT authentication + role authorization
│       └── errors.js           # 404 + global error handler
├── package.json
└── README.md
```

---

## Setup & Run

```bash
# Install dependencies
npm install

# Start the server
npm start
# → Running at http://localhost:3000
```

No environment variables required. A default admin account and 20 seeded financial records are created automatically on first run.

**Default Admin:**
- Email: `admin@finance.dev`
- Password: `admin123`

---

## Roles & Permissions

| Action | Viewer | Analyst | Admin |
|---|:---:|:---:|:---:|
| Login / Register | ✅ | ✅ | ✅ |
| View own profile | ✅ | ✅ | ✅ |
| View financial records | ✅ | ✅ | ✅ |
| View dashboard summary | ❌ | ✅ | ✅ |
| Create financial records | ❌ | ✅ | ✅ |
| Update financial records | ❌ | ❌ | ✅ |
| Delete financial records | ❌ | ❌ | ✅ |
| Manage users | ❌ | ❌ | ✅ |

Access control is enforced in middleware via the `authorize(...roles)` function. Every protected route specifies required roles explicitly.

---

## API Reference

### Authentication

#### `POST /auth/login`
```json
Request:  { "email": "admin@finance.dev", "password": "admin123" }
Response: { "token": "...", "user": { "id", "name", "email", "role" } }
```

#### `POST /auth/register`
```json
Request:  { "name": "Jane", "email": "jane@co.com", "password": "pass123", "role": "viewer" }
Response: { "token": "...", "user": { ... } }
```

> All subsequent requests require: `Authorization: Bearer <token>`

---

### Users `(Admin only)`

| Method | Path | Description |
|---|---|---|
| `GET` | `/users` | List all users (with role/status filter, pagination) |
| `GET` | `/users/me` | Get own profile (any authenticated user) |
| `GET` | `/users/:id` | Get user by ID |
| `PATCH` | `/users/:id` | Update name, role, or status |
| `DELETE` | `/users/:id` | Deactivate user (soft delete) |

**Query params for `GET /users`:** `role`, `status`, `page`, `limit`

---

### Financial Records

| Method | Path | Role Required |
|---|---|---|
| `GET` | `/records` | viewer+ |
| `GET` | `/records/:id` | viewer+ |
| `POST` | `/records` | analyst+ |
| `PATCH` | `/records/:id` | admin |
| `DELETE` | `/records/:id` | admin (soft delete) |

**Query params for `GET /records`:**
- `type` — `income` or `expense`
- `category` — partial match
- `date_from`, `date_to` — `YYYY-MM-DD` range
- `search` — searches notes and category
- `page`, `limit` — pagination

**Record body:**
```json
{
  "amount": 1500.00,
  "type": "income",
  "category": "Salary",
  "date": "2026-04-01",
  "notes": "April salary"
}
```

---

### Dashboard `(Analyst & Admin)`

| Method | Path | Description |
|---|---|---|
| `GET` | `/dashboard/summary` | Total income, expenses, net balance |
| `GET` | `/dashboard/categories` | Category-wise totals, filterable by type |
| `GET` | `/dashboard/trends` | Monthly income vs expense (last N months) |
| `GET` | `/dashboard/weekly` | Week-over-week trends (last 12 weeks) |
| `GET` | `/dashboard/recent` | Latest N transactions |

**`/dashboard/summary` response:**
```json
{
  "total_income": 20296.95,
  "total_expenses": 26683.69,
  "net_balance": -6386.74,
  "income_count": 10,
  "expense_count": 10
}
```

All dashboard endpoints accept optional `date_from` and `date_to` filters.

---

## Design Decisions & Assumptions

### Database
- **sql.js** (in-memory SQLite) was chosen for zero-config portability. Data resets on server restart. In production, this would be replaced with persistent SQLite (better-sqlite3) or PostgreSQL.
- Schema uses `deleted_at` column for soft deletes on both records and users (users set to `inactive` instead).

### Authentication
- JWT tokens expire in **8 hours**. No refresh token mechanism (can be added).
- Passwords are hashed with **bcrypt (cost=10)**.
- The `/auth/register` endpoint is open (no auth required) for demo convenience. In production, self-registration could be restricted or require email verification.

### Access Control
- Authorization is layered: `authenticate` verifies the token; `authorize(...roles)` checks role.
- Inactive users are rejected at the `authenticate` step, not just at login.
- Analysts can *create* records but not *modify* or *delete* them — reflecting a "submit only" analyst workflow assumption.

### Validation
- All inputs are validated with **Zod** schemas before any DB operation.
- Dates must be `YYYY-MM-DD` format.
- Amount must be a positive number.
- Category and notes have length limits.

### Soft Deletes
- Financial records are never hard-deleted; a `deleted_at` timestamp is set instead.
- All queries filter `WHERE deleted_at IS NULL`.

### Seeded Data
- On first boot, 20 financial records (10 income, 10 expense) across random categories and the past 90 days are seeded for demo/testing.

---

## Error Responses

All errors follow a consistent shape:
```json
{ "error": "Human-readable message", "details": { ... } }
```

| Status | Meaning |
|---|---|
| `400` | Validation error or bad request |
| `401` | Missing or invalid JWT |
| `403` | Valid user but insufficient role |
| `404` | Resource not found |
| `409` | Conflict (e.g., duplicate email) |
| `500` | Internal server error |

---

## Tradeoffs & What Would Change in Production

| Aspect | Current | Production |
|---|---|---|
| Database | in-memory sql.js | PostgreSQL or persistent SQLite |
| Auth secrets | hardcoded fallback | Environment variable, rotated regularly |
| Register | open | Admin-invite or email verification |
| Pagination | offset-based | Cursor-based for large datasets |
| Rate limiting | none | express-rate-limit or API gateway |
| Tests | none | Jest with supertest integration tests |
| Logging | console.error | Structured logging (pino/winston) |
| HTTPS | no | TLS termination at reverse proxy |
