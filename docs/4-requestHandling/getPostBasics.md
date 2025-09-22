---
layout: default
title: GET and POST Basics
nav_order: 1
parent: Request Handling
---

# GET and POST Basics

[&laquo; Return to Request Handling Chapter](index.md)

## Introduction to HTTP Methods

HTTP (HyperText Transfer Protocol) defines several methods for communicating between clients and servers. The two most common methods are GET and POST, each serving different purposes in web applications.

## GET Requests

GET requests are used to **retrieve** data from the server. They're the most common type of request and are used when:
- Loading web pages
- Fetching data from APIs
- Searching for information
- Following links

### Characteristics of GET Requests

- **Data in URL**: Information is sent as part of the URL
- **Visible to users**: Users can see the data in their browser's address bar. This is not only good for visability but also allows the browser to set the request as a bookmark.
- **Cacheable**: Browsers and servers can cache GET requests.
- **Limited size**: URLs have length restrictions (typically 2048 characters). You can't have too much data in the GET request.
- **Should not modify data**: While there's technically nothing stopping us from having the server do things based on a GET request, we should not have the sever modify data based on a get request since anyone can send it.

### Example GET Request

When you visit a website like `https://example.com/search?q=javascript&category=programming`, you're making a GET request where:
- The URL path is `/search`
- The query parameters are `q=javascript` and `category=programming`

## POST Requests

POST requests are used to **send** data to the server. They're commonly used for:
- Submitting forms
- Creating new resources
- Uploading files
- User authentication
- Any operation that modifies server data

### Characteristics of POST Requests

- **Data in body**: Information is sent in the request body
- **Not visible in URL**: Data doesn't appear in the browser's address bar
- **Not bookmarkable**: You can't bookmark the data being sent
- **Not cacheable**: Browsers don't cache POST requests by default
- **No size limit**: Can send large amounts of data
- **Can still respond back**: Even though the primary purpose is for the client to send information to the server, the server can (and should) respond back.

### Example POST Request

When you submit a contact form, the form data (name, email, message) is sent in the request body, not in the URL.

## Basic Request Handling with Express

There is not too much difference between a GET route and a POST route in your code. One thing to note is, since they are different methods, you can have them share url paths as seen below:

```javascript
{% raw %}
const express = require('express');
const app = express();
const PORT = 3000;


// GET request example
app.get('/hello', (req, res) => {
    res.send('<h1>Hello from GET request!</h1>');
});

// POST request example
app.post('/hello', (req, res) => {
    res.send('<h1>Hello from POST request!</h1>');
});

app.listen(PORT, () => {
    console.log(`Server running on http://toastcode.net/tschotter_node`);
});
{% endraw %}
```

The primary difference is just using ```.post``` instead of ```.get``` to denote that the callback function will be run when that method (GET or POST) is attached to the proper url.

## Retrieving Request Data

### GET Request Data

GET request data comes through query parameters in the URL. Express automatically parses these into the `req.query` object:

```javascript
// URL: [home]/search?q=javascript&category=programming&page=2
app.get('/search', (req, res) => {
    const searchTerm = req.query.q;
    const category = req.query.category;
    const page = req.query.page;
    
    res.send(`
        <h1>Search Results</h1>
        <p>Searching for: ${searchTerm}</p>
        <p>Category: ${category}</p>
        <p>Page: ${page}</p>
    `);
});
{% endraw %}
```

**What happens:**
- `req.query.q` = "javascript"
- `req.query.category` = "programming"  
- `req.query.page` = "2"

### What happens with missing parameters?

If you try to access a query parameter that doesn't exist in the URL, it will be `undefined`:

```javascript
// URL: [home]/search?q=javascript (missing category and page)
app.get('/search', (req, res) => {
    const searchTerm = req.query.q;        // "javascript"
    const category = req.query.category;   // undefined
    const page = req.query.page;           // undefined
    
    res.send(`
        <h1>Search Results</h1>
        <p>Searching for: ${searchTerm}</p>
        <p>Category: ${category}</p>        <!-- Will show "undefined" -->
        <p>Page: ${page}</p>               <!-- Will show "undefined" -->
    `);
});
{% endraw %}
```

**Result in browser:**
```
Search Results
Searching for: javascript
Category: undefined
Page: undefined
```

### Handling missing parameters safely

You can provide default values to handle missing parameters:

```javascript
app.get('/search', (req, res) => {
    const searchTerm = req.query.q || 'all';
    const category = req.query.category || 'general';
    const page = req.query.page || '1';
    
    res.send(`
        <h1>Search Results</h1>
        <p>Searching for: ${searchTerm}</p>
        <p>Category: ${category}</p>
        <p>Page: ${page}</p>
    `);
});
{% endraw %}
```
In the above example, what is happening is if ```req.query.q``` is "undefined", it is effectively the same as saying it's "false" in a logical check. So by using the ```or``` operation (```||```), you can assign a default value if the GET request didn't include it.


**Now with URL: [home]/search?q=javascript**
**Result in browser:**
```
Search Results
Searching for: javascript
Category: general
Page: 1
```

### POST Request Data

POST request data comes through the request body. You need middleware to parse it:

```javascript
// Middleware to parse different content types
app.use(express.urlencoded({ extended: true })); // For form data
app.use(express.json()); // For JSON data

