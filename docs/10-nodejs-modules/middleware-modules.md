---
layout: default
title: Middleware Modules
nav_order: 2
parent: Creating Your Own Node.js Modules
---

# Middleware Modules

Now that you understand the basics of modules, let's explore how to create reusable middleware modules for Express. Middleware functions are a powerful way to add functionality that runs before or after your route handlers. Understanding the difference between pre-route and post-route middleware is crucial for building effective Express applications.

## Understanding Middleware Execution Order

In Express, middleware functions execute in the order they are defined. The key distinction is:

- **Pre-route middleware**: Runs BEFORE the route handler. Used for authentication, validation, data parsing, etc.
- **Post-route middleware**: Runs AFTER the route handler. Used for logging, response modification, cleanup, etc.

The `next()` function is what controls this flow. When you call `next()`, Express moves to the next middleware or route handler in the chain.

## Pre-Route Middleware: Authentication and Authorization

Pre-route middleware runs before your route handler executes. This is perfect for checking if a user is authenticated, validating input, or performing security checks.

### Creating Authentication Middleware

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

**How it works:**
- `requireAuth` checks if the user has a session with a `userId`
- If authenticated, it calls `next()` to continue to the route handler
- If not authenticated, it redirects to the login page (stops the chain)

### Using Pre-Route Middleware

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
  // This code only runs if requireAuth calls next()
  res.render('profile', { 
    user: req.session.userId,
    username: req.session.username 
  });
});

// Admin route - requires admin role
app.get('/admin', requireAdmin, (req, res) => {
  // This code only runs if requireAdmin calls next()
  res.render('admin', { 
    message: 'Admin panel',
    user: req.session.userId 
  });
});

// Login page (no middleware needed)
app.get('/login', (req, res) => {
  res.render('login');
});

// Access denied page
app.get('/access-denied', (req, res) => {
  res.render('access-denied');
});
```

**Execution flow for `/profile`:**
1. Request comes in
2. `requireAuth` middleware runs first
3. If authenticated → calls `next()` → route handler runs
4. If not authenticated → redirects (stops, route handler never runs)

### Input Validation Middleware

Here's another example of pre-route middleware for validating user input:

```javascript
// validation-middleware.js

function validateEmail(req, res, next) {
  const email = req.body.email;
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  
  if (!email || !emailRegex.test(email)) {
    return res.status(400).json({ error: 'Invalid email address' });
  }
  
  next();
}

function validateRequiredFields(fields) {
  return (req, res, next) => {
    for (const field of fields) {
      if (!req.body[field]) {
        return res.status(400).json({ 
          error: `Missing required field: ${field}` 
        });
      }
    }
    next();
  };
}

module.exports = {
  validateEmail: validateEmail,
  validateRequiredFields: validateRequiredFields
};
```

Use it:

```javascript
// server.js
const { validateEmail, validateRequiredFields } = require('./validation-middleware');

// Validate email before processing
app.post('/subscribe', validateEmail, (req, res) => {
  // Email is valid, process subscription
  res.json({ message: 'Subscribed successfully' });
});

// Validate multiple required fields
app.post('/register', 
  validateRequiredFields(['name', 'email', 'password']), 
  (req, res) => {
    // All required fields are present
    res.json({ message: 'Registration successful' });
  }
);
```

## Post-Route Middleware: Logging and Response Processing

Post-route middleware runs AFTER your route handler executes. This is useful for logging requests, modifying responses, tracking metrics, or cleaning up resources.

### Creating Logging Middleware

Post-route middleware can capture information about the request and response after the route handler has completed:

```javascript
// logging-middleware.js

function requestLogger(req, res, next) {
  // Store the start time
  const startTime = Date.now();
  
  // Override res.end to capture response data
  const originalEnd = res.end;
  
  res.end = function(chunk, encoding) {
    // Calculate response time
    const duration = Date.now() - startTime;
    
    // Log the request details
    console.log(`${new Date().toISOString()} - ${req.method} ${req.path} - ${res.statusCode} - ${duration}ms`);
    
    // Call the original end method
    originalEnd.call(res, chunk, encoding);
  };
  
  next();
}

function detailedLogger(req, res, next) {
  const startTime = Date.now();
  const originalEnd = res.end;
  
  res.end = function(chunk, encoding) {
    const duration = Date.now() - startTime;
    
    const logData = {
      timestamp: new Date().toISOString(),
      method: req.method,
      path: req.path,
      statusCode: res.statusCode,
      duration: `${duration}ms`,
      ip: req.ip,
      userAgent: req.get('user-agent')
    };
    
    console.log(JSON.stringify(logData));
    
    originalEnd.call(res, chunk, encoding);
  };
  
  next();
}

