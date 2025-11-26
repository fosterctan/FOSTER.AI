# FOSTER.AI HomeNote System - Security & Code Audit Report

**Date:** November 26, 2025
**Auditor:** Claude Code
**Scope:** Complete codebase audit of FOSTER.AI Home AI Chat System
**Files Reviewed:** `ai-chat.html`, `README.md`

---

## Executive Summary

The FOSTER.AI HomeNote system is a single-page HTML application providing a chat interface for a home AI assistant. The current implementation is a minimal frontend-only solution with **several critical security vulnerabilities** and **significant functionality gaps** that should be addressed before production use.

**Risk Level: HIGH**

| Category | Issues Found | Critical | High | Medium | Low |
|----------|-------------|----------|------|--------|-----|
| Security | 9 | 3 | 3 | 2 | 1 |
| Code Quality | 8 | 0 | 2 | 4 | 2 |
| Functionality | 7 | 0 | 3 | 3 | 1 |
| Accessibility | 4 | 0 | 1 | 2 | 1 |
| **Total** | **28** | **3** | **9** | **11** | **5** |

---

## Critical Security Issues

### 1. SEC-001: No URL Validation (CRITICAL)
**Location:** `ai-chat.html:274-292`

**Issue:** The `saveSettings()` function accepts any URL input without validation. A malicious actor could:
- Enter a malicious URL that intercepts and logs conversations
- Inject JavaScript through the URL
- Cause the application to send requests to unintended endpoints

**Current Code:**
```javascript
function saveSettings() {
    let url = document.getElementById('apiUrl').value.trim();
    if (url.endsWith('/')) {
        url = url.slice(0, -1);
    }
    if (!url) {
        alert('Please enter a URL');
        return;
    }
    apiUrl = url;  // No validation!
    localStorage.setItem('apiUrl', url);
}
```

**Recommendation:**
```javascript
function saveSettings() {
    let url = document.getElementById('apiUrl').value.trim();

    // Validate URL format
    try {
        const parsed = new URL(url);
        if (!['https:', 'http:'].includes(parsed.protocol)) {
            alert('URL must use HTTP or HTTPS protocol');
            return;
        }
    } catch (e) {
        alert('Please enter a valid URL');
        return;
    }

    // Remove trailing slash
    url = url.replace(/\/+$/, '');

    apiUrl = url;
    localStorage.setItem('apiUrl', url);
}
```

---

### 2. SEC-002: No HTTPS Enforcement (CRITICAL)
**Location:** `ai-chat.html:350`

**Issue:** The application allows HTTP connections, exposing all chat data to interception via man-in-the-middle attacks. Personal home health conversations could be intercepted.

**Recommendation:**
- Enforce HTTPS-only connections
- Add warning if user attempts HTTP connection
- Consider certificate pinning for known endpoints

---

### 3. SEC-003: Sensitive Data in localStorage (CRITICAL)
**Location:** `ai-chat.html:267, 288`

**Issue:** The API URL is stored in localStorage without encryption. If the device is compromised, attackers can easily retrieve the endpoint URL.

**Recommendation:**
- For sensitive applications, consider using sessionStorage (cleared on tab close)
- Implement encryption for stored values
- Add session expiration

---

### 4. SEC-004: No Input Sanitization (HIGH)
**Location:** `ai-chat.html:355-366`

**Issue:** User messages are sent directly to the API without sanitization, potentially enabling prompt injection attacks against the AI backend.

**Recommendation:**
```javascript
function sanitizeInput(input) {
    // Remove potentially dangerous characters
    return input
        .replace(/[<>]/g, '') // Remove HTML brackets
        .slice(0, 10000);     // Limit message length
}
```

---

### 5. SEC-005: No Rate Limiting (HIGH)
**Location:** `ai-chat.html:324-390`

**Issue:** No client-side rate limiting exists. A user could spam requests, potentially causing:
- DoS against the AI backend
- Excessive resource consumption
- API quota exhaustion

**Recommendation:**
```javascript
let lastMessageTime = 0;
const MIN_INTERVAL = 1000; // 1 second minimum between messages

async function sendMessage() {
    const now = Date.now();
    if (now - lastMessageTime < MIN_INTERVAL) {
        showError('Please wait before sending another message');
        return;
    }
    lastMessageTime = now;
    // ... rest of function
}
```

---

### 6. SEC-006: No Authentication/Authorization (HIGH)
**Location:** `ai-chat.html:349-367`

**Issue:** Anyone with the ngrok URL can access and use the AI. No authentication mechanism exists.

