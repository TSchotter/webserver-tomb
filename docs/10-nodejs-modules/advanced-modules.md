---
layout: default
title: Advanced Module Patterns
nav_order: 3
parent: Creating Your Own Node.js Modules
---

# Advanced Module Patterns

Now that you understand the basics of modules and middleware, let's explore more advanced patterns including class-based modules, module factories, and creating specialized modules like custom session stores.

## Creating a Custom Session Store Module

Let's create a complete example by building a custom SQLite session store as a module.

**Why extend Store?** The `express-session` package provides a base `Store` class that defines the interface for session stores. By extending this class, we ensure our custom store is compatible with `express-session` and implements all the required methods (`get`, `set`, `destroy`, etc.). You can find the Store class documentation in the [express-session npm package documentation](https://www.npmjs.com/package/express-session#store). You'll notice at the top of this documentation, the authors specifically mention that the default memory storage method is not designed for production. So we're going to use the database instead.

When you create a custom session store, you need to:
1. Import the `Store` class from `express-session`
2. Extend it with your custom class
3. Implement the required methods that `express-session` expects

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

By doing things this way, rather than creating a whole new session management system for SQLite3, we can use express-session in a near identical fasion. But on the backend, it will interact with the database instead of internal memory.

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
3. **Export what's needed** - Don't export internal functions. If a function is only used by other functions in the file (and not outside), don't export it.
4. **Document exports** - Add comments for complex modules. Not just for others, but also for future you.
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
   ```


These patterns will help you write more maintainable and organized Node.js applications.

---

**[Previous: Middleware Modules](middleware-modules.md)** | **[Next: Creating Your Own Node.js Modules](index.md)**

