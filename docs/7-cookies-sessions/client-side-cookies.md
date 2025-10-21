---
layout: default
title: Client-Side Cookies with Forms
nav_order: 1
parent: Cookies and Sessions
---

# Client-Side Cookies with Forms

In this chapter, we'll learn how to set and read cookies using simple HTML forms. We'll start with a basic application and gradually add cookie functionality to track user data.

## What Are Cookies?

Cookies are small text files stored in the user's browser that contain data sent by your web server. They're commonly used for:
- Remembering user preferences
- Storing login information
- Tracking user behavior
- Maintaining state between page visits

## Starting Project: Basic Form Without Cookies

Let's start with a simple Express.js application that has forms but no cookie functionality yet.

### Step 1: Create the Project Structure

```bash
mkdir cookie-demo
cd cookie-demo
npm init -y
npm install express hbs
```

### Step 2: Create the Basic Server (No Cookies Yet)

Create `server.js`:

```javascript
const express = require('express');
const app = express();
const path = require('path');
const hbs = require('hbs');

const PORT = 3010;

// Set view engine and views directory
app.set('view engine', 'hbs');
app.set('views', path.join(__dirname, 'views'));

// Middleware to parse form submits
app.use(express.urlencoded({ extended: false }));

// Home page - shows static user data
app.get('/', (req, res) => {
    const user = {
        name: "Guest",
        msg: "Welcome! Please set your name."
    };
    
    res.render('home', { user: user });
});

// Page to change name
app.get('/change', (req, res) => {
    res.render('changePage');
});

// Handle form submission (no cookie functionality yet)
app.post('/setname', (req, res) => {
    const name = (req.body && req.body.name) ? req.body.name : '';
    console.log('User entered name:', name);
    
    // For now, just redirect back to home
    res.redirect('/');
});

app.listen(PORT, () => {
    console.log(`Server is running on port ${PORT}`);
});
```

### Step 3: Create the Views Directory

```bash
mkdir views
```

### Step 4: Create the Basic Home Template (No Cookie Functionality Yet)

Create `views/home.hbs`:

```html
{% raw %}
<!DOCTYPE html>
<head>
    <title>Basic Form Demo</title>
</head>
<body>
    <h1>Basic Form Demo</h1>
    
    <h2>Current User Information</h2>
    <p><strong>Name:</strong> {{user.name}}</p>
    <p><strong>Message:</strong> {{user.msg}}</p>
    
    <div>
        <a href="/change">Change Name</a>
    </div>
    
    
</body>
</html>
{% endraw %}
```

### Step 5: Create the Basic Change Name Template

Create `views/changePage.hbs`:

```html
{% raw %}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Change Name - Basic Demo</title>
</head>
<body>
    <h1>Change Your Name</h1>
    
    <form method="POST" action="/setname">
        <div>
            <label for="name">Enter your name:</label>
            <input type="text" id="name" name="name" placeholder="Your name here" required>
        </div>
        
        <div>
            <button type="submit">Save Name</button>
            <a href="/">Cancel</a>
        </div>
    </form>
    
    
</body>
</html>
{% endraw %}
```

## Testing the Basic Application

1. **Start the server:**
   ```bash
   node server.js
   ```

2. **Visit the application:**
   - Go to `http://localhost:3010`
   - You should see "Guest" as the name

3. **Try the form:**
   - Click "Change Name"
   - Enter your name and submit
   - You'll be redirected back to the home page
   - Notice that it still shows "Guest" - the name wasn't saved!

## Now Let's Add Cookie Functionality!

### Step 6: Install Cookie Parser

First, let's install the cookie-parser middleware:

```bash
npm install cookie-parser
```

### Step 7: Update the Server to Use Cookies

Now let's modify `server.js` to add cookie functionality:

```javascript
const express = require('express');
const app = express();
const path = require('path');
const hbs = require('hbs');
const cookieParser = require('cookie-parser'); // Add this line

const PORT = 3010;

// Set view engine and views directory
app.set('view engine', 'hbs');
app.set('views', path.join(__dirname, 'views'));

// Middleware to parse form submits
app.use(express.urlencoded({ extended: false }));
app.use(cookieParser()); // Add this line

// Home page - now reads cookie data
app.get('/', (req, res) => {
    let user = {
        name: "Guest",
        msg: "Welcome! Please set your name."
    };
    
    // Check if cookie exists and parse it
    if (req.cookies && req.cookies.name) {
        user = JSON.parse(req.cookies.name);
    }
    
    res.render('home', { user: user });
});

// Page to change name
app.get('/change', (req, res) => {
    res.render('changePage');
});

// Handle form submission - now sets a cookie
app.post('/setname', (req, res) => {
    const name = (req.body && req.body.name) ? req.body.name : '';
    
    // Set cookie with user data
    res.cookie('name', JSON.stringify({
        name: name, 
        msg: "Hello there!"
    }), { 
        httpOnly: true,
        maxAge: 24 * 60 * 60 * 1000, // 24 hours
        secure: false,
        sameSite: "lax"
    });
    
    res.redirect('/');
});

app.listen(PORT, () => {
    console.log(`Server is running on port ${PORT}`);
});
```

