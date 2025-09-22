---
layout: default
title: Templating
nav_order: 2
parent: Request Handling
---

# Templating

[&laquo; Return to Request Handling Chapter](index.md)

## Introduction to Templating

Templating is a powerful technique for generating dynamic HTML content on the server side. Instead of manually concatenating HTML strings or using complex template literals, templating engines provide a clean, organized way to create reusable HTML templates with dynamic data.

In this chapter, we'll explore:
1. **Basic templating** - Simple string replacement and template functions
2. **Handlebars templating** - A popular, feature-rich templating engine
3. **Best practices** - How to organize and structure your templates

## Basic Templating Without Handlebars

Before diving into templating engines, let's understand the fundamentals using basic JavaScript techniques.

### Method 1: Simple String Replacement

The most basic form of templating is string replacement:

```javascript
{% raw %}
const express = require('express');
const fs = require('fs');
const path = require('path');
const app = express();
const PORT = 3000;

// Middleware
app.use(express.urlencoded({ extended: true }));
app.use(express.json());
app.use(express.static('public'));

// Simple template function
function renderTemplate(template, data) {
    let html = template;
    
    // Replace placeholders with data
    for (const key in data) {
        const placeholder = '{{' + key + '}}';
        const value = data[key] || '';
        html = html.replace(new RegExp(placeholder, 'g'), value);
    }
    
    return html;
}

// User profile route
app.get('/user/:id', (req, res) => {
    const userId = parseInt(req.params.id);
    
    // Mock user data
    const users = [
        { id: 1, name: 'John Doe', email: 'john@example.com', age: 25, city: 'New York' },
        { id: 2, name: 'Jane Smith', email: 'jane@example.com', age: 30, city: 'Los Angeles' },
        { id: 3, name: 'Bob Johnson', email: 'bob@example.com', age: 35, city: 'Chicago' }
    ];
    
    const user = users.find(u => u.id === userId);
    
    if (!user) {
        return res.status(404).send('<h1>User not found</h1>');
    }
    
    // Read template file
    // path.join() creates a proper file path by combining directory parts
    // __dirname is the current directory where the script is running
    // This creates: /your-project/templates/user-profile.html
    const templatePath = path.join(__dirname, 'templates', 'user-profile.html');
    const template = fs.readFileSync(templatePath, 'utf8');
    
    // Render template with user data
    const html = renderTemplate(template, user);
    
    res.send(html);
});

app.listen(PORT, () => {
    console.log('Server running on http://toastcode.net/tschotter_node');
});
{% endraw %}
```

**Template file (`templates/user-profile.html`):**
```html
{% raw %}
<!DOCTYPE html>
<html>
<head>
    <title>{{name}} - Profile</title>
</head>
<body>
    <h1>{{name}}</h1>
    <p>User Profile</p>
    
    <div>
        <p><strong>Email:</strong> {{email}}</p>
        <p><strong>Age:</strong> {{age}}</p>
        <p><strong>City:</strong> {{city}}</p>
        <p><strong>User ID:</strong> #{{id}}</p>
    </div>
    
    <p><a href="/tschotter_node">← Back to Home</a></p>
</body>
</html>
{% endraw %}
```
> As you can see, the templating uses regular expressions to find and replace values. A helper function is used in the example to partially automate the process based on the json object sent to it.


## Introduction to Handlebars

Like how Express simplifies many processes related to webserver hosting, Handlebars simplifies many processes related to templating.

### Installing Handlebars

First, install the hbs package:

```bash
npm install hbs
```

### Setting Up Handlebars with Express