app.post('/submit', (req, res) => {
    const name = req.body.name;
    const email = req.body.email;
    
    res.send(`
        <h1>Form Submitted!</h1>
        <p>Name: ${name}</p>
        <p>Email: ${email}</p>
    `);
});
{% endraw %}
```

## More Complicated Examples

### Example 1: Simple Search API

```javascript
{% raw %}
const express = require('express');
const app = express();
const PORT = 3000;

// Mock data
const products = [
    { id: 1, name: 'Laptop', category: 'Electronics', price: 999 },
    { id: 2, name: 'Book', category: 'Education', price: 19 },
    { id: 3, name: 'Phone', category: 'Electronics', price: 699 },
    { id: 4, name: 'Notebook', category: 'Education', price: 5 }
];

// GET request to search products
app.get('/api/products', (req, res) => {
    const { category, minPrice, maxPrice } = req.query;
    
    let filteredProducts = products;
    
    // Filter by category if provided
    if (category) {
        filteredProducts = filteredProducts.filter(p => 
            p.category.toLowerCase() === category.toLowerCase()
        );
    }
    
    // Filter by price range if provided
    if (minPrice) {
        filteredProducts = filteredProducts.filter(p => p.price >= parseInt(minPrice));
    }
    
    if (maxPrice) {
        filteredProducts = filteredProducts.filter(p => p.price <= parseInt(maxPrice));
    }
    
    res.json({
        count: filteredProducts.length,
        products: filteredProducts
    });
});

app.listen(PORT, () => {
    console.log(`Server running on http://toastcode.net/tschotter_node`);
    console.log('Try: http://toastcode.net/tschotter_node/api/products?category=Electronics&minPrice=500');
});
{% endraw %}
```

### Example 2: User Registration

```javascript
{% raw %}
const express = require('express');
const app = express();
const PORT = 3000;

// Middleware
app.use(express.urlencoded({ extended: true }));
app.use(express.json());

// In-memory storage (use database in production)
let users = [];
let nextId = 1;

// POST request to create a new user
app.post('/api/users', (req, res) => {
    const { name, email, age } = req.body;
    
    // Basic validation
    if (!name || !email) {
        return res.status(400).json({
            error: 'Name and email are required'
        });
    }
    
    // Check if email already exists
    const existingUser = users.find(u => u.email === email);
    if (existingUser) {
        return res.status(409).json({
            error: 'Email already exists'
        });
    }
    
    // Create new user
    const newUser = {
        id: nextId++,
        name,
        email,
        age: age ? parseInt(age) : null,
        createdAt: new Date().toISOString()
    };
    
    users.push(newUser);
    
    res.status(201).json({
        message: 'User created successfully',
        user: newUser
    });
});

