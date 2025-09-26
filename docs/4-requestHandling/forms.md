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
- **HTML form elements** - The building blocks of forms
- **Form submission methods** - GET vs POST
- **Server-side form handling** - Processing form data with Express
- **Form validation** - Basic client and server-side validation
- **Best practices** - Some basic security and user experience considerations

## HTML Form Elements

While this chapter is not going to go in depth into the form elements, there is a certain level of base knowledge that is necessary to be capable of using forms (even if we don't end up ever writing them from scratch)

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

> All of these are built into html and you don't need any special css or javascript to give them functionality. It **IS** recommended that you have css styling them the way you want though.


### Simple Form Example

Here's a basic contact form with just two fields:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <title>Contact Form</title>
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

> Some aspects that might look unfamiliar.
>
> - labels are effectively text that is attached to input fields. This is useful for accessibility rather than just having a standard text field.
> - When an input field has "required" in the tag, it doesn't allow the client to hit the submit button without having it filled. *This does not mean it's not possible to have the data sent to the server*. Since this is a form of validation that is on the HTML page, it's a form of validation that the client can modify.
> - The placeholder attribute shows text in the input field before the user types something in.

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

// Insert request handling callback functions here.

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
    
    // Send results as HTML (modify this to suit your needs)
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

// In-memory storage (we'll be covering how to use databases instead)

let contacts = [];


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
> ### SECURITY NOTE
>
> Always consider any form of client-side validation with the goal of making the site more user-friendly. It should never be considered true validation.
>
> True validation of information must be done on the server.

### Server-Side Validation


```javascript
app.post('/contact', (req, res) => {
    const { name, message } = req.body;
    
    // Validation errors array
    const errors = [];
    
    // Required field validation
    if (!name || name.trim().length === 0) {
        // "push" is the same as "append". Here we are appending an error to an error list so that we can tell the client all the errors at once rather than the first one we see.
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
        // "map" is a way to loop through a list and have each element of the list return something into a new array. Here, each element of the errors list will be assigned as "error", and spit it out as a list element string.

        // Finally, they are joined together with "join" because we need all the elements as a single string rather than an array of strings.
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
> ### Other validation considerations
>
> - Malicious input. Clients may try to insert commands (especially SQL commands if they know the information is going to a database). You will need to properly escape their input so that it doesn't cause issues.
> - Duplicate data. Sometimes information that's already sent to your server will be sent again. Consider how you want to handle that.



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

- **Always validate server-side** - Never trust client-side validation alone
- **Sanitize input** - Clean user input to prevent XSS attacks (will be covered later)
- **Use HTTPS** - Protect form data in transit (again, we will cover this later)

### User Experience

- **Clear labels** - Use descriptive labels for all form fields
- **Helpful error messages** - Provide specific, actionable error messages
- **Confirmation pages** - Confirm successful submissions
- **Accessibility** - Use proper form structure

### Code Organization

- **Separate concerns** - Keep validation, processing, and response logic separate. This will keep your code organized enough to understand what's happening and how to add to it.
- **Reusable validation** - Create validation functions you can reuse
- **Error handling** - Handle errors gracefully
- **Logging** - Log form submissions for debugging and analytics (we'll talk about logging information later)


## Next Steps

Now that you understand the basics of HTML forms and server-side form handling, we'll want to cover how to modify files and communicate with a database to act as more permanent memory storage.


**[Return to Request Handling Chapter →](index.md)**
