# Mod: PIN Gate

## What it does
Replaces a password input with a 4-digit PIN entry screen for EPS web apps and PWAs. Features an on-screen numpad, dot indicators, auto-submit on the 4th digit, a shake animation on wrong PIN, and a Fibonacci lockout after 3 failed attempts. Designed to feel native on iOS.

## Why this works
Keyboard-based password inputs are hostile in PWA standalone mode — they pop the keyboard, obscure the screen, and feel like a website, not an app. A fixed numpad keeps the UI fully in control of the app, works well with thumbs, and pairs naturally with iOS/Android PWA full-screen mode. Fibonacci lockout provides meaningful brute-force resistance without a fixed cap.

## When to apply
Any EPS web app or PWA that needs a lightweight auth gate: terminal apps, dashboards, personal tools behind Tailscale. Not a replacement for production auth — ideal for personal apps where simplicity matters more than enterprise-grade security.

## Implementation

### 1. Add CSS variables (if not already present)
The gate assumes these CSS custom properties exist in `:root`. Adjust values to match your app's palette.
```css
:root {
  --bg:         #0d0d0d;
  --surface:    #161616;
  --border:     #2a2a2a;
  --accent:     #c9a84c;
  --accent-dim: #7a6030;
  --text:       #e8e8e8;
  --text-dim:   #666;
}
```

### 2. Add gate CSS
```css
#gate {
  display: none;
  position: fixed;
  inset: 0;
  background: var(--bg);
  z-index: 100;
  align-items: center;
  justify-content: center;
}
#gate-inner {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 28px;
  width: 240px;
}
#gate-logo {
  font-size: 18px;
  font-weight: 700;
  color: var(--accent);
  letter-spacing: 0.1em;
  text-align: center;
}
#gate-dots {
  display: flex;
  gap: 16px;
}
.gate-dot {
  width: 14px;
  height: 14px;
  border-radius: 50%;
  border: 2px solid var(--border);
  background: transparent;
  transition: background 0.1s, border-color 0.1s;
}
.gate-dot.filled { background: var(--accent); border-color: var(--accent); }
.gate-dot.error  { background: #e05252;        border-color: #e05252; }
.gate-dot.locked { background: var(--text-dim); border-color: var(--text-dim); }

/* Shake animation — plays on wrong PIN */
@keyframes gate-shake {
  0%   { transform: translateX(0); }
  15%  { transform: translateX(-8px); }
  30%  { transform: translateX(7px); }
  45%  { transform: translateX(-6px); }
  60%  { transform: translateX(5px); }
  75%  { transform: translateX(-3px); }
  90%  { transform: translateX(2px); }
  100% { transform: translateX(0); }
}
#gate-dots.shake {
  animation: gate-shake 0.45s ease;
}

#gate-pad {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 10px;
  width: 100%;
}
.pad-key {
  background: var(--surface);
  border: 1px solid var(--border);
  color: var(--text);
  border-radius: 8px;
  font-size: 20px;
  font-weight: 500;
  height: 56px;
  cursor: pointer;
  user-select: none;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.1s;
  -webkit-tap-highlight-color: transparent;
}
.pad-key:active          { background: var(--accent-dim); }
.pad-key.pad-empty       { visibility: hidden; pointer-events: none; }
.pad-key.pad-del         { font-size: 16px; color: var(--text-dim); }
```

### 3. Add gate HTML (before your main app `<div>`)
```html
<div id="gate">
  <div id="gate-inner">
    <div id="gate-logo">yourapp</div>
    <div id="gate-dots">
      <div class="gate-dot" id="dot-0"></div>
      <div class="gate-dot" id="dot-1"></div>
      <div class="gate-dot" id="dot-2"></div>
      <div class="gate-dot" id="dot-3"></div>
    </div>
    <div id="gate-pad">
      <button class="pad-key" data-digit="1">1</button>
      <button class="pad-key" data-digit="2">2</button>
      <button class="pad-key" data-digit="3">3</button>
      <button class="pad-key" data-digit="4">4</button>
      <button class="pad-key" data-digit="5">5</button>
      <button class="pad-key" data-digit="6">6</button>
      <button class="pad-key" data-digit="7">7</button>
      <button class="pad-key" data-digit="8">8</button>
      <button class="pad-key" data-digit="9">9</button>
      <div class="pad-key pad-empty"></div>
      <button class="pad-key" data-digit="0">0</button>
      <button class="pad-key pad-del" id="pad-del">⌫</button>
    </div>
  </div>
</div>
```

### 4. Add gate JS
`submitPin()` starts your auth action but does **not** hide the gate — the gate only disappears once the server confirms success (e.g. `onopen` for a WebSocket). This prevents the UI from flashing through to the app for a split second on a wrong PIN.

