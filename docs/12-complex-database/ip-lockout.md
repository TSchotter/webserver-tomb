---
layout: default
title: IP-Based Login Lockout
nav_order: 3
parent: Complex Database Integration
nav_exclude: true
---

# IP-Based Login Lockout

In this subchapter, we'll implement IP-based login attempt tracking to prevent brute-force attacks. When an IP address makes too many failed login attempts, it will be temporarily locked out from attempting to log in.

## What You'll Learn

- Creating a login attempts table
- Tracking login attempts by IP address
- Implementing IP-based lockout after too many failed attempts
- Cleaning up old login attempt records
- Integrating lockout checks into the authentication flow

## Prerequisites

Before starting, make sure you have:
- Completed [Password Requirements and Encryption](password-encryption.md)
- The authentication system from the previous subchapter
- Understanding of how IP addresses work

## Step 1: Update the Database Module

Update `database.js` to include a table for tracking login attempts:

```javascript
// database.js
const Database = require('better-sqlite3');
const path = require('path');

const dbPath = path.join(__dirname, 'app.db');
const db = new Database(dbPath);

// Enable foreign keys
db.pragma('foreign_keys = ON');

// Create users table
db.exec(`
  CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    last_login DATETIME
  )
`);

// Create login_attempts table for tracking failed login attempts by IP
db.exec(`
  CREATE TABLE IF NOT EXISTS login_attempts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    ip_address TEXT NOT NULL,
    attempt_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    success INTEGER DEFAULT 0,
    username TEXT
  )
`);

// Create index for faster lookups on IP address
db.exec(`
  CREATE INDEX IF NOT EXISTS idx_login_attempts_ip 
  ON login_attempts(ip_address, attempt_time)
`);

module.exports = db;
```

**What this adds:**
- `login_attempts` table to track every login attempt
- `ip_address`: The IP address that made the attempt
- `attempt_time`: When the attempt was made
- `success`: 0 for failed, 1 for successful
- `username`: The username that was attempted (for logging)
- Index on IP address and time for fast lookups

## Step 2: Create Login Tracker Module

Create `modules/login-tracker.js` to track and limit login attempts by IP address:

```javascript
// modules/login-tracker.js
const db = require('../database');

// Configuration
const MAX_ATTEMPTS = 5;           // Maximum failed attempts allowed
const LOCKOUT_DURATION = 15 * 60 * 1000; // 15 minutes in milliseconds

/**
 * Records a login attempt (success or failure)
 * @param {string} ipAddress - The IP address of the client
 * @param {boolean} success - Whether the login was successful
 * @param {string} username - The username attempted (optional)
 */
function recordAttempt(ipAddress, success, username = null) {
  const stmt = db.prepare(`
    INSERT INTO login_attempts (ip_address, success, username)
    VALUES (?, ?, ?)
  `);
  
  stmt.run(ipAddress, success ? 1 : 0, username);
}

/**
 * Checks if an IP address is currently locked out
 * @param {string} ipAddress - The IP address to check
 * @returns {Object} - { locked: boolean, remainingTime: number, attempts: number }
 */
function checkLockout(ipAddress) {
  const cutoffTime = Date.now() - LOCKOUT_DURATION;
  
  // Get all failed attempts in the lockout window
  const stmt = db.prepare(`
    SELECT COUNT(*) as count, MAX(attempt_time) as last_attempt
    FROM login_attempts
    WHERE ip_address = ? 
      AND success = 0
      AND datetime(attempt_time) > datetime(?, 'unixepoch')
  `);
  
  const result = stmt.get(ipAddress, cutoffTime / 1000);
  
  if (result.count >= MAX_ATTEMPTS) {
    // Calculate remaining lockout time
    const lastAttempt = new Date(result.last_attempt).getTime();
    const lockoutEnds = lastAttempt + LOCKOUT_DURATION;
    const remainingTime = Math.max(0, lockoutEnds - Date.now());
    
    return {
      locked: true,
      remainingTime: remainingTime,
      attempts: result.count
    };
  }
  
  return {
    locked: false,
    remainingTime: 0,
    attempts: result.count
  };
}

/**
 * Clears old login attempts (cleanup function)
 * Removes attempts older than the lockout duration
 */
function cleanupOldAttempts() {
  const cutoffTime = Date.now() - LOCKOUT_DURATION;
  
  const stmt = db.prepare(`
    DELETE FROM login_attempts
    WHERE datetime(attempt_time) < datetime(?, 'unixepoch')
  `);
  
  const result = stmt.run(cutoffTime / 1000);
  return result.changes;
}

// Clean up old attempts every hour
setInterval(() => {
  const deleted = cleanupOldAttempts();
  if (deleted > 0) {
    console.log(`Cleaned up ${deleted} old login attempt(s)`);
  }
}, 60 * 60 * 1000);

module.exports = {
  recordAttempt,
  checkLockout,
  cleanupOldAttempts
};
```

