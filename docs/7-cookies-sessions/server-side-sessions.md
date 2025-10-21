---
layout: default
title: Server-Side Sessions
nav_order: 2
parent: Cookies and Sessions
---

# Server-Side Sessions

In this chapter, we'll learn how to implement server-side sessions using Express.js and the `express-session` middleware. Unlike client-side cookies, server-side sessions store sensitive data on the server, making them more secure for authentication and user state management. This doesn't mean that *nothing* is stored on the client though. The client will hold onto a session ID that it uses as a way for the server to recognize which session the client is.

This gives some advantages:

- **Security**: Sensitive data stays on the server
- **Storage capacity**: No 4KB cookie size limit
- **Server control**: Sessions can be invalidated server-side
- **Data integrity**: Server controls what data is stored

## Project Structure

By the end of this chapter, you'll have this folder structure:

```
session-demo/
├── server.js
├── package.json
├── package-lock.json
└── views/
    ├── home.hbs
    ├── login.hbs
    └── profile.hbs
```

## Starting Project: Basic Application Without Sessions

Let's start with a simple Express.js application that has login functionality but no session management yet.

### Step 1: Create the Project Structure

```bash
mkdir session-demo
cd session-demo
npm init -y
npm install express hbs
```

### Step 2: Create the Basic Server (No Sessions Yet)

Create `server.js`:

```javascript
const express = require('express');
const app = express();
const path = require('path');
const hbs = require('hbs');

const PORT = 3011;

// Set view engine and views directory
app.set('view engine', 'hbs');
app.set('views', path.join(__dirname, 'views'));

// Middleware to parse form submits
app.use(express.urlencoded({ extended: false }));

// Home page - shows static user data
app.get('/', (req, res) => {
    const user = {
        name: "Guest",
        isLoggedIn: false,
        loginTime: null,
        visitCount: 0
    };
    
    res.render('home', { user: user });
});

// Login page
app.get('/login', (req, res) => {
    res.render('login');
});

// Handle login form submission (no session functionality yet)
app.post('/login', (req, res) => {
    const username = req.body.username;
    const password = req.body.password;
    
    console.log('User attempted login:', username);
    
    // For now, just redirect back to home
    res.redirect('/');
});

// Profile page (no session check yet)
app.get('/profile', (req, res) => {
    const user = {
        name: "Guest",
        loginTime: null,
        visitCount: 0
    };
    
    res.render('profile', { user: user });
});

app.listen(PORT, () => {
    console.log(`Server is running on port ${PORT}`);
});
```

### Step 3: Create the Views Directory

```bash
mkdir views
```

### Step 4: Create the Basic Home Template (No Session Functionality Yet)

