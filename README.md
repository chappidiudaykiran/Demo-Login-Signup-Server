# GuidEx — Server (Backend API)

A secure REST API built with **Node.js**, **Express.js**, and **MongoDB** that handles user authentication with JWT tokens and bcrypt password hashing.

---

## 🛠️ Technologies Used

| Technology | Purpose |
|---|---|
| **Node.js** | JavaScript runtime for the server |
| **Express.js 4** | Web framework for building REST APIs |
| **MongoDB** | NoSQL database for storing user data |
| **Mongoose 8** | MongoDB ODM (Object Data Modeling) |
| **JSON Web Token (JWT)** | Stateless authentication tokens |
| **bcryptjs** | Password hashing (12 salt rounds) |
| **express-validator** | Server-side request validation |
| **dotenv** | Environment variable management |
| **cors** | Cross-Origin Resource Sharing |
| **nodemon** | Auto-restart server during development |

---

## 📁 Project Structure

```
server/
├── config/
│   └── db.js               # MongoDB connection setup
├── middleware/
│   └── auth.js              # JWT verification middleware
├── models/
│   └── User.js              # Mongoose User schema & password hashing
├── routes/
│   └── auth.js              # Auth API endpoints (signup, login, me)
├── server.js                # Express app entry point
├── vercel.json              # Vercel serverless deployment config
├── .env                     # Environment variables (not committed)
├── .gitignore
└── package.json
```

---

## 🔌 API Endpoints

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `POST` | `/api/auth/signup` | ❌ Public | Register a new user |
| `POST` | `/api/auth/login` | ❌ Public | Log in an existing user |
| `GET` | `/api/auth/me` | ✅ Protected | Get current user's profile |

### Request & Response Examples

**POST `/api/auth/signup`**
```json
// Request Body
{ "name": "John Doe", "email": "john@example.com", "password": "mySecurePass123" }

// Success Response (201)
{
  "message": "Account created successfully!",
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": { "id": "664a...", "name": "John Doe", "email": "john@example.com" }
}
```

**POST `/api/auth/login`**
```json
// Request Body
{ "email": "john@example.com", "password": "mySecurePass123" }

// Success Response (200)
{
  "message": "Login successful!",
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": { "id": "664a...", "name": "John Doe", "email": "john@example.com" }
}
```

**Error Response (400/401)**
```json
{
  "message": "Invalid email or password.",
  "errors": [{ "msg": "Please enter a valid email", "param": "email", "location": "body" }]
}
```

---

## 🔐 How Authentication Works

### Signup Flow

```
POST /api/auth/signup → express-validator validates fields →
Check duplicate email in MongoDB → Hash password (bcrypt, 12 salt rounds) →
Save user to DB → Generate JWT (7-day expiry) → Return token + user
```

1. **Server-side validation** (`express-validator`):
   - `name`: trimmed, 2–50 characters
   - `email`: valid email format, normalized to lowercase
   - `password`: minimum 6 characters
2. **Duplicate check** — `User.findOne({ email })` ensures no existing account.
3. **Password hashing** — Mongoose `pre('save')` hook uses `bcrypt.genSalt(12)` + `bcrypt.hash()`.
4. **JWT generated** — `jwt.sign({ id, email, name }, JWT_SECRET, { expiresIn: '7d' })`.
5. **Response** — `201` with `{ token, user, message }`.

### Login Flow

```
POST /api/auth/login → express-validator validates →
Find user by email → bcrypt.compare(password, hash) →
Generate JWT → Return token + user
```

1. **Validation** — email format and password not empty.
2. **User lookup** — `User.findOne({ email })`. Returns generic error if not found.
3. **Password comparison** — `user.comparePassword()` uses `bcrypt.compare()`.
4. **JWT generated** — same as signup.
5. **Response** — `200` with `{ token, user, message }`.

> **Security:** Login errors use a generic message ("Invalid email or password") for both wrong email and wrong password to prevent email enumeration.

### Token Verification (`GET /api/auth/me`)

1. `authMiddleware` extracts `Authorization: Bearer <token>` header.
2. Verifies token with `jwt.verify(token, JWT_SECRET)`.
3. Attaches decoded `{ id, email, name }` to `req.user`.
4. Returns user data with password field excluded (`.select('-password')`).

---

## ✅ Validation Rules

### express-validator (Route Level)

**Signup (`POST /api/auth/signup`):**

| Field | Rule | Error Message |
|---|---|---|
| `name` | `trim()`, 2–50 characters | `Name must be 2-50 characters` |
| `email` | `isEmail()`, `normalizeEmail()` | `Please enter a valid email` |
| `password` | Minimum 6 characters | `Password must be at least 6 characters` |
| `email` | Must not already exist in DB | `An account with this email already exists.` |

**Login (`POST /api/auth/login`):**

| Field | Rule | Error Message |
|---|---|---|
| `email` | `isEmail()`, `normalizeEmail()` | `Please enter a valid email` |
| `password` | `notEmpty()` | `Password is required` |
| `email` | Must exist in DB | `Invalid email or password.` |
| `password` | Must match stored hash | `Invalid email or password.` |

### Mongoose Schema (Database Level)

| Field | Type | Constraints |
|---|---|---|
| `name` | `String` | **Required**, trimmed, min 2, max 50 chars |
| `email` | `String` | **Required**, unique, trimmed, lowercase, regex validated |
| `password` | `String` | **Required**, min 6 chars, hashed before save |
| `createdAt` | `Date` | Auto-generated (`timestamps: true`) |
| `updatedAt` | `Date` | Auto-generated (`timestamps: true`) |

### JWT Middleware Validation

| Check | Response |
|---|---|
| `Authorization` header missing | `401` — `Access denied. No token provided.` |
| Header format not `Bearer <token>` | `401` — `Access denied. No token provided.` |
| Token invalid or tampered | `401` — `Invalid or expired token.` |
| Token expired (past 7 days) | `401` — `Invalid or expired token.` |

---

## 🔒 Security Features

- **Password hashing** — bcrypt with 12 salt rounds (never stored in plain text)
- **JWT expiry** — tokens auto-expire after 7 days
- **Input validation** — express-validator + Mongoose schema double validation
- **Generic error messages** — login errors don't reveal whether email or password was wrong
- **CORS configured** — only allowed frontend origins can access the API
- **Password excluded** — `/me` endpoint strips password using `.select('-password')`

---

## ⚙️ Environment Variables

Create a `.env` file in the server root:

```env
PORT=5000
MONGO_URI=mongodb+srv://<username>:<password>@cluster.mongodb.net/guidex
JWT_SECRET=your_super_secret_key_here
FRONTEND_URL=https://your-client-app.vercel.app
```

---

## 🚀 Getting Started

### Prerequisites
- Node.js (v18+)
- MongoDB Atlas account (or local MongoDB)

### Installation

```bash
npm install
```

### Running Locally

```bash
npm run dev     # Starts with nodemon on http://localhost:5000
```

### Deploy to Vercel

1. Push this repo to GitHub.
2. Import the repo on [vercel.com](https://vercel.com).
3. Add environment variables in Vercel dashboard:
   - `MONGO_URI`
   - `JWT_SECRET`
   - `FRONTEND_URL` (your deployed client URL)
4. Deploy — Vercel auto-detects `vercel.json` and serves `server.js` as a serverless function.
