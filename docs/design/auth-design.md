# MedMind — Auth Design

**Date:** June 24, 2026  
**Status:** Approved — Ready for Implementation  
**Approach:** Simple JWT + bcrypt. Minimal — auth is not the focus of this project.  

---

## 1. Overview

MedMind uses a straightforward email/password auth system. No OAuth, no social login, no refresh tokens. The goal is to securely identify users so their memories are private, without spending development time on auth complexity.

---

## 2. How It Works

### Registration Flow

```
User submits email + password + display_name
        │
        ▼
Backend validates:
  - Email format (regex check)
  - Email not already taken (DB lookup)
  - Password >= 8 characters
        │
        ▼
Hash password with bcrypt (12 rounds)
        │
        ▼
Insert into users table
        │
        ▼
Generate JWT token (contains user_id, expires in 7 days)
        │
        ▼
Return { user_id, email, display_name, token }
```

### Login Flow

```
User submits email + password
        │
        ▼
Look up user by email
        │
        ▼
Verify password against stored bcrypt hash
        │
        ▼
Update last_active_at timestamp
        │
        ▼
Generate JWT token (7-day expiry)
        │
        ▼
Return { user_id, email, display_name, token }
```

### Authenticated Requests

```
Frontend stores JWT in localStorage
        │
        ▼
Every API request includes header:
  Authorization: Bearer <token>
        │
        ▼
Backend middleware decodes JWT
  - Valid → extract user_id, proceed
  - Expired → return 401
  - Invalid → return 401
```

---

## 3. JWT Token Structure

```json
{
  "sub": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "exp": 1719792000,
  "iat": 1719187200
}
```

| Field | Purpose |
|---|---|
| `sub` | User ID (UUID) |
| `email` | User email (for convenience, not for auth) |
| `exp` | Expiration timestamp (7 days from issue) |
| `iat` | Issued-at timestamp |

**Signing algorithm:** HS256  
**Secret key:** Stored in `.env` as `JWT_SECRET`

---

## 4. Dependencies

```
bcrypt          — password hashing
python-jose     — JWT encode/decode (or PyJWT)
```

---

## 5. FastAPI Middleware

A reusable dependency that extracts the current user from the JWT:

```python
# Pseudocode — actual implementation in code

async def get_current_user(authorization: str = Header()) -> User:
    token = authorization.replace("Bearer ", "")
    payload = jwt.decode(token, JWT_SECRET, algorithms=["HS256"])
    user = await db.get_user(payload["sub"])
    if not user:
        raise HTTPException(401, "User not found")
    return user
```

Used on every protected route:

```python
@app.get("/api/memories")
async def get_memories(user: User = Depends(get_current_user)):
    ...
```

---

## 6. Security Notes

| Concern | Approach |
|---|---|
| **Password storage** | bcrypt with 12 rounds. Never store plaintext. |
| **JWT secret** | Random 256-bit key in `.env`. Never committed to git. |
| **Token storage (frontend)** | `localStorage`. Not ideal (XSS risk), but acceptable for hackathon scope. Production would use httpOnly cookies. |
| **CORS** | Allow frontend origin only (`http://localhost:3000` locally). |
| **Rate limiting** | Not implemented for hackathon. Production would add rate limiting on `/auth/login` to prevent brute force. |
| **Email verification** | Not implemented. Registration is immediate. |
| **Password reset** | Not implemented. Out of scope for hackathon. |

---

## 7. What We're NOT Building (And Why)

| Feature | Why Not |
|---|---|
| OAuth / Google login | Extra complexity, API keys, redirect flows. Not worth the time. |
| Refresh tokens | 7-day JWT is plenty for a hackathon demo. |
| Email verification | No email service to set up. Not the focus. |
| Password reset | Requires email sending infrastructure. Skip. |
| Session revocation | Would need a token blacklist. Overkill. |
| Multi-factor auth | Way out of scope. |

---

## 8. Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| **Token expiry** | 7 days | Long enough that judges won't get logged out during testing. Short enough to be reasonable. |
| **Storage** | localStorage | Simpler than cookies for a SPA. Acceptable risk for hackathon. |
| **Hash algorithm** | bcrypt (12 rounds) | Industry standard. 12 rounds balances security and speed. |
| **No refresh tokens** | Correct | Eliminates token rotation complexity. User just logs in again after 7 days. |