```javascript
{% raw %}
const express = require('express');
const hbs = require('hbs');
const path = require('path');
const app = express();
const PORT = 3000;

// Set up Handlebars
app.set('view engine', 'hbs');
app.set('views', path.join(__dirname, 'views'));

// Register partials directory
hbs.registerPartials(path.join(__dirname, 'views', 'partials'));

// Middleware
app.use(express.urlencoded({ extended: true }));
app.use(express.json());
app.use(express.static('public'));

// Routes
app.get('/', (req, res) => {
    res.render('home', {
        title: 'Welcome to Our Site',
        message: 'This is a Handlebars template!'
    });
});

app.listen(PORT, () => {
    console.log('Server running on http://toastcode.net/tschotter_node');
});
{% endraw %}
```

### Directory Structure

Create this directory structure for your Handlebars templates:

```
your-project/
├── views/
│   ├── partials/
│   │   ├── header.hbs
│   │   └── footer.hbs
│   ├── home.hbs
│   ├── user-profile.hbs
│   └── blog.hbs
├── public/
└── server.js
```

### Base Template (`views/layout.hbs`)

With `hbs`, we'll create a base template that other templates can extend:

```html
{% raw %}
<!DOCTYPE html>
<html>
<head>
    <title>{{title}} - My Website</title>
</head>
<body>
    {{> header}}
    
    <main>
        {{{body}}}
    </main>
    
    {{> footer}}
</body>
</html>
{% endraw %}
```

### Header Partial (`views/partials/header.hbs`)

```html
<header>
    <nav>
        <a href="/tschotter_node">Home</a> |
        <a href="/tschotter_node/blog">Blog</a> |
        <a href="/tschotter_node/users">Users</a>
    </nav>
</header>
```

### Footer Partial (`views/partials/footer.hbs`)

```html
<footer>
    <p>&copy; 2024 My Website. All rights reserved.</p>
</footer>
```

### Home Page Template (`views/home.hbs`)

```html
{% raw %}
{{> header}}

<h1>{{title}}</h1>
<p>{{message}}</p>
<p><a href="/tschotter_node/blog">View Blog</a></p>

{{> footer}}
{% endraw %}
```

### User Profile Template (`views/user-profile.hbs`)

```html
{% raw %}
{{> header}}

<h1>{{user.name}}</h1>
<p>User #{{user.id}}</p>

<div>
    <p><strong>Email:</strong> {{user.email}}</p>
    <p><strong>Age:</strong> {{user.age}} years old</p>
    <p><strong>Location:</strong> {{user.city}}</p>
    <p><strong>Member since:</strong> {{formatDate user.createdAt}}</p>
</div>

<p><a href="/tschotter_node/users">← Back to Users</a></p>

{{> footer}}
{% endraw %}
```

### Blog Template with Loops (`views/blog.hbs`)

```html
{% raw %}
{{> header}}

<h1>Latest Blog Posts</h1>
<p>Stay updated with our latest articles and tutorials</p>

{{#if posts}}
    {{#each posts}}
    <article>
        <h2>{{title}}</h2>
        <p><strong>By {{author}}</strong> on {{formatDate date}}</p>
        <p>{{excerpt}}</p>
        <p><a href="/tschotter_node/post/{{id}}">Read More →</a></p>
        <hr>
    </article>
    {{/each}}
{{else}}
    <p>No blog posts available at the moment.</p>
{{/if}}

{{> footer}}
{% endraw %}
```

### Server Routes with Handlebars

