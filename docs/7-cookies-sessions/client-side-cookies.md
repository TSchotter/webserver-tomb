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

<pre><code>&lt;!DOCTYPE html&gt;
&lt;head&gt;
    &lt;title&gt;Basic Form Demo&lt;/title&gt;
&lt;/head&gt;
&lt;body&gt;
    &lt;h1&gt;Basic Form Demo&lt;/h1&gt;
    
    &lt;h2&gt;Current User Information&lt;/h2&gt;
    &lt;p&gt;&lt;strong&gt;Name:&lt;/strong&gt; {{user.name}}&lt;/p&gt;
    &lt;p&gt;&lt;strong&gt;Message:&lt;/strong&gt; {{user.msg}}&lt;/p&gt;
    
    &lt;div&gt;
        &lt;a href="/change"&gt;Change Name&lt;/a&gt;
    &lt;/div&gt;
    
    
&lt;/body&gt;
&lt;/html&gt;</code></pre>

### Step 5: Create the Basic Change Name Template

Create `views/changePage.hbs`:

<pre><code>&lt;!DOCTYPE html&gt;
&lt;html lang="en"&gt;
&lt;head&gt;
    &lt;meta charset="UTF-8"&gt;
    &lt;meta name="viewport" content="width=device-width, initial-scale=1.0"&gt;
    &lt;title&gt;Change Name - Basic Demo&lt;/title&gt;
&lt;/head&gt;
&lt;body&gt;
    &lt;h1&gt;Change Your Name&lt;/h1&gt;
    
    &lt;form method="POST" action="/setname"&gt;
        &lt;div&gt;
            &lt;label for="name"&gt;Enter your name:&lt;/label&gt;
            &lt;input type="text" id="name" name="name" placeholder="Your name here" required&gt;
        &lt;/div&gt;
        
        &lt;div&gt;
            &lt;button type="submit"&gt;Save Name&lt;/button&gt;
            &lt;a href="/"&gt;Cancel&lt;/a&gt;
        &lt;/div&gt;
    &lt;/form&gt;
    
    
&lt;/body&gt;
&lt;/html&gt;</code></pre>

## Testing the Basic Application

1. **Start the server:**
   ```bash
   node server.js
   ```

2. **Visit the application:**
   - Go to `http://[yourserver]:3010`
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

<pre><code>&lt;!DOCTYPE html&gt;
&lt;head&gt;
    &lt;title&gt;Cookie Demo&lt;/title&gt;
&lt;/head&gt;
&lt;body&gt;
    &lt;h1&gt;Cookie Demo Application&lt;/h1&gt;
    
    &lt;h2&gt;Current User Information&lt;/h2&gt;
    &lt;p&gt;&lt;strong&gt;Name:&lt;/strong&gt; {{user.name}}&lt;/p&gt;
    &lt;p&gt;&lt;strong&gt;Message:&lt;/strong&gt; {{user.msg}}&lt;/p&gt;
    
    &lt;div&gt;
        &lt;a href="/change"&gt;Change Name&lt;/a&gt;
        &lt;form method="POST" action="/resetcookie" style="display: inline;"&gt;
            &lt;button type="submit"&gt;Reset Cookie&lt;/button&gt;
        &lt;/form&gt;
    &lt;/div&gt;
    
&lt;/body&gt;
&lt;/html&gt;</code></pre>

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
   - Go to `http://[your server]:3010`
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
---

**[Previous: Cookies and Sessions Overview](index.md)** | **[Next: Server-Side Sessions](server-side-sessions.md)**