### Step 8: Update the Home Template

Update `views/home.hbs` to show the cookie functionality:

```html
{% raw %}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Cookie Demo</title>
</head>
<body>
    <h1>Cookie Demo Application</h1>
    
    <h2>Current User Information</h2>
    <p><strong>Name:</strong> {{user.name}}</p>
    <p><strong>Message:</strong> {{user.msg}}</p>
    
    <div>
        <a href="/change">Change Name</a>
        <form method="POST" action="/resetcookie" style="display: inline;">
            <button type="submit">Reset Cookie</button>
        </form>
    </div>
    
    <div>
        <h3>Success! Cookies are working:</h3>
        <ul>
            <li>When you first visit, you're shown as "Guest"</li>
            <li>Click "Change Name" to set a cookie with your name</li>
            <li>After setting the cookie, refresh the page - your name persists!</li>
            <li>Click "Reset Cookie" to clear the stored data</li>
        </ul>
    </div>
</body>
</html>
{% endraw %}
```

### Step 9: Add Cookie Reset Functionality

Add this route to your `server.js`:

```javascript
// Handle cookie reset
app.post('/resetcookie', (req, res) => {
    res.clearCookie('name');
    res.redirect('/');
});
```

## Testing the Updated Application

1. **Restart the server:**
   ```bash
   node server.js
   ```

2. **Visit the application:**
   - Go to `http://localhost:3010`
   - You should see "Guest" as the name

3. **Set a cookie:**
   - Click "Change Name"
   - Enter your name and submit
   - You'll be redirected back to the home page
   - Your name should now be displayed!

4. **Test persistence:**
   - Refresh the page - your name should still be there
   - Close and reopen your browser - your name should persist

5. **Reset the cookie:**
   - Click "Reset Cookie"
   - You should be back to "Guest"

## Understanding the Code

### Cookie Middleware

```javascript
app.use(cookieParser());
```

The `cookie-parser` middleware automatically parses cookies from incoming requests and makes them available in `req.cookies`.

### Setting Cookies

```javascript
res.cookie('name', JSON.stringify({
    name: name, 
    msg: "Hello there!"
}), { 
    httpOnly: true,
    maxAge: 24 * 60 * 60 * 1000, // 24 hours
    secure: false,
    sameSite: "lax"
});
```

**Cookie options explained:**
- `httpOnly: true` - Cookie can't be accessed by JavaScript (security)
- `maxAge: 24 * 60 * 60 * 1000` - Cookie expires in 24 hours
- `secure: false` - Cookie works on HTTP (set to `true` for HTTPS only)
- `sameSite: "lax"` - Cookie is sent with same-site requests

### Reading Cookies

```javascript
if (req.cookies && req.cookies.name) {
    user = JSON.parse(req.cookies.name);
}
```

We check if the cookie exists, then parse the JSON data stored in it.

### Clearing Cookies

```javascript
res.clearCookie('name');
```

This removes the cookie from the browser.

## Common Cookie Use Cases

### 1. User Preferences
```javascript
// Store user's theme preference
res.cookie('theme', 'dark', { maxAge: 30 * 24 * 60 * 60 * 1000 }); // 30 days
```

### 2. Shopping Cart
```javascript
// Store cart items
const cartItems = ['item1', 'item2', 'item3'];
res.cookie('cart', JSON.stringify(cartItems), { maxAge: 7 * 24 * 60 * 60 * 1000 });
```

### 3. Language Settings
```javascript
// Store user's language preference
res.cookie('language', 'en', { maxAge: 365 * 24 * 60 * 60 * 1000 }); // 1 year
```

## Security Considerations

### 1. Never Store Sensitive Data
```javascript
// ❌ DON'T do this
res.cookie('password', userPassword);

// ✅ DO this instead
res.cookie('user_id', userId);
```

### 2. Use httpOnly for Security
```javascript
// ✅ Prevents JavaScript access
res.cookie('session', sessionId, { httpOnly: true });
```

### 3. Set Appropriate Expiration
```javascript
// ✅ Short expiration for sensitive data
res.cookie('temp_token', token, { maxAge: 15 * 60 * 1000 }); // 15 minutes
```

## Troubleshooting

### Cookie Not Being Set
- Check if the response is being sent after setting the cookie
- Verify the cookie name doesn't contain special characters
- Ensure the browser allows cookies

### Cookie Not Being Read
- Check if `cookie-parser` middleware is properly installed
- Verify the cookie name matches exactly
- Check if the cookie has expired

### Cookie Persistence Issues
- Verify `maxAge` or `expires` is set correctly
- Check if the browser is in private/incognito mode
- Ensure the domain and path are correct

## Next Steps

Now that you understand basic cookie usage, you're ready to learn about:
- **[Server-Side Sessions](server-side-sessions.md)** - More secure session management
- **[Cookie Security](cookie-security.md)** - Advanced security options
- **[Authentication Patterns](authentication-patterns.md)** - Login systems with cookies

---

**[Previous: Cookies and Sessions Overview](index.md)** | **[Next: Server-Side Sessions](server-side-sessions.md)**