## Step 3: Update Authentication Middleware

Update `modules/auth-middleware.js` to include lockout checking:

```javascript
// modules/auth-middleware.js
const loginTracker = require('./login-tracker');

/**
 * Middleware to check if user is authenticated
 * Returns 401 if not authenticated
 */
function requireAuth(req, res, next) {
  if (req.session && req.session.userId) {
    next();
  } else {
    res.status(401).json({ error: 'Authentication required' });
  }
}

/**
 * Middleware to check IP-based login lockout
 * Should be used before login route handlers
 */
function checkLoginLockout(req, res, next) {
  const ipAddress = getClientIP(req);
  const lockoutStatus = loginTracker.checkLockout(ipAddress);
  
  if (lockoutStatus.locked) {
    const minutesRemaining = Math.ceil(lockoutStatus.remainingTime / (60 * 1000));
    return res.status(429).json({
      error: 'Too many failed login attempts',
      message: `IP address locked. Please try again in ${minutesRemaining} minute(s).`,
      remainingTime: lockoutStatus.remainingTime
    });
  }
  
  next();
}

/**
 * Helper function to get client IP address
 * Handles proxies and various connection types
 */
function getClientIP(req) {
  return req.ip || 
         req.headers['x-forwarded-for']?.split(',')[0] || 
         req.connection.remoteAddress || 
         'unknown';
}

module.exports = {
  requireAuth,
  checkLoginLockout,
  getClientIP
};
```

## Step 4: Update Authentication Routes

Update `routes/auth.js` to use the login tracker:

