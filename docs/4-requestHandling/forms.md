---
layout: default
title: Forms
nav_order: 3
parent: Request Handling
---

# Forms

[&laquo; Return to Request Handling Chapter](index.md)

## Introduction to Forms

Forms are the primary way users interact with web applications by submitting data to servers. Understanding how to create HTML forms and handle form submissions on the server side is essential for building interactive web applications.

In this chapter, we'll cover:
1. **HTML form elements** - The building blocks of forms
2. **Form submission methods** - GET vs POST for forms
3. **Server-side form handling** - Processing form data with Express
4. **Form validation** - Basic client and server-side validation
5. **Best practices** - Security and user experience considerations

## HTML Form Elements

Before we can handle forms on the server, we need to understand the HTML elements that make up forms.

### Basic Form Structure

Every form starts with a `<form>` element that defines how and where the data will be submitted:

```html
<form action="/submit" method="POST">
    <!-- Form fields go here -->
</form>
```

**Key attributes:**
- `action` - The URL where the form data will be sent
- `method` - The HTTP method (GET or POST)

### Common Input Types

#### Text Inputs

```html
<!-- Single-line text input -->
<input type="text" name="username" placeholder="Enter username" required>

<!-- Email input (with built-in validation) -->
<input type="email" name="email" placeholder="Enter email" required>

<!-- Password input (hidden text) -->
<input type="password" name="password" placeholder="Enter password" required>

<!-- Number input -->
<input type="number" name="age" min="1" max="120" placeholder="Enter age">

<!-- URL input -->
<input type="url" name="website" placeholder="Enter website URL">
```

#### Selection Inputs

```html
<!-- Dropdown selection -->
<select name="country" required>
    <option value="">Select a country</option>
    <option value="us">United States</option>
    <option value="ca">Canada</option>
    <option value="uk">United Kingdom</option>
</select>

<!-- Radio buttons (single selection) -->
<div>
    <label>Gender:</label>
    <input type="radio" name="gender" value="male" id="male">
    <label for="male">Male</label>
    
    <input type="radio" name="gender" value="female" id="female">
    <label for="female">Female</label>
    
    <input type="radio" name="gender" value="other" id="other">
    <label for="other">Other</label>
</div>

<!-- Checkboxes (multiple selection) -->
<div>
    <label>Interests:</label>
    <input type="checkbox" name="interests" value="programming" id="prog">
    <label for="prog">Programming</label>
    
    <input type="checkbox" name="interests" value="design" id="design">
    <label for="design">Design</label>
    
    <input type="checkbox" name="interests" value="music" id="music">
    <label for="music">Music</label>
</div>
```

#### Other Input Types

```html
<!-- Text area (multi-line text) -->
<textarea name="message" rows="4" cols="50" placeholder="Enter your message"></textarea>

<!-- File upload -->
<input type="file" name="avatar" accept="image/*">

<!-- Hidden input (for passing data without user interaction) -->
<input type="hidden" name="timestamp" value="2024-01-15">

<!-- Submit button -->
<button type="submit">Submit Form</button>

<!-- Reset button -->
<button type="reset">Reset Form</button>
```

### Simple Form Example

