---
layout: default
title: Advanced Module Patterns
nav_order: 2
parent: Creating Your Own Node.js Modules
nav_exclude: true
---

# Advanced Module Patterns

Now that you understand the basics of modules, let's explore more advanced patterns including class-based modules, module factories, and creating modules that work with Express middleware.

## Creating Custom Express Middleware Modules

Middleware modules are perfect for reusable Express functionality:

```javascript
// auth-middleware.js

function requireAuth(req, res, next) {
  if (req.session && req.session.userId) {
    // User is authenticated, continue to next middleware
    next();
  } else {
    // User is not authenticated, redirect to login
    res.redirect('/login');
  }
}

function requireAdmin(req, res, next) {
  if (req.session && req.session.role === 'admin') {
    next();
  } else {
    // User is not authorized, redirect to access denied page
    res.redirect('/access-denied');
  }
}

module.exports = {
  requireAuth: requireAuth,
  requireAdmin: requireAdmin
};
```

In the above, these will be used as middleware functions in the app.get function calls. They will happen *first*, then potentially call "next" when conditions are right. 

Use it with Handlebars:

```javascript
// server.js
const express = require('express');
const exphbs = require('express-handlebars');
const { requireAuth, requireAdmin } = require('./auth-middleware');

const app = express();

// Set up Handlebars as the view engine
app.engine('hbs', exphbs.engine({ extname: '.hbs' }));
app.set('view engine', 'hbs');
app.set('views', './views');

// Protected route - requires authentication
app.get('/profile', requireAuth, (req, res) => {
  res.render('profile', { 
    user: req.session.userId,
    username: req.session.username 
  });
});

// Admin route - requires admin role
app.get('/admin', requireAdmin, (req, res) => {
  res.render('admin', { 
    message: 'Admin panel',
    user: req.session.userId 
  });
});

// Login page
app.get('/login', (req, res) => {
  res.render('login');
});

// Access denied page
app.get('/access-denied', (req, res) => {
  res.render('access-denied');
});
```



## Creating a Custom Session Store Module

Let's create a complete example by building a custom SQLite session store as a module:

```javascript
// sqlite-session-store.js
const { Store } = require('express-session');
const Database = require('better-sqlite3');
const path = require('path');

class SQLiteStore extends Store {
  constructor(options = {}) {
    super(options);
    
    // Use provided database or default to sessions.db
    const dbPath = options.db || path.join(__dirname, 'sessions.db');
    this.db = new Database(dbPath);
    
    // Table name (default: sessions)
    this.table = options.table || 'sessions';
    
    // Create sessions table if it doesn't exist
    this.db.exec(`
      CREATE TABLE IF NOT EXISTS ${this.table} (
        sid TEXT PRIMARY KEY,
        sess TEXT NOT NULL,
        expire INTEGER NOT NULL
      )
    `);
    
    // Create index for faster expiration lookups
    this.db.exec(`
      CREATE INDEX IF NOT EXISTS idx_${this.table}_expire 
      ON ${this.table}(expire)
    `);
    
    // Clean up expired sessions periodically (every 15 minutes)
    this.cleanupInterval = setInterval(() => {
      this.cleanup();
    }, 15 * 60 * 1000);
  }
  
  // Get session by ID
  get(sid, callback) {
    const row = this.db.prepare(
      `SELECT sess FROM ${this.table} WHERE sid = ? AND expire > ?`
    ).get(sid, Date.now());
    
    if (row) {
      try {
        const session = JSON.parse(row.sess);
        callback(null, session);
      } catch (err) {
        callback(err);
      }
    } else {
      callback(null, null);
    }
  }
  
  // Save session
  set(sid, sess, callback) {
    const maxAge = sess.cookie?.maxAge;
    const expire = maxAge ? Date.now() + maxAge : Date.now() + (24 * 60 * 60 * 1000);
    
    const sessData = JSON.stringify(sess);
    
    try {
      this.db.prepare(
        `INSERT OR REPLACE INTO ${this.table} (sid, sess, expire) VALUES (?, ?, ?)`
      ).run(sid, sessData, expire);
      
      callback(null);
    } catch (err) {
      callback(err);
    }
  }
  
  // Destroy session
  destroy(sid, callback) {
    try {
      this.db.prepare(`DELETE FROM ${this.table} WHERE sid = ?`).run(sid);
      callback(null);
    } catch (err) {
      callback(err);
    }
  }
  
  // Get all sessions
  all(callback) {
    try {
      const rows = this.db.prepare(
        `SELECT sess FROM ${this.table} WHERE expire > ?`
      ).all(Date.now());
      
      const sessions = rows.map(row => JSON.parse(row.sess));
      callback(null, sessions);
    } catch (err) {
      callback(err);
    }
  }
  
  // Clean up expired sessions
  cleanup() {
    try {
      const result = this.db.prepare(
        `DELETE FROM ${this.table} WHERE expire <= ?`
      ).run(Date.now());
      
      if (result.changes > 0) {
        console.log(`Cleaned up ${result.changes} expired session(s)`);
      }
    } catch (err) {
      console.error('Error cleaning up sessions:', err);
    }
  }
  
  // Close the database connection
  close() {
    if (this.cleanupInterval) {
      clearInterval(this.cleanupInterval);
    }
    this.db.close();
  }
}

module.exports = SQLiteStore;
```

