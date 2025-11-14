---
layout: default
title: User Authentication with Database
nav_order: 2
parent: Complex Database Integration
nav_exclude: true
---

# User Authentication with Database

In this subchapter, we'll integrate the password utilities from the previous subchapter with a SQLite3 database. You'll learn how to create a users table, store encrypted passwords, and authenticate users by comparing passwords against the database.

## What You'll Learn

- Creating a users table in SQLite3
- Storing encrypted passwords in the database
- Building registration and login routes
- Comparing passwords during authentication
- Creating authentication middleware

## Prerequisites

Before starting, make sure you have:
- Completed [Password Requirements and Encryption](password-encryption.md)
- The `modules/password-utils.js` module from the previous subchapter
- Basic understanding of [SQLite3 with Node.js Express](../9-sqlite3-server/sqlite3-nodejs-express.md)

## Step 1: Install Additional Dependencies

Add the required packages for Express and SQLite3:

```bash
npm install express express-session better-sqlite3
```

**Package explanations:**
- **express**: Web framework for building routes
- **express-session**: Session management for maintaining login state
- **better-sqlite3**: SQLite3 database driver

## Step 2: Create the Database Module

Create `database.js` to set up your SQLite3 database with a users table:

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

module.exports = db;
```

**What this does:**
- Creates a connection to `app.db` (creates the file if it doesn't exist)
- Creates a `users` table with:
  - `id`: Primary key, auto-incrementing
  - `username`: Unique username (no duplicates allowed)
  - `password_hash`: Stores the bcrypt hash (never plain text!)
  - `created_at`: Timestamp when account was created
  - `last_login`: Timestamp of last successful login

## Step 3: Create Authentication Middleware

Create `modules/auth-middleware.js` for reusable authentication checks:

```javascript
// modules/auth-middleware.js

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

module.exports = {
  requireAuth
};
```

This middleware will be used to protect routes that require login.

## Step 4: Create Authentication Routes

Create `routes/auth.js` to handle registration, login, and logout:

```javascript
// routes/auth.js
const express = require('express');
const router = express.Router();
const db = require('../database');
const { validatePassword, hashPassword, comparePassword } = require('../modules/password-utils');

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
 */
router.post('/login', async (req, res) => {
  try {
    const { username, password } = req.body;
    
    // Validate input
    if (!username || !password) {
      return res.status(400).json({ 
        error: 'Username and password are required' 
      });
    }
    
    // Find user by username
    const user = db.prepare('SELECT * FROM users WHERE username = ?').get(username);
    
    if (!user) {
      // Don't reveal if username exists (security best practice)
      return res.status(401).json({ 
        error: 'Invalid username or password' 
      });
    }
    
    // Compare entered password with stored hash
    const passwordMatch = await comparePassword(password, user.password_hash);
    
    if (!passwordMatch) {
      return res.status(401).json({ 
        error: 'Invalid username or password' 
      });
    }
    
    // Successful login - update last login time
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

## Step 5: Create the Main Server

Create `server.js` to tie everything together:

```javascript
// server.js
const express = require('express');
const session = require('express-session');
const SQLiteStore = require('./sqlite-session-store'); // From Chapter 10
const path = require('path');
const authRoutes = require('./routes/auth');
const { requireAuth } = require('./modules/auth-middleware');

const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Session configuration with SQLite store (from Chapter 10)
const sessionStore = new SQLiteStore({
  db: path.join(__dirname, 'sessions.db'),
  table: 'sessions'
});

app.use(session({
  store: sessionStore,
  secret: process.env.SESSION_SECRET || 'change-this-secret-in-production',
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: false, // Set to true if using HTTPS
    maxAge: 24 * 60 * 60 * 1000 // 24 hours
  }
}));

// Routes
app.use('/api/auth', authRoutes);

// Protected route example
app.get('/api/protected', requireAuth, (req, res) => {
  res.json({
    message: 'This is a protected route',
    user: {
      id: req.session.userId,
      username: req.session.username
    }
  });
});

// Health check route
app.get('/api/health', (req, res) => {
  res.json({ status: 'ok' });
});

// Start server
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
  console.log(`Register: POST http://localhost:${PORT}/api/auth/register`);
  console.log(`Login: POST http://localhost:${PORT}/api/auth/login`);
});

// Graceful shutdown
process.on('SIGINT', () => {
  console.log('\nShutting down gracefully...');
  sessionStore.close();
  process.exit(0);
});
```

**Note:** You'll need to copy the `SQLiteStore` class from Chapter 10's `advanced-modules.md` into a file called `sqlite-session-store.js` in your project root.

## Understanding the Authentication Flow

### Registration Flow

1. User submits username and password
2. Server validates password meets requirements
3. Server checks if username already exists
4. Server hashes the password with bcrypt
5. Server stores username and password hash in database
6. Server returns success response

### Login Flow

1. User submits username and password
2. Server finds user by username in database
3. Server compares submitted password with stored hash using `comparePassword`
4. If passwords match:
   - Update `last_login` timestamp
   - Create session with user ID and username
   - Return success response
5. If passwords don't match:
   - Return error (don't reveal which was wrong - username or password)

### Why We Don't Reveal Specific Errors

Notice that both "user not found" and "wrong password" return the same error message: "Invalid username or password". This is a security best practice because:

- Prevents username enumeration (attackers can't check if usernames exist)
- Doesn't give attackers hints about which part was wrong
- Slows down brute-force attacks

## Step 6: Testing the Authentication System

### Test Registration

```bash
# Register a new user with a valid password
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","password":"Test123!@#"}'
```

**Expected response:**
```json
{
  "message": "User registered successfully",
  "userId": 1
}
```

**Test with invalid password:**
```bash
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser2","password":"weak"}'
```

**Expected response:**
```json
{
  "error": "Password does not meet requirements",
  "errors": [
    "Password must be at least 8 characters long",
    "Password must contain at least one uppercase letter",
    "Password must contain at least one number",
    "Password must contain at least one special character"
  ]
}
```

**Test with duplicate username:**
```bash
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","password":"Another123!@#"}'
```

**Expected response:**
```json
{
  "error": "Username already exists"
}
```

### Test Login

```bash
# Successful login
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","password":"Test123!@#"}' \
  -c cookies.txt
```

**Expected response:**
```json
{
  "message": "Login successful",
  "user": {
    "id": 1,
    "username": "testuser"
  }
}
```

**Test with wrong password:**
```bash
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","password":"wrong"}'
```

**Expected response:**
```json
{
  "error": "Invalid username or password"
}
```

**Test with non-existent username:**
```bash
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"nonexistent","password":"Test123!@#"}'
```

**Expected response:**
```json
{
  "error": "Invalid username or password"
}
```

### Test Protected Route

```bash
# Get current user info (requires session cookie from login)
curl -X GET http://localhost:3000/api/auth/me \
  -b cookies.txt