**Recommendation:**
- Implement API key authentication
- Add user session management
- Consider OAuth2 for multi-user scenarios

---

### 7. SEC-007: Exposed Configuration (MEDIUM)
**Location:** `ai-chat.html:356-365`

**Issue:** Model configuration (model name, temperature, max_tokens) is hardcoded in client-side JavaScript, exposing system details.

```javascript
body: JSON.stringify({
    model: "local-model",    // Exposed
    temperature: 0.7,        // Exposed
    max_tokens: -1,          // Potentially dangerous
    stream: false
})
```

**Recommendation:**
- Move configuration to server-side
- Use environment variables
- Implement configuration endpoint

---

### 8. SEC-008: Dangerous max_tokens Setting (MEDIUM)
**Location:** `ai-chat.html:364`

**Issue:** `max_tokens: -1` is set, which depending on the backend implementation could mean unlimited tokens, potentially causing resource exhaustion.

**Recommendation:**
- Set explicit token limit (e.g., `max_tokens: 2048`)
- Implement backend validation

---

### 9. SEC-009: No Request Timeout (LOW)
**Location:** `ai-chat.html:349-367`

**Issue:** The fetch request has no timeout, potentially causing the UI to hang indefinitely.

**Recommendation:**
```javascript
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 30000);

const response = await fetch(url, {
    signal: controller.signal,
    // ... other options
});
clearTimeout(timeout);
```

---

## Code Quality Issues

### 1. CQ-001: No Conversation History (HIGH)
**Location:** `ai-chat.html:357-361`

**Issue:** Only the current message is sent to the API. The AI has no context of previous messages, making multi-turn conversations impossible.

**Current Code:**
```javascript
messages: [
    {
        role: "user",
        content: message
    }
]
```

**Recommendation:**
```javascript
let conversationHistory = [];

async function sendMessage() {
    // ...
    conversationHistory.push({ role: "user", content: message });

    const response = await fetch(url, {
        body: JSON.stringify({
            model: "local-model",
            messages: conversationHistory,
            // ...
        })
    });

    const aiResponse = data.choices[0].message.content;
    conversationHistory.push({ role: "assistant", content: aiResponse });
}
```

---

### 2. CQ-002: No Error Handling for Malformed Responses (HIGH)
**Location:** `ai-chat.html:373-377`

**Issue:** The code assumes the API response will always have `choices[0].message.content`. If the structure is different, it will crash.

```javascript
const data = await response.json();
const aiResponse = data.choices[0].message.content;  // No null checking
```

**Recommendation:**
```javascript
const data = await response.json();
const aiResponse = data?.choices?.[0]?.message?.content;
if (!aiResponse) {
    throw new Error('Invalid response format from AI');
}
```

---

### 3. CQ-003: Global Variables (MEDIUM)
**Location:** `ai-chat.html:267`

**Issue:** `apiUrl` is a global variable, polluting the global namespace and making the code harder to test.

**Recommendation:**
- Use a module pattern or ES6 modules
- Encapsulate state in a class or closure

---

### 4. CQ-004: Inline Event Handlers (MEDIUM)
**Location:** `ai-chat.html:232, 260-262`

**Issue:** Using inline `onclick` and `onkeypress` handlers mixes HTML and JavaScript.

```html
<button onclick="saveSettings()">Connect</button>
<input onkeypress="if(event.key==='Enter') sendMessage()" />
```

**Recommendation:**
```javascript
document.getElementById('apiUrl').addEventListener('keypress', (e) => {
    if (e.key === 'Enter') saveSettings();
});
document.getElementById('connectBtn').addEventListener('click', saveSettings);
```

---

### 5. CQ-005: No Separation of Concerns (MEDIUM)
**Location:** Entire file

**Issue:** HTML, CSS, and JavaScript are all in one file. This makes maintenance difficult.

**Recommendation:**
- Separate into `index.html`, `styles.css`, `app.js`
- Consider using a build system (Vite, Webpack)

---

### 6. CQ-006: Magic Numbers/Strings (MEDIUM)
**Location:** Throughout

**Issue:** Hardcoded values like colors (`#667eea`), model names, temperatures are scattered throughout.

**Recommendation:**
- Create a configuration object at the top
- Use CSS custom properties for colors

---

### 7. CQ-007: Confusing Function Naming (LOW)
**Location:** `ai-chat.html:290`

**Issue:** `showError()` is used to show success messages, which is confusing.

```javascript
showError('Connected! You can start chatting now.', false);
```