```js
let pin = '';
let failCount = 0;
let fibA = 1, fibB = 1;
let padLocked = false;

function nextFib() {
  const val = fibA;
  [fibA, fibB] = [fibB, fibA + fibB];
  return val;
}

function updateDots(state) {
  // state: 'normal' | 'error' | 'locked'
  for (let i = 0; i < 4; i++) {
    const dot = document.getElementById('dot-' + i);
    dot.classList.toggle('filled', state !== 'locked' && i < pin.length);
    dot.classList.toggle('error',  state === 'error');
    dot.classList.toggle('locked', state === 'locked');
  }
}

function setPadDisabled(disabled) {
  padLocked = disabled;
  document.querySelectorAll('.pad-key').forEach(k => k.disabled = disabled);
}

function lockPad(seconds) {
  pin = '';
  setPadDisabled(true);
  updateDots('locked');
  const logo = document.getElementById('gate-logo');
  const origText = logo.textContent;
  let remaining = seconds;
  logo.textContent = `try again in ${remaining}s`;
  const t = setInterval(() => {
    remaining--;
    if (remaining <= 0) {
      clearInterval(t);
      logo.textContent = origText;
      setPadDisabled(false);
      updateDots('normal');
    } else {
      logo.textContent = `try again in ${remaining}s`;
    }
  }, 1000);
}

function showGate() {
  pin = '';
  if (!padLocked) updateDots('normal');
  document.getElementById('gate').style.display = 'flex';
}

function hideGate() {
  document.getElementById('gate').style.display = 'none';
}

function pinError() {
  failCount++;
  updateDots('error');
  // Shake the dots
  const dotsEl = document.getElementById('gate-dots');
  dotsEl.classList.remove('shake');
  void dotsEl.offsetWidth; // force reflow so re-triggering works
  dotsEl.classList.add('shake');
  if (failCount >= 3) {
    failCount = 0;
    const wait = nextFib();
    setTimeout(() => lockPad(wait), 600);
  } else {
    setTimeout(() => { pin = ''; updateDots('normal'); }, 600);
  }
}

function submitPin() {
  // ── Replace this with your app's auth action ──
  // Do NOT hide the gate here. Hide it only after the server confirms success.
  // e.g. sessionStorage.setItem('token', pin); connect();
}

document.getElementById('gate-pad').addEventListener('click', (e) => {
  if (padLocked) return;
  const key = e.target.closest('.pad-key');
  if (!key) return;
  if (key.id === 'pad-del') {
    pin = pin.slice(0, -1);
    updateDots('normal');
    return;
  }
  const digit = key.dataset.digit;
  if (digit === undefined || pin.length >= 4) return;
  pin += digit;
  updateDots('normal');
  if (pin.length === 4) submitPin();
});

// Show the gate on load
showGate();
```

### 5. Server-side enforcement (Node.js)
If your app authenticates via a server (e.g. WebSocket token), add IP-based Fibonacci lockout there too. Client-side lockout is UX; server-side is actual security.

```js
const loginState = new Map(); // ip -> { failures, lockedUntil, fibA, fibB }

function getState(ip) {
  return loginState.get(ip) || { failures: 0, lockedUntil: 0, fibA: 1, fibB: 1 };
}

function recordFailure(ip) {
  const s = getState(ip);
  s.failures++;
  if (s.failures >= 3) {
    const wait = s.fibA;
    [s.fibA, s.fibB] = [s.fibB, s.fibA + s.fibB];
    s.lockedUntil = Date.now() + wait * 1000;
    s.failures = 0;
  }
  loginState.set(ip, s);
}

function recordSuccess(ip) { loginState.delete(ip); }

// Example: ws verifyClient callback form
verifyClient: ({ req }, cb) => {
  const ip = req.socket.remoteAddress;
  const s = getState(ip);
  if (Date.now() < s.lockedUntil) { cb(false, 429, 'Too Many Requests'); return; }
  const token = new URL(req.url, 'https://localhost').searchParams.get('token');
  if (token === SECRET) { recordSuccess(ip); cb(true); }
  else { recordFailure(ip); cb(false, 401, 'Unauthorized'); }
}
```

### 6. Wiring gate hide + pinError to server response (WebSocket pattern)
The key insight: **`onopen` fires only if the PIN was correct**. Track a `wsConnected` flag to distinguish a bad token (handshake never opened) from a network drop (connection opened then closed).

```js
let wsConnected = false;

// In your connect() function:
ws.onopen = () => {
  wsConnected = true;
  hideGate(); // ← gate hidden here, not in submitPin()
  // ... rest of setup
};

ws.onclose = () => {
  const wasConnected = wsConnected;
  wsConnected = false;
  if (!wasConnected) {
    // Handshake was rejected — bad token
    sessionStorage.removeItem('token');
    showGate();
    pinError(); // ← shake + red dots
  } else if (!intentionalClose) {
    // Was connected, then dropped — reconnect silently
    reconnectTimer = setTimeout(connect, 2000);
  }
};
```

This pattern also means a network blip while you're logged in will silently reconnect rather than kicking you back to the PIN screen.

## Customization
- **PIN length** — change the `4` in `pin.length >= 4` and add/remove `.gate-dot` elements
- **Attempt limit** — change `>= 3` in `pinError()` to any number
- **Logo text** — set `#gate-logo` content to your app name
- **Colors** — override `--accent`, `--surface`, `--border` in `:root`
- **Key size** — adjust `height: 56px` and `font-size: 20px` on `.pad-key`
- **Shake intensity** — adjust the `translateX` values in `@keyframes gate-shake`

## Guidelines
- Store the PIN server-side in an env var, not in client code
- Never hide the gate in `submitPin()` — only hide it on confirmed server success
- Call `showGate()` whenever the session expires or the user logs out
- Do not call `haptic()` on `pinError` — the red flash + shake is feedback enough; haptic on success only

## Notes
- Auto-submits on the 4th digit — no submit button needed
- Lockout sequence: 3 fails → 1s, next 3 → 1s, next 3 → 2s, 3s, 5s, 8s, 13s…
- `void dotsEl.offsetWidth` forces a reflow so the shake animation re-triggers on repeated wrong PINs
- `padLocked` guard prevents race conditions if the user taps during the error animation
- Works in PWA standalone mode — no keyboard, no native popups
- Reference implementation: [palantir](https://github.com/nickagliano/palantir)
