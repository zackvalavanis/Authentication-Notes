# 📚 Auth & Session Management Notes & Cheat Sheet

A comprehensive reference guide for tracking authentication states and token storage. Use this sheet to quickly look up core concepts, commands, and syntax.

---

## 🗺️ Table of Contents
* [🚀 Getting Started](#-getting-started)
* [💻 Core Syntax & Commands](#-core-syntax--commands)
* [💡 Best Practices & Tips](#-best-practices--tips)
* [🔍 Troubleshooting & Common Errors](#-troubleshooting--common-errors)
* [🔗 Useful Resources](#-useful-resources)

---

## 🚀 Getting Started

### Basic Configuration
To handle secure identity verification, we need a robust client-side storage mechanism that balances security vulnerabilities (like XSS and CSRF) against performance and seamless user experience.

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

---

## 💡 Best Practices & Tips

### 🛡️ JWT Storage Strategy: State + LocalStorage

> [!NOTE]
> The standard production answer for storing JWTs is a combination of **in-memory state** and **local storage**.

#### The Core Problem: In-Memory State
In-memory state **does not survive a refresh**. When a user reloads the page, the JavaScript runtime completely wipes and restarts.
* **The Consequence:** If your token *only* lives in memory, the user is logged out instantly on refresh.

#### The Solution: LocalStorage / SessionStorage
Unlike memory, `LocalStorage` and `SessionStorage` **survive a page refresh**. 

#### 🚀 The Hybrid Approach
Because of this behavior, the common industry practice is to:
1. **Keep the token in state** for fast, synchronous client-side API requests.
2. **Copy the token to local storage** so that on page reload, an initialization script can read it back into state and seamlessly maintain the user session.

### Additional Development Workflow Tips
* **Keep it modular:** Break down large authentication functions into smaller single-purpose middleware or helper components.
* **Comment the 'Why':** Don't comment *what* the storage code does; comment *why* you chose a specific option (e.g., listing why a certain token isn't being stored inside a cookie).
* **Shortcut:** Use `Ctrl + Shift + P` (or `Cmd + Shift + P` on Mac) inside GitHub's browser editor to quickly open the command palette.

---

## 🔍 Troubleshooting & Common Errors

### ❌ Error: `TypeError: Cannot read properties of null (reading 'token')`
* **Cause:** This usually happens when the application state tries to fetch the JWT before the initialization script finishes parsing the token from local storage.
* **Solution:** Wrap the downstream API requests in a validation check or add an explicit loading state conditional check:
  ```bash
  # Clear local storage keys to reset corrupt or expired tokens manually during debugging
  window.localStorage.removeItem('jwt_token');
  ```

---

## 🔗 Useful Resources
* [Official MDN Web Storage API Documentation](https://mozilla.org) - The source of truth for LocalStorage workflows.
* [GitHub Markdown Guide](https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax) - Official GitHub text formatting syntax.