Here's a basic contact form with just two fields:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Contact Form</title>
    <style>
        body { font-family: Arial, sans-serif; max-width: 400px; margin: 0 auto; padding: 20px; }
        .form-group { margin-bottom: 15px; }
        label { display: block; margin-bottom: 5px; font-weight: bold; }
        input, textarea { width: 100%; padding: 8px; border: 1px solid #ddd; border-radius: 4px; }
        button { background-color: #007bff; color: white; padding: 10px 20px; border: none; border-radius: 4px; cursor: pointer; }
        button:hover { background-color: #0056b3; }
    </style>
</head>
<body>
    <h1>Contact Us</h1>
    
    <form action="/contact" method="POST">
        <div class="form-group">
            <label for="name">Name *</label>
            <input type="text" id="name" name="name" required>
        </div>
        
        <div class="form-group">
            <label for="message">Message *</label>
            <textarea id="message" name="message" rows="4" required placeholder="Your message..."></textarea>
        </div>
        
        <button type="submit">Send Message</button>
    </form>
</body>
</html>
```

## Form Submission Methods

### GET vs POST for Forms

**When to use GET:**
- Search forms
- Filtering data
- Any form that doesn't modify server data
- When you want the form data visible in the URL

**When to use POST:**
- Contact forms
- User registration
- File uploads
- Any form that modifies server data
- When form data should be private

### GET Form Example

```html
<!-- Search form using GET -->
<form action="/search" method="GET">
    <input type="text" name="query" placeholder="Search..." required>
    <input type="text" name="category" placeholder="Category" required>
    <button type="submit">Search</button>
</form>
```

When submitted, this creates a URL like: `/search?query=javascript&category=articles`

### POST Form Example

```html
<!-- Contact form using POST -->
<form action="/contact" method="POST">
    <input type="text" name="name" placeholder="Your name" required>
    <textarea name="message" placeholder="Your message" required></textarea>
    <button type="submit">Send Message</button>
</form>
```

The form data is sent in the request body, not in the URL.

## Server-Side Form Handling with Express

Now let's create an Express server to handle form submissions.

### Basic Express Server Setup

```javascript
const express = require('express');
const path = require('path');
const app = express();
const PORT = 3000;

// Middleware to parse form data
app.use(express.urlencoded({ extended: true }));
app.use(express.json());

// Serve static files (for our HTML forms)
app.use(express.static('public'));

// Routes
app.get('/', (req, res) => {
    res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

app.listen(PORT, () => {
    console.log(`Server running on http://localhost:${PORT}`);
});
```

### Handling GET Form Submissions

```javascript
// Handle search form (GET request)
app.get('/search', (req, res) => {
    const { query, category } = req.query;
    
    // Simple search results
    const results = [
        { title: 'JavaScript Basics', category: 'articles' },
        { title: 'Node.js Tutorial', category: 'articles' },
        { title: 'Web Development Course', category: 'courses' }
    ];
    
    // Filter results
    let filteredResults = results;
    
    if (query) {
        filteredResults = filteredResults.filter(item => 
            item.title.toLowerCase().includes(query.toLowerCase())
        );
    }
    
    if (category) {
        filteredResults = filteredResults.filter(item => 
            item.category === category
        );
    }
    
    // Send results as HTML
    res.send(`
        <h1>Search Results</h1>
        <p>Searching for: "${query}" in category "${category}"</p>
        <p>Found ${filteredResults.length} results:</p>
        <ul>
            ${filteredResults.map(item => 
                `<li>${item.title} (${item.category})</li>`
            ).join('')}
        </ul>
        <a href="/">← Back to Search</a>
    `);
});
```

### Handling POST Form Submissions

```javascript
// Handle contact form (POST request)
app.post('/contact', (req, res) => {
    const { name, message } = req.body;
    
    // Basic validation
    if (!name || !message) {
        return res.status(400).send(`
            <h1>Error</h1>
            <p>Name and message are required fields.</p>
            <a href="/">← Back to Form</a>
        `);
    }
    
    // Log the form data (in a real app, you'd save to database)
    console.log('Contact form submission:', {
        name,
        message,
        timestamp: new Date().toISOString()
    });
    
    // Send success response
    res.send(`
        <h1>Thank You!</h1>
        <p>Hi ${name}, thank you for your message!</p>
        <p>We'll get back to you as soon as possible.</p>
        <a href="/">← Back to Home</a>
    `);
});
```

### Complete Server Example

Here's a complete Express server that handles both forms:

```javascript
const express = require('express');
const path = require('path');
const app = express();
const PORT = 3000;

// Middleware
app.use(express.urlencoded({ extended: true }));
app.use(express.json());
app.use(express.static('public'));

// In-memory storage (use database in production)
let contacts = [];

// Home page
app.get('/', (req, res) => {
    res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

// Handle search form (GET)
app.get('/search', (req, res) => {
    const { query, category } = req.query;
    
    // Simple search results
    const results = [
        { title: 'JavaScript Basics', category: 'articles' },
        { title: 'Node.js Tutorial', category: 'articles' },
        { title: 'Web Development Course', category: 'courses' }
    ];
    
    // Filter results
    let filteredResults = results;
    
    if (query) {
        filteredResults = filteredResults.filter(item => 
            item.title.toLowerCase().includes(query.toLowerCase())
        );
    }
    
    if (category) {
        filteredResults = filteredResults.filter(item => 
            item.category === category
        );
    }
    
    // Send results as HTML
    res.send(`
        <h1>Search Results</h1>
        <p>Searching for: "${query}" in category "${category}"</p>
        <p>Found ${filteredResults.length} results:</p>
        <ul>
            ${filteredResults.map(item => 
                `<li>${item.title} (${item.category})</li>`
            ).join('')}
        </ul>
        <a href="/">← Back to Search</a>
    `);
});

// Handle contact form (POST)
app.post('/contact', (req, res) => {
    const { name, message } = req.body;
    
    // Validation
    if (!name || !message) {
        return res.status(400).send(`
            <h1>Error</h1>
            <p>Name and message are required fields.</p>
            <a href="/">← Back to Form</a>
        `);
    }
    
    // Create contact object
    const contact = {
        name: name.trim(),
        message: message.trim(),
        timestamp: new Date().toISOString()
    };
    
    // Store contact
    contacts.push(contact);
    
    // Send success response
    res.send(`
        <h1>Thank You, ${contact.name}!</h1>
        <p>Your message has been received successfully.</p>
        <p><strong>Your message:</strong></p>
        <blockquote>${contact.message}</blockquote>
        <a href="/">← Back to Home</a>
    `);
});

// Display all contacts (for demonstration)
app.get('/contacts', (req, res) => {
    res.send(`
        <h1>All Contacts</h1>
        <p>Total contacts: ${contacts.length}</p>
        <ul>
            ${contacts.map(contact => `
                <li>
                    <strong>${contact.name}</strong>
                    <br>Message: ${contact.message}
                    <br>Submitted: ${contact.timestamp}
                </li>
            `).join('')}
        </ul>
        <a href="/">← Back to Home</a>
    `);
});

app.listen(PORT, () => {
    console.log(`Server running on http://localhost:${PORT}`);
    console.log('Available routes:');
    console.log('  GET  / - Home page with forms');
    console.log('  GET  /search - Search form');
    console.log('  POST /contact - Contact form submission');
    console.log('  GET  /contacts - View all contacts');
});
```

## Form Validation

### Client-Side Validation

HTML5 provides built-in validation attributes:

```html
<form action="/contact" method="POST">
    <!-- Required field -->
    <input type="text" name="name" required>
    
    <!-- Email validation -->
    <input type="email" name="email" required>
    
    <!-- Number with range -->
    <input type="number" name="age" min="13" max="120">
    
    <!-- Text length limits -->
    <textarea name="message" minlength="10" maxlength="500" required></textarea>
    
    <!-- Pattern matching -->
    <input type="text" name="phone" pattern="[0-9]{3}-[0-9]{3}-[0-9]{4}" 
           placeholder="123-456-7890">
    
    <!-- Custom validation message -->
    <input type="text" name="username" required 
           oninvalid="this.setCustomValidity('Username is required')"
           oninput="this.setCustomValidity('')">
    
    <button type="submit">Submit</button>
</form>
```

### Server-Side Validation

Always validate on the server, even if you have client-side validation:

```javascript
app.post('/contact', (req, res) => {
    const { name, message } = req.body;
    
    // Validation errors array
    const errors = [];
    
    // Required field validation
    if (!name || name.trim().length === 0) {
        errors.push('Name is required');
    }
    
    if (!message || message.trim().length === 0) {
        errors.push('Message is required');
    }
    
    // Length validation
    if (name && name.trim().length < 2) {
        errors.push('Name must be at least 2 characters long');
    }
    
    if (message && message.trim().length < 10) {
        errors.push('Message must be at least 10 characters long');
    }
    
    // If there are errors, return them
    if (errors.length > 0) {
        return res.status(400).send(`
            <h1>Validation Errors</h1>
            <ul>
                ${errors.map(error => `<li>${error}</li>`).join('')}
            </ul>
            <a href="/">← Back to Form</a>
        `);
    }
    
    // Process the form if validation passes
    // ... rest of your form handling code
});
```

## Handling Different Input Types

### Checkboxes and Radio Buttons

```javascript
app.post('/survey', (req, res) => {
    const { name, interests } = req.body;
    
    // interests will be an array if multiple checkboxes are selected
    
    console.log('Survey data:', {
        name,
        interests: Array.isArray(interests) ? interests : (interests ? [interests] : [])
    });
    
    res.send(`
        <h1>Survey Submitted!</h1>
        <p>Name: ${name}</p>
        <p>Interests: ${Array.isArray(interests) ? interests.join(', ') : 'None selected'}</p>
    `);
});
```


## Best Practices

### Security Considerations

1. **Always validate server-side** - Never trust client-side validation alone
2. **Sanitize input** - Clean user input to prevent XSS attacks
3. **Use HTTPS** - Protect form data in transit
4. **Rate limiting** - Prevent spam and abuse
5. **CSRF protection** - Use CSRF tokens for state-changing operations

### User Experience

1. **Clear labels** - Use descriptive labels for all form fields
2. **Helpful error messages** - Provide specific, actionable error messages
3. **Progress indicators** - Show users what's happening during submission
4. **Confirmation pages** - Confirm successful submissions
5. **Accessibility** - Use proper form structure and ARIA attributes

### Code Organization

1. **Separate concerns** - Keep validation, processing, and response logic separate
2. **Reusable validation** - Create validation functions you can reuse
3. **Error handling** - Handle errors gracefully
4. **Logging** - Log form submissions for debugging and analytics

## Common Form Patterns

### Multi-Step Forms

```javascript
// Step 1: Personal Information
app.get('/signup/step1', (req, res) => {
    res.sendFile(path.join(__dirname, 'public', 'signup-step1.html'));
});

app.post('/signup/step1', (req, res) => {
    // Store step 1 data in session or temporary storage
    req.session.step1Data = req.body;
    res.redirect('/signup/step2');
});

// Step 2: Preferences
app.get('/signup/step2', (req, res) => {
    res.sendFile(path.join(__dirname, 'public', 'signup-step2.html'));
});

app.post('/signup/step2', (req, res) => {
    // Combine step 1 and step 2 data
    const userData = {
        ...req.session.step1Data,
        ...req.body
    };
    
    // Process complete user data
    // ... save to database
    
    res.redirect('/signup/success');
});
```

### AJAX Form Submission

```html
<!-- HTML form with AJAX submission -->
<form id="contactForm">
    <input type="text" name="name" required>
    <textarea name="message" required></textarea>
    <button type="submit">Send Message</button>
</form>

<div id="result"></div>

<script>
document.getElementById('contactForm').addEventListener('submit', async (e) => {
    e.preventDefault();
    
    const formData = new FormData(e.target);
    const resultDiv = document.getElementById('result');
    
    try {
        const response = await fetch('/contact', {
            method: 'POST',
            body: formData
        });
        
        const result = await response.text();
        resultDiv.innerHTML = result;
        
        if (response.ok) {
            e.target.reset(); // Clear form on success
        }
    } catch (error) {
        resultDiv.innerHTML = '<p>Error submitting form. Please try again.</p>';
    }
});
</script>
```

## Next Steps

Now that you understand HTML forms and server-side form handling, you have the foundation to build interactive web applications that can collect and process user data. The next logical steps would be:

1. **Database integration** - Store form data in a database
2. **Advanced validation** - More sophisticated validation libraries
3. **File handling** - Upload and process files
4. **Authentication** - User login and registration systems
5. **API development** - Create RESTful APIs for form data

**[Return to Request Handling Chapter →](index.md)**