Create `views/home.hbs`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Basic Login Demo</title>
</head>
<body>
    <h1>Basic Login Demo</h1>
    
    <h2>Current User Information</h2>
    <p><strong>Name:</strong> {{user.name}}</p>
    <p><strong>Status:</strong> {{#if user.isLoggedIn}}Logged In{{else}}Guest{{/if}}</p>
    <p><strong>Message:</strong> Please login to access your profile.</p>
    
    <p>
        <a href="/login">Login</a> | 
        <a href="/profile">View Profile</a>
    </p>
    
</body>
</html>
```

### Step 5: Create the Basic Login Template

Create `views/login.hbs`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Login - Basic Demo</title>
</head>
<body>
    <h1>Login</h1>
    
    <form method="POST" action="/login">
        <p>
            <label for="username">Username:</label><br>
            <input type="text" id="username" name="username" required>
        </p>
        
        <p>
            <label for="password">Password:</label><br>
            <input type="password" id="password" name="password" required>
        </p>
        
        <p>
            <button type="submit">Login</button>
            <a href="/">Cancel</a>
        </p>
    </form>
    
</body>
</html>
```

### Step 6: Create the Basic Profile Template

Create `views/profile.hbs`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Profile - Basic Demo</title>
</head>
<body>
    <h1>User Profile</h1>
    
    <h2>Welcome, {{user.name}}!</h2>
    <p><strong>Login Time:</strong> {{user.loginTime}}</p>
    <p><strong>Visit Count:</strong> {{user.visitCount}}</p>
    
    <p><a href="/">Home</a></p>
    
</body>
</html>
```

## Testing the Basic Application

**Start the server:**
   ```bash
   node server.js
   ```

**Visit the application:**
   - Go to `http://[yourserver]:3011`
   - You should see "Guest" as the name

**Try the login form:**
   - Click "Login"
   - Enter any username and password
   - You'll be redirected back to the home page
   - Notice that it still shows "Guest" - the login wasn't remembered!

**Try the profile page:**
   - Click "View Profile"
   - You'll see the profile page, but it shows guest information
   - There's no way to tell if you're actually logged in!

## Time to add some session functionality!

### Step 7: Install Express Session

First, let's install the express-session middleware:

```bash
npm install express-session
```

### Step 8: Update the Server to Use Sessions

Now let's modify `server.js` to add session functionality:

```javascript
const express = require('express');
const session = require('express-session'); // Add this line
const app = express();
const path = require('path');
const hbs = require('hbs');

const PORT = 3011;

// Set view engine and views directory
app.set('view engine', 'hbs');
app.set('views', path.join(__dirname, 'views'));

// Middleware to parse form submits
app.use(express.urlencoded({ extended: false }));

// Session middleware configuration - Add this block
app.use(session({
    secret: 'your-secret-key-change-this-in-production',
    resave: false,
    saveUninitialized: false,
    cookie: {
        secure: false, // Set to true if using HTTPS
        maxAge: 24 * 60 * 60 * 1000 // 24 hours
    }
}));
```

**What this session middleware does:**

The `express-session` middleware creates a session for each user that visits your application. Here's what each part does:

- **`secret`**: This is a secret key used to sign the session ID cookie. It should be a long, random string. This prevents users from tampering with their session ID.

- **`resave: false`**: Tells the session store not to save the session back to the store if it wasn't modified during the request. This prevents unnecessary database writes. (The "store" is where session data is kept - by default it's stored in memory, but eventually we'll be using a database)

- **`saveUninitialized: false`**: Prevents saving sessions that are new but not modified. This helps with privacy compliance and reduces storage usage. Without this, we'll be saving sessions for people who don't use the session at all (and we could have just given them the default values).

- **`cookie.secure: false`**: When `true`, the cookie is only sent over HTTPS connections. Right now we need false since this server can't run on https.

- **`cookie.maxAge`**: How long the session cookie lasts (24 hours in this case). After this time, the session expires and the user needs to log in again.

**What happens when the session expires:**
- The browser automatically deletes the session ID cookie after 24 hours. We can't trust this though since that's controlled by the client.
- The server also tracks when each session was created and automatically expires sessions after 24 hours, regardless of what the browser does. It does this by comparing the session age to the value it expects to expire by. If the server gets a request from an expired session, it deletes the session and treats the user as a new user.

Continuing with the modifications to the server.js...

```javascript
// Home page - now reads session data
app.get('/', (req, res) => {
    let user = {  // We keep the Guest object to act as a default if there is no session
        name: "Guest",
        isLoggedIn: false,
        loginTime: null,
        visitCount: 0
    };
    
    // Check if user is logged in via session
    if (req.session.isLoggedIn) {
        user = {
            name: req.session.username,
            isLoggedIn: true,
            loginTime: req.session.loginTime,
            visitCount: req.session.visitCount || 0
        };
        
        // Increment visit count
        req.session.visitCount = (req.session.visitCount || 0) + 1;
    }
    
    res.render('home', { user: user });
});

// Login page
app.get('/login', (req, res) => {
    res.render('login');
});

// Handle login form submission - now sets session data
app.post('/login', (req, res) => {
    const username = req.body.username;
    const password = req.body.password;
    
    // Simple authentication (in production, use proper password hashing)
    if (username && password) {
        // Set session data
        req.session.isLoggedIn = true;
        req.session.username = username;
        req.session.loginTime = new Date().toISOString();
        req.session.visitCount = 0;
        
        console.log(`User ${username} logged in at ${req.session.loginTime}`);
        res.redirect('/');
    } else {
        res.redirect('/login?error=1');
    }
});

// Logout route - Add this new route
app.post('/logout', (req, res) => {
    req.session.destroy((err) => {
        if (err) {
            console.log('Error destroying session:', err);
        }
        res.redirect('/');
    });
});

// Profile page - now requires login
app.get('/profile', (req, res) => {
    if (!req.session.isLoggedIn) {
        return res.redirect('/login');
    }
    
    const user = {
        name: req.session.username,
        loginTime: req.session.loginTime,
        visitCount: req.session.visitCount || 0
    };
    
    res.render('profile', { user: user });
});

app.listen(PORT, () => {
    console.log(`Server is running on port ${PORT}`);
});
```

**How it works:**
1. When a user first visits, the middleware creates a unique session ID
2. This session ID is stored in a cookie on the user's browser
3. The actual session data (like `req.session.username`) is stored on the server
4. On each request, the middleware uses the session ID to find the user's session data

### Step 9: Update the Home Template

Update `views/home.hbs` to show the session functionality:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Session Demo</title>
</head>
<body>
    <h1>Session Demo Application</h1>
    
    <h2>Current User Information</h2>
    <p><strong>Name:</strong> {{user.name}}</p>
    <p><strong>Status:</strong> {{#if user.isLoggedIn}}Logged In{{else}}Guest{{/if}}</p>
    {{#if user.isLoggedIn}}
    <p><strong>Login Time:</strong> {{user.loginTime}}</p>
    <p><strong>Visit Count:</strong> {{user.visitCount}}</p>
    {{/if}}
    
    <p>
        {{#if user.isLoggedIn}}
            <a href="/profile">View Profile</a> | 
            <form method="POST" action="/logout" style="display: inline;">
                <button type="submit">Logout</button>
            </form>
        {{else}}
            <a href="/login">Login</a>
        {{/if}}
    </p>
    
</body>
</html>
```

### Step 10: Update the Login Template

Update `views/login.hbs` to handle errors:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Login - Session Demo</title>
</head>
<body>
    <h1>Login</h1>
    
    <form method="POST" action="/login">
        <p>
            <label for="username">Username:</label><br>
            <input type="text" id="username" name="username" required>
        </p>
        
        <p>
            <label for="password">Password:</label><br>
            <input type="password" id="password" name="password" required>
        </p>
        
        <p>
            <button type="submit">Login</button>
            <a href="/">Cancel</a>
        </p>
    </form>
    
    {{#if error}}
    <p style="color: red;">Please enter both username and password.</p>
    {{/if}}
    
</body>
</html>
```

### Step 11: Update the Profile Template

Update `views/profile.hbs` to include logout functionality:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Profile - Session Demo</title>
</head>
<body>
    <h1>User Profile</h1>
    
    <h2>Welcome, {{user.name}}!</h2>
    <p><strong>Login Time:</strong> {{user.loginTime}}</p>
    <p><strong>Visit Count:</strong> {{user.visitCount}}</p>
    
    <p>
        <a href="/">Home</a> | 
        <form method="POST" action="/logout" style="display: inline;">
            <button type="submit">Logout</button>
        </form>
    </p>
    
</body>
</html>
```

## Testing the Updated Session Application

1. **Restart the server:**
   ```bash
   node server.js
   ```

2. **Visit the application:**
   - Go to `http://[yourserver]:3011`
   - You should see "Guest" as the name

3. **Test login:**
   - Click "Login"
   - Enter any username and password
   - You should be redirected to home with your session active
   - Notice the page now shows your username and login status!

4. **Test session persistence:**
   - Refresh the page - you should stay logged in
   - Visit the profile page - your session data should be there
   - Close and reopen your browser - you should still be logged in

5. **Test logout:**
   - Click "Logout"
   - You should be redirected to home as a guest

6. **Test protected routes:**
   - Try to visit `/profile` without logging in
   - You should be redirected to the login page

## Understanding the Code

Some of this was explained above, but here is the footnotes version.


### Session Middleware Configuration

```javascript
app.use(session({
    secret: 'your-secret-key-change-this-in-production',
    resave: false,
    saveUninitialized: false,
    cookie: {
        secure: false, // Set to true if using HTTPS
        maxAge: 24 * 60 * 60 * 1000 // 24 hours
    }
}));
```

**Configuration options explained:**
- `secret`: Used to sign the session ID cookie (change this in production!)
- `resave`: Forces session to be saved back to session store
- `saveUninitialized`: Forces uninitialized sessions to be saved
- `cookie.secure`: Set to true for HTTPS only
- `cookie.maxAge`: Session expiration time

### Setting Session Data

```javascript
req.session.isLoggedIn = true;
req.session.username = username;
req.session.loginTime = new Date().toISOString();
```

Session data is stored in `req.session` and automatically persisted. You don't need to go and find the session yourself, the session middleware does this for you.

### Reading Session Data

```javascript
if (req.session.isLoggedIn) {
    user = {
        name: req.session.username,
        isLoggedIn: true,
        loginTime: req.session.loginTime
    };
}
```

Notice that we didn't need to check if "req.ression" exists? Every user has a session, even if they don't modify it.

Access session data through `req.session` object.

### Destroying Sessions

```javascript
req.session.destroy((err) => {
    if (err) {
        console.log('Error destroying session:', err);
    }
    res.redirect('/');
});
```

Use `req.session.destroy()` to completely remove the session.

## Advanced Session Features


### Session Regeneration

```javascript
// Regenerate session ID after login (security best practice)
app.post('/login', (req, res) => {
    // ... authentication logic ...
    
    req.session.regenerate((err) => {
        if (err) throw err;
        
        req.session.isLoggedIn = true;
        req.session.username = username;
        res.redirect('/');
    });
});
```

Here we're creating a fresh session for the user once they log in. This is useful for security purposes in case the previous session id (before they log in) was compromised. It also ensures that there isn't any session information bleed-over from a previous login. Just because the server sees this client as a single client, doesn't mean that the client is only being operated by one person.


## Common Session Use Cases

### 1. User Authentication
```javascript
// Check if user is authenticated
app.get('/protected', (req, res) => {
    if (!req.session.isLoggedIn) {
        return res.status(401).send('Unauthorized');
    }
    res.send('Protected content');
});
```

If the user isn't logged in, they can't access the /protected site.

### 2. Shopping Cart
```javascript
// Add item to cart
app.post('/add-to-cart', (req, res) => {
    if (!req.session.cart) {
        req.session.cart = [];
    }
    req.session.cart.push(req.body.item);
    res.json({ success: true });
});
```

No matter where you go on the site, it should store what you have in the cart.


### 3. Flash Messages
```javascript
// Set flash message
req.session.flash = {
    type: 'success',
    message: 'Profile updated successfully!'
};

// Display and clear flash message
app.get('/', (req, res) => {
    const flash = req.session.flash;
    delete req.session.flash; // Clear after reading
    res.render('home', { flash: flash });
});
```

User do something somewhere and you want to display a message (and have all pages be capable of messages)? Have the session remember that something happened. Then tell the page that this user should have a notification (then delete the flash message).

---

**[Previous: Client-Side Cookies](client-side-cookies.md)** | **[Next: Cookies and Sessions Overview](index.md)**
