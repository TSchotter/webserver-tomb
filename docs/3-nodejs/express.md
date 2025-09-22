---
layout: default
title: Express.js
nav_order: 4
parent: NodeJS
---

# Express.js

[&laquo; Return to NodeJS Chapter](index.md)

## Introduction to Express.js

Express.js is a popular web framework for NodeJS that makes building web applications much easier. Instead of writing all the HTTP handling code manually (like we did in the previous chapters), Express provides a much more human-readable way to handle many aspects that would have been painful otherwise (such as the if/elseif chain we had earlier).

### Installing Express

First, let's install Express in your project (make sure you're at the same directory as your package.json):

```bash
npm install express
```

This will add Express to your `package.json` dependencies and create a `node_modules` folder if it didn't already exist from other modules.

## Basic Express Server

Here is an extremely basic Express server to make sure everything is functioning:

```javascript
// server.js
const express = require('express');
const app = express();
const PORT = 3030;

// Basic route
app.get('/', (req, res) => {
    res.send('Hello from Express!');
});

// Start the server
app.listen(PORT, () => {
    console.log(`Express server running on http://localhost:${PORT}`);
});
{% endraw %}
```

Much simpler than our previous NodeJS server! Let's break down what's happening:

- `const express = require('express')` - Import the Express module
- `const app = express()` - Create an Express application
- `app.get('/', ...)` - Define a route for GET requests to the root path
- `res.send()` - Send a response (automatically sets content-type)
- `app.listen()` - Start the server on the specified port

## Serving Static Files with Express

One of Express's most useful features is serving static files. This is much easier than writing all the file handling code manually.

### Setting Up Static File Serving

```javascript
// server.js
const express = require('express');
const path = require('path');
const app = express();
const PORT = 3000;

// Serve static files from the 'public' directory
app.use(express.static('public'));

// Basic route
app.get('/', (req, res) => {
    res.send('Hello from Express!');
});

app.listen(PORT, () => {
    console.log(`Express server running on http://localhost:${PORT}`);
    console.log('Serving static files from the "public" directory');
});
{% endraw %}
```

### File Structure for Static Files

Create this directory structure:

```
www/
└── your-project/
    ├── package.json
    ├── server.js
    └── public/
        ├── index.html
        ├── style.css
        ├── script.js
        └── images/
            └── logo.png
```


### How Static File Serving Works

When you use `app.use(express.static('public'))`, Express:

**Maps URLs to files**: `http://localhost:3030/style.css` → `public/style.css`

**Sets MIME types automatically**: CSS files get `text/css`, images get appropriate image types

**Handles 404s**: Returns 404 for files that don't exist

**Supports subdirectories**: `http://localhost:3030/images/logo.png` → `public/images/logo.png`

This cuts out some of the busywork that would otherwise need to be handled by you.


## Express Routes

Express makes it easy to create different routes for different parts of your application:

```javascript
{% raw %}
const express = require('express');
const app = express();
const PORT = 3000;

// Home page
app.get('/', (req, res) => {
    res.send('<h1>Welcome to My Express Site!</h1>');
});

// About page
app.get('/about', (req, res) => {
    res.send('<h1>About Us</h1><p>This is our about page.</p>');
});

// Contact page
app.get('/contact', (req, res) => {
    res.send('<h1>Contact Us</h1><p>Get in touch with us!</p>');
});

// API endpoint
app.get('/api/hello', (req, res) => {
    res.json({
        message: 'Hello from the API!',
        timestamp: new Date().toISOString()
    });
});

// 404 handler (must be last)
app.use((req, res) => {
    res.status(404).send('<h1>404 - Page Not Found</h1>');
});

app.listen(PORT, () => {
    console.log(`Express server running on http://localhost:${PORT}`);
});
{% endraw %}
```
> The above can be combined with the ```app.use(express.static('public'));``` so that you will redirect clients to static pages if you don't have a specific route.

> A form of this was done in the original nodejs server example, with each if/else chain leading to a different function. This is much more readable and you do not need the if/else chain.


## Express Middleware

Middleware are functions that run between the request and response. Express comes with built-in middleware and you can create your own:

```javascript
{% raw %}
const express = require('express');
const app = express();
const PORT = 3000;

// Built-in middleware for parsing JSON
app.use(express.json());

// Custom middleware for logging
app.use((req, res, next) => {
    console.log(`${new Date().toISOString()} - ${req.method} ${req.url}`);
    next(); // Important: call next() to continue to the next middleware
});

// Routes
app.get('/', (req, res) => {
    res.send('Hello from Express!');
});

app.post('/api/data', (req, res) => {
    console.log('Received data:', req.body);
    res.json({ message: 'Data received successfully' });
});

app.listen(PORT, () => {
    console.log(`Express server running on http://localhost:${PORT}`);
});
{% endraw %}
```
> The middleware above will intercept every request and ```console.log``` the date/time, method, and req url before moving on to the next applicible method.



## Next Steps

Now that you understand Express.js and how to serve static files, you have the foundation for building complete web applications. You can:

- Create multiple routes for different pages
- Serve static files like HTML, CSS, and JavaScript
- Use middleware for common tasks like logging and parsing
- Build APIs that return JSON data

This completes the NodeJS chapter! You now have the essential knowledge to build web servers and applications with NodeJS and Express.

**[Return to NodeJS Chapter →](index.md)**