Use it:

```javascript
// server.js
const express = require('express');
const session = require('express-session');
const SQLiteStore = require('./sqlite-session-store');
const path = require('path');

const app = express();

const sessionStore = new SQLiteStore({
  db: path.join(__dirname, 'sessions.db'),
  table: 'sessions'
});

app.use(session({
  store: sessionStore,
  secret: 'your-secret-key',
  resave: false,
  saveUninitialized: false
}));

// Your routes here...
```

## Module Factory Pattern

Sometimes you want to create modules that return configured instances:

```javascript
// logger.js

function createLogger(level = 'info') {
  const levels = ['error', 'warn', 'info', 'debug'];
  const currentLevel = levels.indexOf(level);
  
  function log(messageLevel, message) {
    const messageLevelIndex = levels.indexOf(messageLevel);
    if (messageLevelIndex <= currentLevel) {
      const timestamp = new Date().toISOString();
      console.log(`[${timestamp}] [${messageLevel.toUpperCase()}] ${message}`);
    }
  }
  
  return {
    error: (msg) => log('error', msg),
    warn: (msg) => log('warn', msg),
    info: (msg) => log('info', msg),
    debug: (msg) => log('debug', msg)
  };
}

module.exports = createLogger;
```

Use it:

```javascript
// app.js
const createLogger = require('./logger');

// Create different loggers for different parts of your app
const appLogger = createLogger('info');
const debugLogger = createLogger('debug');

appLogger.info('Application started');
appLogger.error('Something went wrong');
debugLogger.debug('Detailed debug information');
```

## Creating a Router Module

Organize routes into separate modules:

```javascript
// routes/users.js
const express = require('express');
const router = express.Router();
const db = require('../database');

// GET all users
router.get('/', (req, res) => {
  try {
    const users = db.prepare('SELECT * FROM users').all();
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// GET user by ID
router.get('/:id', (req, res) => {
  try {
    const user = db.prepare('SELECT * FROM users WHERE id = ?').get(req.params.id);
    if (user) {
      res.json(user);
    } else {
      res.status(404).json({ error: 'User not found' });
    }
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// POST create user
router.post('/', (req, res) => {
  try {
    const { name, email } = req.body;
    const stmt = db.prepare('INSERT INTO users (name, email) VALUES (?, ?)');
    const result = stmt.run(name, email);
    res.status(201).json({ 
      id: result.lastInsertRowid,
      name,
      email 
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

Use it:

```javascript
// server.js
const express = require('express');
const userRoutes = require('./routes/users');

const app = express();

app.use(express.json());
app.use('/users', userRoutes); // All routes in users.js are prefixed with /users

app.listen(3000);
```

## Error Handling Module

Create a reusable error handling module:

```javascript
// error-handler.js

function handleError(err, req, res, next) {
  console.error('Error:', err);
  
  // Default error
  let status = 500;
  let message = 'Internal server error';
  
  // Handle specific error types
  if (err.code === 'SQLITE_CONSTRAINT') {
    status = 400;
    message = 'Database constraint violation';
  } else if (err.message && err.message.includes('UNIQUE constraint')) {
    status = 409;
    message = 'Resource already exists';
  } else if (err.status) {
    status = err.status;
    message = err.message;
  }
  
  res.status(status).json({
    error: message,
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack })
  });
}

function notFound(req, res, next) {
  res.status(404).json({ error: 'Route not found' });
}

module.exports = {
  handleError: handleError,
  notFound: notFound
};
```

Use it:

```javascript
// server.js
const express = require('express');
const { handleError, notFound } = require('./error-handler');

const app = express();

// Your routes here...

// Error handlers must be last
app.use(notFound);        // 404 handler
app.use(handleError);     // General error handler
```

## Testing Your Modules

You can test modules independently:

```javascript
// test/math-utils.test.js
const { add, multiply } = require('../math-utils');

// Simple tests
console.assert(add(2, 3) === 5, 'Add function failed');
console.assert(multiply(4, 5) === 20, 'Multiply function failed');

console.log('All tests passed!');
```

## Module Organization Best Practices

1. **Keep modules focused** - One module, one purpose
2. **Use descriptive names** - `auth-middleware.js` not `helpers.js`
3. **Export what's needed** - Don't export internal functions
4. **Document exports** - Add comments for complex modules
5. **Group related modules** - Use folders for organization:
   ```
   project/
   ├── middleware/
   │   ├── auth.js
   │   └── logger.js
   ├── routes/
   │   ├── users.js
   │   └── posts.js
   └── utils/
       ├── database.js
       └── helpers.js
   ```

## Summary

You've learned:
- How to create custom middleware modules
- Building class-based modules (like session stores)
- Using factory patterns for configurable modules
- Organizing routes into separate modules
- Creating reusable error handling modules

These patterns will help you write more maintainable and organized Node.js applications.

---

**[Previous: Module Basics](module-basics.md)** | **[Next: Creating Your Own Node.js Modules](index.md)**