module.exports = {
  requestLogger: requestLogger,
  detailedLogger: detailedLogger
};
```

### Using Post-Route Middleware

```javascript
// server.js
const express = require('express');
const { requestLogger, detailedLogger } = require('./logging-middleware');

const app = express();

// Apply logging middleware globally (runs for all routes)
app.use(requestLogger);

// Your routes
app.get('/', (req, res) => {
  res.send('Home page');
});

app.get('/api/users', (req, res) => {
  res.json({ users: [] });
});

// The logging middleware will run AFTER each route handler
// and log the request details
```

**Execution flow:**
1. Request comes in
2. `requestLogger` middleware runs (sets up response tracking)
3. Calls `next()` → route handler runs
4. Route handler sends response
5. `requestLogger`'s overridden `res.end` runs → logs the request

### Response Modification Middleware

You can also use post-route middleware to modify responses:

```javascript
// response-middleware.js

function addTimestamp(req, res, next) {
  const originalJson = res.json;
  
  res.json = function(data) {
    // Add timestamp to all JSON responses
    const dataWithTimestamp = {
      ...data,
      timestamp: new Date().toISOString()
    };
    
    return originalJson.call(this, dataWithTimestamp);
  };
  
  next();
}

function addRequestId(req, res, next) {
  // Generate a unique request ID
  req.requestId = `req-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  
  const originalJson = res.json;
  res.json = function(data) {
    const dataWithId = {
      ...data,
      requestId: req.requestId
    };
    
    return originalJson.call(this, dataWithId);
  };
  
  next();
}

module.exports = {
  addTimestamp: addTimestamp,
  addRequestId: addRequestId
};
```

Use it:

```javascript
// server.js
const { addTimestamp, addRequestId } = require('./response-middleware');

app.use(addRequestId); // Add request ID to all responses

app.get('/api/data', addTimestamp, (req, res) => {
  // This response will have both requestId and timestamp
  res.json({ message: 'Hello' });
  // Response will be: { message: 'Hello', requestId: 'req-...', timestamp: '...' }
});
```

## Combining Pre-Route and Post-Route Middleware

You can combine both types of middleware in a single route:

```javascript
// server.js
const { requireAuth } = require('./auth-middleware');
const { requestLogger } = require('./logging-middleware');

// Apply logging globally (runs after all routes)
app.use(requestLogger);

// Protected route with authentication
app.get('/dashboard', requireAuth, (req, res) => {
  res.render('dashboard', { user: req.session.userId });
});

// Execution order:
// 1. requestLogger sets up tracking
// 2. requireAuth checks authentication
// 3. Route handler renders dashboard
// 4. requestLogger logs the request
```

## Error Handling Middleware

Error handling middleware is a special type of post-route middleware that runs when an error occurs:

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
app.get('/api/users', (req, res) => {
  // If this throws an error, handleError will catch it
  const users = getUsersFromDatabase();
  res.json(users);
});

// Error handlers must be last
app.use(notFound);        // 404 handler (runs if no route matches)
app.use(handleError);     // General error handler (runs if error occurs)
```

**Important:** Error handling middleware must be defined after all routes and other middleware. It has a special signature: `(err, req, res, next)` - notice the `err` parameter first.

## Middleware Best Practices

1. **Use pre-route middleware for**: Authentication, authorization, validation, data parsing
2. **Use post-route middleware for**: Logging, response modification, metrics, cleanup
3. **Always call `next()`** unless you're intentionally stopping the request (like redirecting)
4. **Error handlers go last** - They must be defined after all routes
5. **Keep middleware focused** - One middleware, one responsibility
6. **Export reusable middleware** - Create modules for middleware you'll use in multiple places

## Summary

You've learned:
- **Pre-route middleware**: Runs before route handlers (authentication, validation)
- **Post-route middleware**: Runs after route handlers (logging, response modification)
- How to create authentication and validation middleware modules
- How to create logging and response modification middleware
- How to handle errors with middleware
- Best practices for organizing middleware modules

Understanding the execution order of middleware is crucial for building robust Express applications. Pre-route middleware protects and prepares your routes, while post-route middleware helps you monitor and enhance responses.

---

**[Previous: Module Basics](module-basics.md)** | **[Next: Advanced Module Patterns](advanced-modules.md)**