```javascript
// routes/auth.js
const express = require('express');
const router = express.Router();
const db = require('../database');
const { validatePassword, hashPassword, comparePassword } = require('../modules/password-utils');
const loginTracker = require('../modules/login-tracker');
const { checkLoginLockout, getClientIP } = require('../modules/auth-middleware');

/**
 * POST /register - Register a new user
 */
router.post('/register', async (req, res) => {
  try {
    const { username, password } = req.body;
    
    // Validate input
    if (!username || !password) {
      return res.status(400).json({ 
        error: 'Username and password are required' 
      });
    }
    
    // Validate password requirements
    const validation = validatePassword(password);
    if (!validation.valid) {
      return res.status(400).json({
        error: 'Password does not meet requirements',
        errors: validation.errors
      });
    }
    
    // Check if username already exists
    const existingUser = db.prepare('SELECT id FROM users WHERE username = ?').get(username);
    if (existingUser) {
      return res.status(409).json({ 
        error: 'Username already exists' 
      });
    }
    
    // Hash the password before storing
    const passwordHash = await hashPassword(password);
    
    // Insert new user into database
    const stmt = db.prepare('INSERT INTO users (username, password_hash) VALUES (?, ?)');
    const result = stmt.run(username, passwordHash);
    
    res.status(201).json({
      message: 'User registered successfully',
      userId: result.lastInsertRowid
    });
    
  } catch (error) {
    console.error('Registration error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

/**
 * POST /login - Authenticate user
 * Now includes lockout checking and attempt tracking
 */
router.post('/login', checkLoginLockout, async (req, res) => {
  try {
    const { username, password } = req.body;
    const ipAddress = getClientIP(req);
    
    // Validate input
    if (!username || !password) {
      loginTracker.recordAttempt(ipAddress, false, username);
      return res.status(400).json({ 
        error: 'Username and password are required' 
      });
    }
    
    // Find user by username
    const user = db.prepare('SELECT * FROM users WHERE username = ?').get(username);
    
    if (!user) {
      // Record failed attempt (user doesn't exist)
      loginTracker.recordAttempt(ipAddress, false, username);
      return res.status(401).json({ 
        error: 'Invalid username or password' 
      });
    }
    
    // Compare entered password with stored hash
    const passwordMatch = await comparePassword(password, user.password_hash);
    
    if (!passwordMatch) {
      // Record failed attempt (wrong password)
      loginTracker.recordAttempt(ipAddress, false, username);
      return res.status(401).json({ 
        error: 'Invalid username or password' 
      });
    }
    
    // Successful login
    loginTracker.recordAttempt(ipAddress, true, username);
    
    // Update last login time
    db.prepare('UPDATE users SET last_login = CURRENT_TIMESTAMP WHERE id = ?')
      .run(user.id);
    
    // Create session
    req.session.userId = user.id;
    req.session.username = user.username;
    req.session.isLoggedIn = true;
    
    res.json({
      message: 'Login successful',
      user: {
        id: user.id,
        username: user.username
      }
    });
    
  } catch (error) {
    console.error('Login error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

/**
 * POST /logout - Logout user
 */
router.post('/logout', (req, res) => {
  req.session.destroy((err) => {
    if (err) {
      console.error('Logout error:', err);
      return res.status(500).json({ error: 'Error logging out' });
    }
    res.json({ message: 'Logged out successfully' });
  });
});

/**
 * GET /me - Get current user info (requires authentication)
 */
router.get('/me', (req, res) => {
  if (!req.session || !req.session.userId) {
    return res.status(401).json({ error: 'Not authenticated' });
  }
  
  const user = db.prepare('SELECT id, username, created_at, last_login FROM users WHERE id = ?')
    .get(req.session.userId);
  
  if (!user) {
    return res.status(404).json({ error: 'User not found' });
  }
  
  res.json({ user });
});

module.exports = router;
```

## How IP-Based Lockout Works

1. **Record every attempt**: Every login attempt (success or failure) is recorded with the IP address
2. **Count failed attempts**: When checking for lockout, count failed attempts in the last 15 minutes
3. **Lock if threshold reached**: If there are 5 or more failed attempts, the IP is locked out
4. **Time-based expiration**: The lockout expires 15 minutes after the last failed attempt
5. **Automatic cleanup**: Old attempts are periodically deleted to keep the database small

### Example Timeline

```
Time  | Action                    | Failed Attempts | Status
------|---------------------------|-----------------|--------
10:00 | Failed login #1           | 1               | Allowed
10:01 | Failed login #2           | 2               | Allowed
10:02 | Failed login #3           | 3               | Allowed
10:03 | Failed login #4           | 4               | Allowed
10:04 | Failed login #5           | 5               | LOCKED
10:05 | Attempt to login          | 5               | Rejected (locked)
10:19 | Attempt to login          | 0 (expired)     | Allowed
```

## Step 5: Testing IP-Based Lockout

### Test Normal Login

```bash
# Successful login (should work)
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","password":"Test123!@#"}'
```

### Test Lockout by Making Multiple Failed Attempts

```bash
# Try wrong password 5 times to trigger lockout
for i in {1..5}; do
  echo "Attempt $i:"
  curl -X POST http://localhost:3000/api/auth/login \
    -H "Content-Type: application/json" \
    -d '{"username":"testuser","password":"wrong"}'
  echo ""
done
```

