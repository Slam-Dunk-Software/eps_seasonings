# Seasoning: Haptics

## What it does
Adds tactile haptic feedback on iOS (iPhone/iPad) for user-initiated actions in EPS web apps and PWAs. Uses the iOS Taptic Engine via a hidden `<input type="checkbox" switch>` element — clicking the label programmatically fires a native haptic pulse.

## Why this works
iOS does not support the standard `navigator.vibrate()` API. However, iOS 17.4+ introduced a native `switch` input type that fires the Taptic Engine when toggled. By hiding one offscreen and clicking it from JS, any user action can trigger a haptic without a native app or special entitlements.

## When to apply
Any EPS web app or PWA with user-initiated actions: button presses, form submissions, command palette buttons, send actions, confirmations.

## Implementation

### 1. Add the hidden trigger element to the HTML `<body>`
```html
<!-- iOS haptic trigger: clicking this label toggles a switch input, firing the Taptic Engine -->
<label id="haptic-trigger" for="haptic-input" style="position:fixed;opacity:0;pointer-events:none;z-index:-1">
  <input type="checkbox" switch="" id="haptic-input" style="all:initial;appearance:auto">
</label>
```

### 2. Add the `haptic()` helper in your JS
```js
const hapticTrigger = document.getElementById('haptic-trigger');
function haptic() { hapticTrigger.click(); }
```

### 3. Call `haptic()` on user-initiated actions
```js
// Example: send button
sendBtn.addEventListener('click', () => {
  haptic();
  // ... rest of send logic
});

// Example: command palette button
function sendCmd(cmd) {
  haptic();
  send({ type: 'data', data: cmd });
}
```

## Guidelines
- Call `haptic()` on actions the user *initiates* — button taps, sends, confirmations
- Do not call on passive events — incoming messages, auto-updates, background activity
- One pulse per action is enough — do not chain multiple `haptic()` calls for a single interaction
- On non-iOS browsers this is a no-op (the click fires but has no haptic effect)

## Notes
- Requires iOS 17.4+ for the `switch` input type to be recognized
- Works in both Safari and PWA standalone mode
- The element must be in the DOM before the first haptic call — add it at body open, not dynamically