// GET request to retrieve all users
app.get('/api/users', (req, res) => {
    res.json({
        count: users.length,
        users: users
    });
});

// GET request to retrieve specific user
app.get('/api/users/:id', (req, res) => {
    const userId = parseInt(req.params.id);
    const user = users.find(u => u.id === userId);
    
    if (user) {
        res.json(user);
    } else {
        res.status(404).json({ error: 'User not found' });
    }
});

app.listen(PORT, () => {
    console.log(`Server running on http://toastcode.net/tschotter_node`);
});
{% endraw %}
```

## Redirecting to Static HTML Pages

Instead of sending HTML directly in your responses, you can redirect the client to static HTML pages if you're expecting them to send POST data through a form (and not a self-contained javascript request).

### Setting Up Static File Serving

First, create a `public` directory and add the middleware to serve static files:

```javascript
{% raw %}
const express = require('express');
const path = require('path');
const app = express();
const PORT = 3000;

// Middleware
app.use(express.urlencoded({ extended: true }));
app.use(express.json());
app.use(express.static('public')); // Serve static files from 'public' directory

// GET request that redirects to static page
app.get('/search', (req, res) => {
    const { q, category, page } = req.query;
    
    // Redirect to static page with query parameters
    res.redirect(`/search-results.html?q=${q || 'all'}&category=${category || 'general'}&page=${page || '1'}`);
});

// POST request that redirects after processing
app.post('/submit', (req, res) => {
    const { name, email } = req.body;
    
    // Process the data (save to database, etc.)
    console.log('Received:', { name, email });
    
    // Redirect to success page
    res.redirect('/success.html');
});
```


## Testing Your Routes

### Testing GET Requests

You can test GET requests by:
1. **Using your browser**: Simply visit the URL
2. **Using curl**: `curl http://toastcode.net/tschotter_node/api/products?category=Electronics`
3. **Using Postman**: A popular API testing tool

### Testing POST Requests

You can test POST requests by:

**Using curl:**
```bash
# JSON data
curl -X POST http://toastcode.net/tschotter_node/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"John Doe","email":"john@example.com","age":25}'

# Form data
curl -X POST http://toastcode.net/tschotter_node/api/users \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "name=Jane Smith&email=jane@example.com&age=30"
```

**Using a simple HTML form:**
```html
<!DOCTYPE html>
<html>
<head>
    <title>User Registration</title>
</head>
<body>
    <h1>Register New User</h1>
    <form action="http://toastcode.net/tschotter_node/api/users" method="POST">
        <div>
            <label for="name">Name:</label>
            <input type="text" id="name" name="name" required>
        </div>
        <div>
            <label for="email">Email:</label>
            <input type="email" id="email" name="email" required>
        </div>
        <div>
            <label for="age">Age:</label>
            <input type="number" id="age" name="age">
        </div>
        <button type="submit">Register</button>
    </form>
</body>
</html>
```

## Key Differences Summary

| Aspect | GET | POST |
|--------|-----|------|
| **Purpose** | Retrieve data | Send data |
| **Data location** | URL parameters | Request body |
| **Visibility** | Visible in URL | Hidden in body |
| **Bookmarkable** | Yes | No |
| **Cacheable** | Yes | No |
| **Size limit** | Limited (URL length) | No practical limit |
| **Security** | Less secure (data visible) | More secure (data hidden) |
| **Use cases** | Search, navigation, APIs | Forms, file uploads, login |

## Best Practices

1. **Use GET for retrieving data** - When you want to fetch information
2. **Use POST for sending data** - When you want to create or modify something
3. **Validate all input** - Always check the data you receive, even if you think the form doesn't allow it.
4. **Use appropriate HTTP status codes** - 200 for success, 201 for created, 400 for bad request
5. **Handle errors gracefully** - Provide meaningful error messages
6. **Consider security** - Never trust user input. **Never**

## Next Steps

Now that you understand the basics of GET and POST requests, you're ready to learn about templating to create dynamic HTML pages.

**[Next: Templating →](templating.md)**