```javascript
{% raw %}
// User profile route
app.get('/user/:id', (req, res) => {
    const userId = parseInt(req.params.id);
    
    const users = [
        { 
            id: 1, 
            name: 'John Doe', 
            email: 'john@example.com', 
            age: 25, 
            city: 'New York',
            createdAt: '2024-01-15'
        },
        { 
            id: 2, 
            name: 'Jane Smith', 
            email: 'jane@example.com', 
            age: 30, 
            city: 'Los Angeles',
            createdAt: '2024-02-20'
        }
    ];
    
    const user = users.find(u => u.id === userId);
    
    if (!user) {
        return res.status(404).render('error', {
            title: 'User Not Found',
            message: 'The requested user could not be found.'
        });
    }
    
    res.render('user-profile', {
        title: user.name + ' - Profile',
        user: user
    });
});

// Blog route
app.get('/blog', (req, res) => {
    const posts = [
        { 
            id: 1, 
            title: 'Getting Started with Node.js', 
            author: 'John Doe', 
            date: '2024-01-15', 
            excerpt: 'Learn the basics of Node.js development and how to build your first server application.' 
        },
        { 
            id: 2, 
            title: 'Express.js Best Practices', 
            author: 'Jane Smith', 
            date: '2024-01-20', 
            excerpt: 'Discover the best practices for building robust and scalable Express applications.' 
        },
        { 
            id: 3, 
            title: 'Database Design Patterns', 
            author: 'Bob Johnson', 
            date: '2024-01-25', 
            excerpt: 'Explore common database design patterns and when to use each one in your applications.' 
        }
    ];
    
    res.render('blog', {
        title: 'Blog',
        posts: posts
    });
});
{% endraw %}
```

### Handlebars Helpers

Handlebars allows you to create custom helpers for more complex logic:

```javascript
{% raw %}
// Register custom helpers
hbs.registerHelper('formatDate', function(date) {
    return new Date(date).toLocaleDateString('en-US', {
        year: 'numeric',
        month: 'long',
        day: 'numeric'
    });
});

hbs.registerHelper('capitalize', function(str) {
    return str.charAt(0).toUpperCase() + str.slice(1);
});

hbs.registerHelper('ifEquals', function(arg1, arg2, options) {
    return (arg1 == arg2) ? options.fn(this) : options.inverse(this);
});

hbs.registerHelper('times', function(n, block) {
    let accum = '';
    for (let i = 0; i < n; ++i) {
        accum += block.fn(i);
    }
    return accum;
});
{% endraw %}
```

### Using Helpers in Templates

Here's how these helpers are used in your Handlebars templates:

**Using the `formatDate` helper:**
```html
{% raw %}
<!-- In your template -->
<p>Published on: {{formatDate post.date}}</p>
<!-- Output: Published on: January 15, 2024 -->
{% endraw %}
```

**Using the `capitalize` helper:**
```html
{% raw %}
<!-- In your template -->
<h1>Welcome, {{capitalize user.name}}!</h1>
<!-- If user.name is "john doe", output: Welcome, John doe! -->
{% endraw %}
```

**Using the `ifEquals` helper:**
```html
{% raw %}
<!-- In your template -->
{{#ifEquals user.role "admin"}}
    <p>You have admin privileges</p>
{{else}}
    <p>You are a regular user</p>
{{/ifEquals}}
{% endraw %}
```

**Using the `times` helper:**
```html
{% raw %}
<!-- In your template -->
{{#times 5}}
    <p>Star {{this}}</p>
{{/times}}
<!-- Output: 5 paragraphs with "Star 0", "Star 1", etc. -->
{% endraw %}
```

## Benefits of Using Handlebars

1. **Clean syntax** - Easy to read and maintain templates
2. **Logic-less** - Encourages separation of concerns
3. **Partials** - Reusable template components
4. **Layouts** - Consistent page structure
5. **Helpers** - Custom functions for complex logic
6. **Conditionals and loops** - Built-in support for dynamic content
7. **Escape by default** - Automatic HTML escaping for security

## Best Practices

1. **Organize templates** - Use clear directory structure
2. **Use partials** - Break down complex templates into reusable components
3. **Keep logic minimal** - Move complex logic to helpers or server-side code
4. **Validate data** - Always validate data before passing to templates
5. **Use layouts** - Maintain consistent page structure
6. **Escape output** - Let Handlebars handle HTML escaping automatically

## Next Steps

Now that you understand templating with both basic techniques and Handlebars, you have the foundation to create dynamic HTML pages that respond to user input. The remaining chapters on forms, validation, and filesystem operations will be available soon.

**[Return to Request Handling Chapter →](index.md)**
