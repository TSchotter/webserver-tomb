---
layout: default
title: User Authentication with Database
nav_order: 2
parent: Complex Database Integration
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
    res.status(401).send(`
      <!DOCTYPE html>
      <html>
      <head><title>Authentication Required</title></head>
      <body>
        <h1>Authentication Required</h1>
        <p>You must be logged in to access this page.</p>
        <p><a href="/api/auth/login">Login here</a></p>
        <p><a href="/">← Back to Home</a></p>
      </body>
      </html>
    `);
  }
}

module.exports = {
  requireAuth
};
```

This middleware will be used to protect routes that require login.

## Step 4: Create Static HTML Pages

Create a `public` directory and add the following HTML files for our authentication pages. This can be replaced with the handlebars middleware if nessessary.

### Create the Public Directory

```bash
mkdir public
```

### Registration Page (`public/register.html`)

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Register</title>
</head>
<body>
  <h1>Register New User</h1>
  <div>
    <strong>Password Requirements:</strong>
    <ul>
      <li>At least 8 characters long</li>
      <li>At least one uppercase letter</li>
      <li>At least one lowercase letter</li>
      <li>At least one number</li>
      <li>At least one special character</li>
    </ul>
  </div>
  <form method="POST" action="/api/auth/register">
    <div>
      <label for="username">Username:</label>
      <input type="text" id="username" name="username" required>
    </div>
    <div>
      <label for="password">Password:</label>
      <input type="password" id="password" name="password" required>
    </div>
    <button type="submit">Register</button>
  </form>
  <p><a href="/api/auth/login">Already have an account? Login here</a></p>
  <p><a href="/">← Back to Home</a></p>
</body>
</html>
```

### Login Page (`public/login.html`)

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Login</title>
</head>
<body>
  <h1>Login</h1>
  <form method="POST" action="/api/auth/login">
    <div>
      <label for="username">Username:</label>
      <input type="text" id="username" name="username" required>
    </div>
    <div>
      <label for="password">Password:</label>
      <input type="password" id="password" name="password" required>
    </div>
    <button type="submit">Login</button>
  </form>
  <p><a href="/api/auth/register">Don't have an account? Register here</a></p>
  <p><a href="/">← Back to Home</a></p>
</body>
</html>
```

### Success/Error Pages

Create these simple pages for displaying messages:

**Registration Success (`public/register-success.html`)**

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Registration Successful</title>
</head>
<body>
  <h1>Registration Successful!</h1>
  <p id="message">User has been registered successfully.</p>
  <p><a href="/api/auth/login">Login here</a></p>
  <p><a href="/">← Back to Home</a></p>
</body>
</html>
```

**Login Success (`public/login-success.html`)**

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Login Successful</title>
</head>
<body>
  <h1>Login Successful!</h1>
  <p id="message">Welcome!</p>
  <p><a href="/api/auth/me">View your profile</a></p>
  <p><a href="/api/auth/logout">Logout</a></p>
  <p><a href="/">← Back to Home</a></p>
</body>
</html>
```

**Error Page (`public/error.html`)**

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Error</title>
</head>
<body>
  <h1>Error</h1>
  <p id="message">An error occurred.</p>
  <a href="/" id="back-link">← Back to Home</a>
</body>
</html>
```

**User Profile Page (`public/profile.html`)**

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>User Profile</title>
</head>
<body>
  <h1>User Profile</h1>
  <div id="user-info">
    <div><strong>Loading...</strong></div>
  </div>
  <p><a href="/api/auth/logout">Logout</a></p>
  <p><a href="/">← Back to Home</a></p>
</body>
</html>
```

**Logged Out Page (`public/logged-out.html`)**

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Logged Out</title>
</head>
<body>
  <h1>Logged Out Successfully</h1>
  <p>You have been logged out.</p>
  <p><a href="/api/auth/login">Login again</a></p>
  <p><a href="/">← Back to Home</a></p>
</body>
</html>
```

## Step 5: Create Authentication Routes

Create `routes/auth.js` to handle registration, login, and logout. The routes will serve static HTML files and handle form submissions:

```javascript
// routes/auth.js
const express = require('express');
const router = express.Router();
const path = require('path');
const db = require('../database');
const { validatePassword, hashPassword, comparePassword } = require('../modules/password-utils');

/**
 * GET /register - Show registration form
 */
router.get('/register', (req, res) => {
  res.sendFile(path.join(__dirname, '../public/register.html'));
});

/**
 * POST /register - Register a new user
 */
router.post('/register', async (req, res) => {
  try {
    const { username, password } = req.body;
    
    // Validate input
    if (!username || !password) {
      return res.redirect('/api/auth/register?error=' + encodeURIComponent('Username and password are required'));
    }
    
    // Validate password requirements
    const validation = validatePassword(password);
    if (!validation.valid) {
      const errorsText = validation.errors.join(', ');
      return res.redirect('/api/auth/register?error=' + encodeURIComponent('Password does not meet requirements: ' + errorsText));
    }
    
    // Check if username already exists
    const existingUser = db.prepare('SELECT id FROM users WHERE username = ?').get(username);
    if (existingUser) {
      return res.redirect('/api/auth/register?error=' + encodeURIComponent('Username already exists. Please choose a different username.'));
    }
    
    // Hash the password before storing
    const passwordHash = await hashPassword(password);
    
    // Insert new user into database
    const stmt = db.prepare('INSERT INTO users (username, password_hash) VALUES (?, ?)');
    const result = stmt.run(username, passwordHash);
    
    // Redirect to success page with username
    res.redirect(`/public/register-success.html?username=${encodeURIComponent(username)}&userId=${result.lastInsertRowid}`);
    
  } catch (error) {
    console.error('Registration error:', error);
    res.redirect('/public/error.html?message=' + encodeURIComponent('An internal server error occurred. Please try again later.') + '&back=/api/auth/register');
  }
});

/**
 * GET /login - Show login form
 */
router.get('/login', (req, res) => {
  res.sendFile(path.join(__dirname, '../public/login.html'));
});

/**
 * POST /login - Authenticate user
 */
router.post('/login', async (req, res) => {
  try {
    const { username, password } = req.body;
    
    // Validate input
    if (!username || !password) {
      return res.redirect('/api/auth/login?error=' + encodeURIComponent('Username and password are required'));
    }
    
    // Find user by username
    const user = db.prepare('SELECT * FROM users WHERE username = ?').get(username);
    
    if (!user) {
      // Don't reveal if username exists (security best practice)
      return res.redirect('/api/auth/login?error=' + encodeURIComponent('Invalid username or password'));
    }
    
    // Compare entered password with stored hash
    const passwordMatch = await comparePassword(password, user.password_hash);
    
    if (!passwordMatch) {
      return res.redirect('/api/auth/login?error=' + encodeURIComponent('Invalid username or password'));
    }
    
    // Successful login - update last login time
    db.prepare('UPDATE users SET last_login = CURRENT_TIMESTAMP WHERE id = ?')
      .run(user.id);
    
    // Create session
    req.session.userId = user.id;
    req.session.username = user.username;
    req.session.isLoggedIn = true;
    
    // Redirect to success page
    res.redirect(`/public/login-success.html?username=${encodeURIComponent(user.username)}`);
    
  } catch (error) {
    console.error('Login error:', error);
    res.redirect('/public/error.html?message=' + encodeURIComponent('An internal server error occurred. Please try again later.') + '&back=/api/auth/login');
  }
});

/**
 * GET /logout - Logout user (GET version for easy link access)
 */
router.get('/logout', (req, res) => {
  req.session.destroy((err) => {
    if (err) {
      console.error('Logout error:', err);
      return res.redirect('/public/error.html?message=' + encodeURIComponent('An error occurred while logging out.') + '&back=/');
    }
    res.redirect('/public/logged-out.html');
  });
});

/**
 * POST /logout - Logout user (POST version)
 */
router.post('/logout', (req, res) => {
  req.session.destroy((err) => {
    if (err) {
      console.error('Logout error:', err);
      return res.redirect('/public/error.html?message=' + encodeURIComponent('An error occurred while logging out.') + '&back=/');
    }
    res.redirect('/public/logged-out.html');
  });
});

/**
 * GET /me - Get current user info (requires authentication)
 */
router.get('/me', (req, res) => {
  if (!req.session || !req.session.userId) {
    return res.redirect('/public/error.html?message=' + encodeURIComponent('You must be logged in to view this page.') + '&back=/api/auth/login');
  }
  
  const user = db.prepare('SELECT id, username, created_at, last_login FROM users WHERE id = ?')
    .get(req.session.userId);
  
  if (!user) {
    return res.redirect('/public/error.html?message=' + encodeURIComponent('User not found in database.') + '&back=/');
  }
  
  // Pass user data as query parameters to the profile page
  const params = new URLSearchParams({
    id: user.id,
    username: user.username,
    created_at: user.created_at || 'N/A',
    last_login: user.last_login || 'Never'
  });
  
  res.redirect(`/public/profile.html?${params.toString()}`);
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
app.use(express.static('public'));

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

// Protected route example (doing this manually by sending)
app.get('/api/protected', requireAuth, (req, res) => {
  res.send(`Protected route that needs authentication. User: ${req.session.username} ID: ${req.session.userId}`);
});

// Start server
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

// Graceful shutdown, this will help the session to close the db gracefully since we're now using it.
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

---

**[Previous: Complex Database Integration](index.md)** | **[Next: Putting the Server on the Map](../11-putting-server-on-map/index.md)**