**After 5 failed attempts, the 6th attempt will return:**
```json
{
  "error": "Too many failed login attempts",
  "message": "IP address locked. Please try again in 15 minute(s).",
  "remainingTime": 900000
}
```

### Test Lockout Persistence

```bash
# Try to login again immediately (should still be locked)
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","password":"Test123!@#"}'
```

**Expected response:**
```json
{
  "error": "Too many failed login attempts",
  "message": "IP address locked. Please try again in 15 minute(s).",
  "remainingTime": 899000
}
```

### Verify Attempts Are Recorded

You can check the database to see recorded attempts:

```bash
sqlite3 app.db "SELECT ip_address, attempt_time, success, username FROM login_attempts ORDER BY attempt_time DESC LIMIT 10;"
```

## Understanding IP Address Detection

The `getClientIP` function handles different scenarios:

1. **Direct connection**: Uses `req.ip` (set by Express trust proxy)
2. **Behind proxy**: Checks `x-forwarded-for` header (first IP in chain)
3. **Fallback**: Uses `req.connection.remoteAddress`
4. **Unknown**: Returns 'unknown' if all methods fail

**Important:** If your server is behind a reverse proxy (like nginx), you may need to configure Express to trust the proxy:

```javascript
// In server.js, before routes
app.set('trust proxy', true); // Trust first proxy
```

This ensures `req.ip` correctly reflects the client's IP address.

## Configuration Options

You can adjust the lockout behavior by changing constants in `login-tracker.js`:

```javascript
const MAX_ATTEMPTS = 5;           // Increase to allow more attempts
const LOCKOUT_DURATION = 15 * 60 * 1000; // Change lockout duration
```

**Common configurations:**
- **Strict**: `MAX_ATTEMPTS = 3`, `LOCKOUT_DURATION = 30 * 60 * 1000` (3 attempts, 30 min lockout)
- **Moderate**: `MAX_ATTEMPTS = 5`, `LOCKOUT_DURATION = 15 * 60 * 1000` (5 attempts, 15 min lockout) - **Default**
- **Lenient**: `MAX_ATTEMPTS = 10`, `LOCKOUT_DURATION = 5 * 60 * 1000` (10 attempts, 5 min lockout)

## Limitations of IP-Based Lockout

**Important considerations:**

1. **Shared IP addresses**: Users behind the same NAT/proxy share an IP
   - If one user gets locked out, others on the same network are also locked out
   - This is a trade-off for security

2. **IP spoofing**: Attackers can change their IP address
   - IP lockout is one layer of security, not a complete solution
   - Should be combined with other security measures

3. **Legitimate lockouts**: Legitimate users can accidentally trigger lockouts
   - Consider adding account recovery mechanisms
   - Provide clear error messages about lockout status

4. **IPv6**: IPv6 addresses are longer and more complex
   - The system handles them the same way, but be aware of address format

## Best Practices

1. **Log successful attempts too**: Helps with security auditing
2. **Clean up old records**: Prevents database bloat (already implemented)
3. **Monitor lockout patterns**: Watch for unusual activity
4. **Combine with other security**: Use alongside rate limiting, CAPTCHA, etc.
5. **Provide user feedback**: Tell users how long they're locked out

## Advanced: Per-Account Lockout

You could extend this system to also lock individual accounts:

```javascript
// Add to users table
db.exec(`
  ALTER TABLE users ADD COLUMN failed_login_count INTEGER DEFAULT 0;
  ALTER TABLE users ADD COLUMN account_locked_until DATETIME;
`);
```

This would track failed attempts per username, not just per IP, providing an additional layer of security.

## Summary

You've now learned how to:
- Track login attempts by IP address
- Implement IP-based lockout after too many failed attempts
- Clean up old login attempt records automatically
- Integrate lockout checks into the authentication flow
- Handle IP address detection behind proxies

Your authentication system now has protection against brute-force attacks!

---

**[Previous: Complex Database Integration](index.md)** | **[Next: Putting the Server on the Map](../11-putting-server-on-map/index.md)**