```

**Expected response (if logged in):**
```json
{
  "user": {
    "id": 1,
    "username": "testuser",
    "created_at": "2024-01-15 10:30:00",
    "last_login": "2024-01-15 11:45:00"
  }
}
```

**Expected response (if not logged in):**
```json
{
  "error": "Not authenticated"
}
```

### Test Logout

```bash
curl -X POST http://localhost:3000/api/auth/logout \
  -b cookies.txt
```

**Expected response:**
```json
{
  "message": "Logged out successfully"
}
```

## Understanding Password Comparison

When a user logs in, here's what happens:

1. **User submits password**: `"MyPassword123!"`
2. **Server retrieves hash**: `"$2b$10$N9qo8uLOickgx2ZMRZoMye..."`
3. **bcrypt compares**: 
   - Takes the submitted password
   - Extracts the salt from the stored hash
   - Hashes the submitted password with that salt
   - Compares the result with the stored hash
4. **Returns true/false**: Whether they match

**Key point:** bcrypt's `comparePassword` function handles all of this automatically. You never need to manually extract salts or compare hashes - bcrypt does it securely for you.

## Database Security Considerations

1. **Never store plain text passwords**: Always hash before storing
2. **Use prepared statements**: Prevents SQL injection (already done in examples)
3. **Unique usernames**: The `UNIQUE` constraint prevents duplicate accounts
4. **Index on username**: SQLite automatically creates an index for UNIQUE columns, making lookups fast

## Common Issues and Solutions

### Issue: "Username already exists" even for new users

**Solution:** Check if you're reusing the same database file. Delete `app.db` to start fresh, or check existing users:
```bash
sqlite3 app.db "SELECT username FROM users;"
```

### Issue: Passwords don't match even when correct

**Solution:** Make sure you're using `await` with async functions:
```javascript
// CORRECT
const match = await comparePassword(password, hash);

// WRONG - returns a Promise, not a boolean
const match = comparePassword(password, hash);
```

### Issue: Session not persisting

**Solution:** Make sure you're sending cookies with requests:
```bash
# Save cookies
curl ... -c cookies.txt

# Use cookies
curl ... -b cookies.txt
```

## Summary

You've now learned how to:
- Create a users table in SQLite3
- Store encrypted passwords in the database
- Register new users with password validation
- Authenticate users by comparing passwords
- Create sessions for logged-in users
- Protect routes with authentication middleware

In the next subchapter, we'll add IP-based login attempt tracking to prevent brute-force attacks.

---

**[Previous: Complex Database Integration](index.md)** | **[Next: Putting the Server on the Map](../11-putting-server-on-map/index.md)**