**Recommendation:**
- Rename to `showNotification()` or create separate `showSuccess()` function

---

### 8. CQ-008: No TypeScript (LOW)
**Location:** Entire codebase

**Issue:** No type safety, making refactoring risky and bugs harder to catch.

**Recommendation:**
- Migrate to TypeScript
- Add JSDoc type annotations at minimum

---

## Functionality Issues

### 1. FUNC-001: Chat History Lost on Refresh (HIGH)
**Location:** `ai-chat.html`

**Issue:** All chat history is lost when the page is refreshed.

**Recommendation:**
- Store conversation history in localStorage/IndexedDB
- Implement export/import functionality

---

### 2. FUNC-002: No Settings Management After Initial Setup (HIGH)
**Location:** `ai-chat.html:269-272`

**Issue:** Once the API URL is saved, there's no UI to change it without clearing localStorage.

**Recommendation:**
- Add a settings button/menu
- Allow URL modification at any time

---

### 3. FUNC-003: No Connection Validation (HIGH)
**Location:** `ai-chat.html:274-292`

**Issue:** The app shows "Connected!" without actually testing the connection.

**Recommendation:**
```javascript
async function testConnection(url) {
    try {
        const response = await fetch(`${url}/v1/models`, {
            method: 'GET',
            signal: AbortSignal.timeout(5000)
        });
        return response.ok;
    } catch {
        return false;
    }
}
```

---

### 4. FUNC-004: No Clear Chat Function (MEDIUM)
**Location:** N/A

**Issue:** Users cannot clear the chat history.

**Recommendation:**
- Add "Clear Chat" button
- Implement keyboard shortcut

---

### 5. FUNC-005: No Streaming Support (MEDIUM)
**Location:** `ai-chat.html:365`

**Issue:** Streaming is disabled (`stream: false`), causing users to wait for the full response.

**Recommendation:**
- Enable streaming for better UX
- Show partial responses as they arrive

---

### 6. FUNC-006: No Message Copy/Delete (MEDIUM)
**Location:** N/A

**Issue:** Users cannot copy or delete individual messages.

---

### 7. FUNC-007: No Reconnection Logic (LOW)
**Location:** `ai-chat.html:379-383`

**Issue:** On connection failure, user must manually retry.

**Recommendation:**
- Implement automatic retry with exponential backoff
- Add manual "Reconnect" button

---

## Accessibility Issues

### 1. A11Y-001: Missing ARIA Labels (HIGH)
**Location:** `ai-chat.html:256-263`

**Issue:** Input fields and buttons lack proper ARIA labels for screen readers.

**Recommendation:**
```html
<input
    type="text"
    id="userInput"
    placeholder="Type your message..."
    aria-label="Chat message input"
    role="textbox"
/>
<button id="sendBtn" aria-label="Send message">Send</button>
```

---

### 2. A11Y-002: No Keyboard Navigation for Messages (MEDIUM)
**Location:** `ai-chat.html:243-247`

**Issue:** Message elements are not focusable or navigable via keyboard.

---

### 3. A11Y-003: Low Color Contrast (MEDIUM)
**Location:** `ai-chat.html:89-90`

**Issue:** Gray background (`#f0f0f0`) with dark text (`#333`) may not meet WCAG AA contrast requirements in all cases.

---

### 4. A11Y-004: No Skip Links (LOW)
**Location:** N/A

**Issue:** No skip links for keyboard users to bypass repeated content.

---

## Recommendations Summary

### Immediate Actions (Critical)
1. Implement URL validation before accepting API endpoints
2. Enforce HTTPS connections only
3. Add basic authentication mechanism

### Short-term Improvements (High)
1. Add conversation history support
2. Implement proper error handling
3. Add rate limiting
4. Add settings management UI
5. Add connection validation

### Medium-term Improvements
1. Separate HTML/CSS/JS into distinct files
2. Add accessibility features (ARIA labels)
3. Implement streaming responses
4. Add message persistence
5. Migrate to TypeScript

### Long-term Considerations
1. Implement proper backend authentication
2. Add user session management
3. Consider Progressive Web App (PWA) features
4. Add offline support
5. Implement end-to-end encryption for sensitive conversations

---

## Conclusion

The FOSTER.AI HomeNote system is a functional prototype with significant security and usability gaps. Before using this system for any sensitive home health conversations, the **critical security issues must be addressed**. The current implementation is suitable only for local development/testing purposes.

---

*This audit was performed on the codebase as of commit `f9cc12b`. Future updates may address some of these issues.*
