# 📚 Auth & Session Management Notes & Cheat Sheet

A comprehensive reference guide for tracking authentication states and token storage. Use this sheet to quickly look up core concepts, commands, and syntax.

---

## 🗺️ Table of Contents
* [🚀 Getting Started](#-getting-started)
* [💻 Core Syntax & Commands](#-core-syntax--commands)
* [🛠️ Building the AuthProvider: Step by Step](#️-building-the-authprovider-step-by-step)
* [💡 Best Practices & Tips](#-best-practices--tips)
* [🔍 Troubleshooting & Common Errors](#-troubleshooting--common-errors)
* [🔗 Useful Resources](#-useful-resources)

---

## 🚀 Getting Started

### Basic Configuration
To handle secure identity verification, we need a robust client-side storage mechanism that balances security vulnerabilities (like XSS and CSRF) against performance and seamless user experience.

**Stack for this project (MoonWatch):** React + TypeScript (Vite) frontend, FastAPI backend, JWT auth.

---

## 💻 Core Syntax & Commands

> [!NOTE]
> Keep your code snippets minimal and focused on a single actionable concept.

### 1. Initializing Auth State
Explanation of what this does and when to use it.

```javascript
// Example initialization script to load saved tokens
function initializeSession() {
    const savedToken = localStorage.getItem('jwt_token');
    if (savedToken) {
        // Hydrate your fast synchronous in-memory application state
        setAppState({ token: savedToken, isAuthenticated: true });
        console.log("Session successfully restored from LocalStorage");
    }
}
```

### 2. Core Storage Properties Lookup
Quick reference overview of the web storage options available in modern browsers:

| API / Mechanism | Lifespan | Scope | Access Speed |
| :--- | :--- | :--- | :--- |
| `In-Memory State` | Wiped instantly on page reload / tab close | Isolated to the runtime instance | Extremely fast (Synchronous) |
| `LocalStorage` | Persistent indefinitely until explicitly cleared | Shared across tabs/windows on the same origin | Fast (Synchronous Disk I/O) |
| `SessionStorage` | Survives reloads, cleared when tab closes | Tab-specific isolation | Fast (Synchronous Disk I/O) |

### 3. TypeScript Patterns Used

| What | Syntax | Notes |
| :--- | :--- | :--- |
| State that can be empty | `useState<User \| null>(null)` | Without the type, `useState(null)` can *only* ever be `null` |
| Function type (async) | `login: (credentials: LoginInfo) => Promise<void>` | Every `async` function returns a `Promise` |
| Function type (sync, no args) | `logout: () => void` | |
| Children prop | `{ children }: { children: ReactNode }` | `children` is a special name React fills in automatically |
| Context with no default | `createContext<AuthContextType \| null>(null)` | `null` only shows up if used outside the provider |

---

## 🛠️ Building the AuthProvider: Step by Step

### Step 0: Know how the backend proves login
* Two common ways: a **token** (JWT) the frontend stores and sends, or an **httpOnly cookie** the browser sends automatically.
* MoonWatch uses **JWT**. That decides where the token lives and what "logout" means.

### Step 1: Decide what the provider holds
* **State:** `user`, `token`, `loading`
* **Actions:** `login`, `logout`, `signup`
* Signup does **not** return a JWT, so `signup` will call `login` with the same credentials after the account is created.
* ❌ Never set a "temp user." `user` should only mean "authenticated and has a token." If state and reality disagree, you get confusing bugs.

### Step 2: Describe the context (the type)
```ts
export interface AuthContextType {
  user: User | null
  loading: boolean
  login: (credentials: LoginInfo) => Promise<void>
  logout: () => void
}
```
* `LoginInfo` (email + password) is **not** the same as `User`. Never put a password on the `User` type.

### Step 3: Create the context
```ts
const AuthContext = createContext<AuthContextType | null>(null)
```
* Created **outside** the component, so it's made once, not every render.

### Step 4: The provider component
```tsx
export function AuthProvider({ children }: { children: ReactNode }) {
  // state + actions live here
  return (
    <AuthContext.Provider value={{ user, login, loading, logout }}>
      {children}
    </AuthContext.Provider>
  )
}
```
* `AuthProvider` = **my** component. `AuthContext.Provider` = the built-in component React attaches to the context.
* ⚠️ Rendering `<AuthProvider>` inside `AuthProvider` = infinite loop / "maximum call stack."
* `value={{ ... }}`: outer braces = "JavaScript goes here," inner braces = the object.
* ⚠️ `value={user, login, loading}` (single braces) only keeps the **last** value.

### Step 5: The `useAuth` hook
```ts
export function useAuth() {
  const context = useContext(AuthContext)
  if (context === null) {
    throw new Error("useAuth must be inside AuthProvider")
  }
  return context
}
```
* Keeps the raw context private: everyone goes through `useAuth()`.
* The guard gives a clear error instead of a vague "cannot read properties of null," and tells TypeScript the result is never `null`.
* Fine to keep in the same file as the provider. (Vite's ESLint may warn about Fast Refresh; it's only a dev warning.)

### Step 6: Wrap the app
* In `main.tsx`, wrap `<App />` with `<AuthProvider>`.
* If using React Router, put the provider **inside** the router so auth actions can navigate.
* ✅ Test: `console.log(useAuth())` should show `{ user: null, loading: false, login: ƒ }`.

### Step 7: Call the login endpoint
* Test with a bare login page and a button with hardcoded credentials before building the real form (one new thing at a time).
* ⚠️ `onClick={login}` passes the **click event** as the argument. Use `onClick={() => login({ email, password })}`.
* FastAPI's `OAuth2PasswordRequestForm` expects **form data**, not JSON, with a field named `username`:
```ts
const res = await fetch(`${api}/auth/login`, {
  method: "POST",
  body: new URLSearchParams({ username: credentials.email, password: credentials.password }),
})
```
* No `Content-Type` header: `fetch` sets the right one automatically for `URLSearchParams`. Adding `application/json` overrides it and causes a 422.
* Kept the backend as-is so the **Authorize** button in `/docs` still works.

### Step 8: Store the token
```ts
const accessToken = data.access_token
localStorage.setItem("Access_token", accessToken)
setToken(accessToken)
```
* The token is a `string`, the user is an object: they get **separate** state.
* A JWT is header.payload.signature. The payload (`sub` = user id, `exp` = expiry) is only **encoded, not encrypted**. Anyone can read it on jwt.io, so never put secrets in it.
* TODO: make the storage key a constant, since it's used in login, logout, and refresh.

### Step 9: Fetch the user with the token
```ts
const meRes = await fetch(`${api}/auth/me`, {
  headers: { Authorization: `Bearer ${accessToken}` },
})
```
* Use the **local** `accessToken`, not `token` state. `setToken` doesn't update `token` until the next render.
* Why keep `user` at all: the UI can show who's logged in, and on refresh `/me` doubles as a **token validity check** (works = valid, 401 = expired).
* If `/me` fails: call `logout()` so there's never a token with no user ("half-logged-in").

### `logout`
```ts
const logout = () => {
  setToken(null)
  localStorage.removeItem("Access_token")
  setUser(null)
}
```
* Written once, reused by every failure path. No server call needed for JWT logout.

### Step 10: Make failures reach the form
* Decision: **throw** errors instead of logging and returning.
* `!res.ok` → `throw new Error(data.detail || "Login failed")` (backend's message, with a fallback)
* No token → generic `"Something went wrong"`
* `/me` failure → `logout()` then a user-friendly message
* The outer `catch` must re-throw the **same** error so the original message survives:
```ts
} catch (error) {
  console.error("Login error", error)
  throw error
} finally {
  setLoading(false)
}
```
* `finally` still runs when re-throwing.

### Next up
- [ ] Real login page: controlled inputs, `<form onSubmit>`, `preventDefault`, password `type`
- [ ] Form `try/catch` around `await login(...)`, error state, `error instanceof Error`
- [ ] Submitting state to disable the button
- [ ] Refresh: on mount, read the token from localStorage and validate it with `/me` (`loading` starts as `true` for this)
- [ ] `signup` (calls `login` after creating the account)
- [ ] API helper that attaches `Authorization: Bearer` to every request
- [ ] Protected routes
- [ ] Cleanup: flatten nested `try`, rename storage key to lowercase constant, remove debug `console.log`s

---

## 💡 Best Practices & Tips

### 🛡️ JWT Storage Strategy: State + LocalStorage

> [!NOTE]
> A very common approach for storing JWTs is a combination of **in-memory state** and **local storage**. It's not the most secure option: see the tradeoff below.

#### The Core Problem: In-Memory State
In-memory state **does not survive a refresh**. When a user reloads the page, the JavaScript runtime completely wipes and restarts.
* **The Consequence:** If your token *only* lives in memory, the user is logged out instantly on refresh.

#### The Solution: LocalStorage / SessionStorage
Unlike memory, `LocalStorage` and `SessionStorage` **survive a page refresh**.

#### 🚀 The Hybrid Approach
Because of this behavior, the common industry practice is to:
1. **Keep the token in state** for fast, synchronous client-side API requests.
2. **Copy the token to local storage** so that on page reload, an initialization script can read it back into state and seamlessly maintain the user session.

#### ⚖️ The Tradeoff (interview answer)
* `localStorage` is readable by **any** script on the page, so an XSS attack can steal the token.
* The more security-focused production approach is an **httpOnly cookie**: JavaScript can't read it at all.
* Good answer: "I used localStorage for simplicity, here's the XSS risk, and in production I'd move to httpOnly cookies."

### Provider vs. Form: who owns what
| Provider (`AuthContext.tsx`) | Login page (`Login.tsx`) |
| :--- | :--- |
| API calls, URLs, tokens, localStorage | Input values, submit handler |
| Setting `user` / `token` | Showing error messages |
| Throwing errors | Catching errors, navigating after success |

Rule of thumb: if the form mentions a URL or `localStorage`, it belongs in the provider. If the provider mentions input values or error display, it belongs in the form.

### Additional Development Workflow Tips
* **Keep it modular:** Break down large authentication functions into smaller single-purpose middleware or helper components.
* **Comment the 'Why':** Don't comment *what* the storage code does; comment *why* you chose a specific option (e.g., listing why a certain token isn't being stored inside a cookie).
* **One new thing at a time:** test each piece before building the next, so you always know which part broke.
* **Shortcut:** Use `Ctrl + Shift + P` (or `Cmd + Shift + P` on Mac) inside GitHub's browser editor to quickly open the command palette.

---

## 🔍 Troubleshooting & Common Errors

### ❌ Error: `TypeError: Cannot read properties of null (reading 'token')`
* **Cause:** This usually happens when the application state tries to fetch the JWT before the initialization script finishes parsing the token from local storage.
* **Solution:** Wrap the downstream API requests in a validation check or add an explicit loading state conditional check. To reset a corrupt or expired token manually while debugging, run this in the browser console:
  ```javascript
  window.localStorage.removeItem('jwt_token');
  ```

### ❌ `POST http://localhost:5173/undefined/auth/login 404`
* **Cause:** `import.meta.env.VITE_BACKEND_API` was `undefined`. The variable was only in `.env.production` / `.env.test`.
* **How Vite loads env files:** `npm run dev` (development mode) reads `.env`, `.env.local`, `.env.development`, `.env.development.local`. `npm run build` reads `.env.production`.
* **Fix:** put `VITE_BACKEND_API="http://localhost:8000"` in `.env.development`, in the same folder as `vite.config.ts`, then **restart the dev server**.
* Variables must start with `VITE_` to reach the frontend.

### ❌ CORS: "No 'Access-Control-Allow-Origin' header is present"
* **Cause:** frontend (port 5173) and backend (port 8000) are different **origins**. The browser blocks cross-origin requests unless the server allows them. Postman/curl don't enforce this, browsers do.
* **Fix is on the backend**, in `main.py`:
```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=CORS_ORIGINS,
    allow_methods=["*"],
    allow_headers=["*"],
)
```
* A list of origins in a config file does nothing until it's passed to the middleware.
* Origins must match **exactly**: scheme (`http` vs `https`), host, and port.

### ❌ CORS: "Response to preflight request doesn't pass access control check: It does not have HTTP ok status"
* The browser sends an `OPTIONS` **preflight** before the real request. A 400 means the middleware rejected it on purpose.
* **Debug:** Network tab → the `preflight` row → read the `Access-Control-Allow-*` response headers.
* Defaults if not set: `allow_methods` = `GET` only (blocks `POST`), `allow_headers` = safelisted headers only (blocks `Authorization`).
* Hit it twice: once for `POST` (login), once for `Authorization` (`/me`).

### ❌ `422 Unprocessable Entity`
* FastAPI received the request, but the body didn't match what the route expects. The Response tab's `detail` names the field.
* Here: the route used `OAuth2PasswordRequestForm` (form data, `username` field) but the frontend sent JSON with `email`.

### ❌ Login "succeeds" in the form even with a wrong password
* **Cause:** a `catch` that logs but doesn't re-throw swallows the error.
* **Fix:** `throw error` at the end of the `catch`.

### ℹ️ Console noise to ignore
* `chrome-extension://...`, `gpc.js`, `runtime.lastError`: browser extensions, not the app.
* Logs printing twice: React StrictMode renders twice in development on purpose.

---

## 🔗 Useful Resources
* [MDN Web Storage API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API) - The source of truth for LocalStorage workflows.
* [MDN URLSearchParams](https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams) - Building form-encoded request bodies.
* [React: useContext](https://react.dev/reference/react/useContext) - Context and providers.
* [FastAPI: CORS](https://fastapi.tiangolo.com/tutorial/cors/) - CORSMiddleware settings.
* [jwt.io](https://jwt.io) - Inspect what's inside a JWT.
* [GitHub Markdown Guide](https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax) - Official GitHub text formatting syntax.
