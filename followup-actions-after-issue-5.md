# Follow-up Actions After Issue #5

This document contains unresolved code review remarks and suggestions from the Admin Mode implementation (Issue #5). These items are organized by subject matter for future improvement.

---

## Security

### 1. Password Storage - Plaintext Credentials
**File:** `src/teachers.json:3-5`

**Issue:** Storing passwords in plaintext is a security vulnerability. Teacher credentials should be hashed using a secure algorithm like bcrypt, argon2, or PBKDF2.

**Recommendation:**
```python
from passlib.context import CryptContext
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
# Store hashed passwords in teachers.json
# Verify with: pwd_context.verify(plain_password, hashed_password)
```

### 2. Timing Attack Vulnerability
**File:** `src/app.py:111-112`

**Issue:** The password comparison uses simple string equality which is vulnerable to timing attacks. An attacker could potentially determine valid usernames by measuring response times.

**Recommendation:**
```python
import secrets
if teacher_credentials.get(req.username) and secrets.compare_digest(
    teacher_credentials.get(req.username), req.password
):
```

### 3. XSS Vulnerability - Token Storage
**File:** `src/static/app.js:249`

**Issue:** Storing the authentication token in localStorage is vulnerable to XSS attacks. If an attacker injects malicious JavaScript, they can steal the token and impersonate a teacher.

**Recommendation:**
```python
# Backend change:
from fastapi import Response
@app.post("/login")
def login(req: LoginRequest, response: Response):
    # ... validate credentials ...
    response.set_cookie(key="auth_token", value=token, httponly=True, secure=True, samesite="strict")
```
Frontend: Remove localStorage and let the browser handle cookies automatically.

### 4. Session Management Issues
**File:** `src/app.py:91-92`

**Issue:** The in-memory session store will lose all active sessions when the application restarts, forcing all logged-in teachers to re-authenticate. Additionally, sessions never expire, which is a security risk.

**Recommendations:**
1. Implement session expiration (e.g., timeout after inactivity)
2. Use persistent session storage (e.g., Redis, database) for production
3. Add session cleanup mechanism to prevent memory leaks

**Example:**
```python
from datetime import datetime, timedelta
sessions = {}  # token -> {"username": str, "expires": datetime}
```

---

## Input Validation

### 5. Email Validation - Signup Endpoint
**File:** `src/app.py:148`

**Issue:** The email parameter should be validated to ensure it's a properly formatted email address. Currently, any string can be passed.

**Recommendation:**
```python
from pydantic import EmailStr

@app.post("/activities/{activity_name}/signup")
def signup_for_activity(activity_name: str, email: EmailStr, authorization: str | None = Header(default=None)):
```

Or manual validation:
```python
import re
email_pattern = re.compile(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$')
if not email_pattern.match(email):
    raise HTTPException(status_code=400, detail="Invalid email format")
```

### 6. Email Validation - Unregister Endpoint
**File:** `src/app.py:171`

**Issue:** Same as above - email parameter needs validation for the unregister endpoint.

**Recommendation:** Apply the same email validation as described in item #5.

---

## Frontend UX & Error Handling

### 7. Token Validation on Page Load
**File:** `src/static/app.js:13-15`

**Issue:** The authentication state is only checked by verifying if a token exists in localStorage, but doesn't validate if the token is still valid on the server. If a token expires or is invalidated (e.g., after server restart), the UI will still show the user as authenticated.

**Recommendation:**
```javascript
async function validateAuthToken() {
    const token = localStorage.getItem("authToken");
    if (!token) return false;
    
    try {
        const resp = await fetch("/auth/status", {
            headers: { Authorization: `Bearer ${token}` }
        });
        const result = await resp.json();
        if (!result.authenticated) {
            localStorage.removeItem("authToken");
            return false;
        }
        return true;
    } catch {
        return false;
    }
}

// Call this on page load and update UI accordingly
```

### 8. 401 Error Handling - Signup
**File:** `src/static/app.js:145-152`

**Issue:** When an unauthenticated user triggers signup, there's no specific handling for 401 Unauthorized responses to prompt the user to login with a clearer message.

**Recommendation:**
```javascript
if (response.status === 401) {
    messageDiv.textContent = "Please login as a teacher to register students";
    messageDiv.className = "error";
    loginModal.classList.remove("hidden");
} else if (response.ok) {
    // ... existing success handling
```

### 9. 401 Error Handling - Unregister
**File:** `src/static/app.js:101-108`

**Issue:** Same as above - when an unauthenticated user triggers unregister, there's no specific handling for 401 responses.

**Recommendation:**
```javascript
if (response.status === 401) {
    messageDiv.textContent = "Please login as a teacher to unregister students";
    messageDiv.className = "error";
    loginModal.classList.remove("hidden");
} else if (response.ok) {
    // ... existing success handling
```

### 10. Inefficient DOM Queries
**File:** `src/static/app.js:17-27`

**Issue:** The `updateUIAuth()` function queries all delete buttons using `querySelectorAll()` every time it's called, even when there might be no delete buttons in the DOM yet.

**Recommendation:**
```javascript
function updateUIAuth() {
    if (isAuthenticated()) {
        userIcon.textContent = "Logout";
        signupContainer.classList.remove("hidden");
        const deleteButtons = document.querySelectorAll(".delete-btn");
        if (deleteButtons.length > 0) {
            deleteButtons.forEach((btn) => btn.classList.remove("hidden"));
        }
    } else {
        userIcon.textContent = "👤";
        signupContainer.classList.add("hidden");
        const deleteButtons = document.querySelectorAll(".delete-btn");
        if (deleteButtons.length > 0) {
            deleteButtons.forEach((btn) => btn.classList.add("hidden"));
        }
    }
}
```

---

## CSS & Styling

### 11. Secondary Button Styling
**File:** `src/static/styles.css:72-78`

**Issue:** The `.secondary` button style is defined but lacks important base button properties. It should inherit or redefine button properties to ensure consistent styling.

**Recommendation:**
```css
.secondary {
    background: #757575;
    color: white;
    border: none;
    padding: 10px 15px;
    font-size: 16px;
    border-radius: 5px;
    cursor: pointer;
    transition: background-color 0.2s;
}
```

### 12. Missing Branding Class Definition
**File:** `src/static/styles.css:18-28`

**Issue:** The `.branding` class is used in the HTML but has no corresponding CSS definition. While existing styling may work, it's better to explicitly define the class.

**Recommendation:**
```css
header .branding {
    flex: 1;
}
```

---

## Code Quality & Maintainability

### 13. Unused Return Value - Signup
**File:** `src/app.py:150`

**Issue:** The function returns the username from `require_teacher()` but never uses it. Either remove the return value or use it for logging/auditing purposes.

**Recommendation:**
```python
# Option 1: Don't return username
def require_teacher(auth_header: str | None) -> None:

# Option 2: Use it for logging
teacher_username = require_teacher(authorization)
# Log: f"Teacher {teacher_username} registered {email} for {activity_name}"
```

### 14. Unused Return Value - Unregister
**File:** `src/app.py:173`

**Issue:** Same as above - the username from `require_teacher()` is returned but never used.

**Recommendation:** Apply the same solution as described in item #13.

---

## Summary

**Total unresolved items:** 14

**Priority breakdown:**
- **High Priority (Security):** 4 items (#1-4)
- **Medium Priority (Validation & Error Handling):** 6 items (#5-10)
- **Low Priority (Styling & Code Quality):** 4 items (#11-14)

**Recommended order of implementation:**
1. Address all security issues first (#1-4)
2. Implement input validation (#5-6)
3. Improve error handling and UX (#7-10)
4. Clean up styling and code quality issues (#11-14)
