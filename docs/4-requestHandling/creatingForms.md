---
layout: default
title: Creating Forms
nav_order: 3
parent: Request Handling
---

# Creating Forms

[&laquo; Return to Request Handling Chapter](index.md)

## Introduction to HTML Forms

Time to dip our toes in some HTML again. Forms are a simple way for the client to send POST data to the user.

## Basic HTML Form Structure

Every HTML form has these essential components:

```html
<form action="/submit" method="POST">
    <!-- Form fields go here -->
    <input type="text" name="username" required>
    <button type="submit">Submit</button>
</form>
```

### Key Attributes

- **`action`**: The URL where the form data will be sent
- **`method`**: The HTTP method (usually GET or POST)
- **`name`**: Identifies the field in the submitted data
- **`type`**: Specifies the input type (text, email, password, etc.)

## Common Form Input Types

### Text Inputs

```html
<!-- Single-line text -->
<input type="text" name="name" placeholder="Enter your name" required>

<!-- Multi-line text -->
<textarea name="message" rows="4" cols="50" placeholder="Enter your message"></textarea>

<!-- Password (hidden text) -->
<input type="password" name="password" placeholder="Enter password" required>

<!-- Email (with validation) -->
<input type="email" name="email" placeholder="Enter your email" required>

<!-- Number -->
<input type="number" name="age" min="1" max="120" placeholder="Enter your age">

```



### Selection Inputs

```html
<!-- Dropdown -->
<select name="country" required>
    <option value="">Select a country</option>
    <option value="us">United States</option>
    <option value="ca">Canada</option>
    <option value="uk">United Kingdom</option>
</select>

<!-- Radio buttons -->
<fieldset>
    <legend>Gender</legend>
    <input type="radio" id="male" name="gender" value="male">
    <label for="male">Male</label>
    
    <input type="radio" id="female" name="gender" value="female">
    <label for="female">Female</label>
    
    <input type="radio" id="other" name="gender" value="other">
    <label for="other">Other</label>
</fieldset>

<!-- Checkboxes -->
<fieldset>
    <legend>Interests</legend>
    <input type="checkbox" id="sports" name="interests" value="sports">
    <label for="sports">Sports</label>
    
    <input type="checkbox" id="music" name="interests" value="music">
    <label for="music">Music</label>
    
    <input type="checkbox" id="travel" name="interests" value="travel">
    <label for="travel">Travel</label>
</fieldset>
```

### Other Input Types

```html
<!-- Date -->
<input type="date" name="birthdate">

<!-- Time -->
<input type="time" name="appointment">

<!-- File upload -->
<input type="file" name="avatar" accept="image/*">

<!-- Hidden field (for storing data not shown to user) -->
<input type="hidden" name="timestamp" value="2024-01-01">

<!-- Submit button -->
<button type="submit">Submit Form</button>

<!-- Reset button -->
<button type="reset">Reset Form</button>
```

> ## Security Warning
> 
> Keep in mind, even if the box is trying to force a certain input from the user it doesn't guarantee that type of input will be what the server recieves. *You cannot trust anything that the client has access to*. The client can modify the HTML to bypass restrictions.
>
> This means performing your own validation and cleaning on the server end.


## Complete Contact Form Example

Let's create a comprehensive contact form with proper styling:

### HTML Form (`public/contact.html`)

```html
<!DOCTYPE html>
<html>
<head>
    <title>Contact Form</title>
</head>
<body>
    <h1>Contact Us</h1>
    
    <form action="/tschotter_node/contact" method="POST">
        <input type="text" name="name" placeholder="Your Name" required>
        <input type="email" name="email" placeholder="Your Email" required>
        <input type="text" name="subject" placeholder="Subject" required>
        <textarea name="message" placeholder="Your Message" required></textarea>
        <button type="submit">Send Message</button>
    </form>
</body>
</html>
```

## Server-Side Form Handling

Now let's create the server to handle this form:

```javascript
const express = require('express');
const path = require('path');
const app = express();
const PORT = 3000;

// Middleware
app.use(express.urlencoded({ extended: true }));
app.use(express.json());
app.use(express.static('public')); // Serve static files

// Serve the contact form
app.get('/contact', (req, res) => {
    res.sendFile(path.join(__dirname, 'public', 'contact.html'));
});

// Handle form submission
app.post('/contact', (req, res) => {
    // Extract form data
    const {
        firstName,
        lastName,
        email,
        phone,
        contactMethod,
        subject,
        message,
        company,
        website,
        interests
    } = req.body;
    
    // Log the form data (in production, you'd save to database)
    console.log('=== Contact Form Submission ===');
    console.log('Name:', `${firstName} ${lastName}`);
    console.log('Email:', email);
    console.log('Phone:', phone || 'Not provided');
    console.log('Contact Method:', contactMethod);
    console.log('Subject:', subject);
    console.log('Message:', message);
    console.log('Company:', company || 'Not provided');
    console.log('Website:', website || 'Not provided');
    console.log('Interests:', interests || 'None selected');
    console.log('================================');
    
    // Send confirmation response
    res.send(`
        <!DOCTYPE html>
        <html>
        <head>
            <title>Thank You</title>
            <style>
                body { font-family: Arial, sans-serif; max-width: 600px; margin: 50px auto; padding: 20px; }
                .success { background-color: #d4edda; border: 1px solid #c3e6cb; color: #155724; padding: 20px; border-radius: 5px; }
                .info { background-color: #f8f9fa; border: 1px solid #dee2e6; padding: 15px; border-radius: 5px; margin-top: 20px; }
            </style>
        </head>
        <body>
            <div class="success">
                <h1>Thank You, ${firstName}!</h1>
                <p>We have received your message and will get back to you soon.</p>
            </div>
            
            <div class="info">
                <h3>Submission Details:</h3>
                <p><strong>Subject:</strong> ${subject}</p>
                <p><strong>Contact Method:</strong> ${contactMethod}</p>
                <p><strong>Email:</strong> ${email}</p>
                ${phone ? `<p><strong>Phone:</strong> ${phone}</p>` : ''}
                ${company ? `<p><strong>Company:</strong> ${company}</p>` : ''}
            </div>
            
            <p><a href="/contact">Send another message</a></p>
        </body>
        </html>
    `);
});

app.listen(PORT, () => {
    console.log(`Server running on http://localhost:${PORT}`);
    console.log('Visit http://localhost:3000/contact to see the form');
});
```

## Handling Different Form Data Types

### Single Values vs Arrays

When form fields have the same name, they can create arrays:

```html
<!-- Multiple checkboxes with same name -->
<input type="checkbox" name="interests" value="sports">
<input type="checkbox" name="interests" value="music">
<input type="checkbox" name="interests" value="travel">

<!-- Multiple select -->
<select name="skills" multiple>
    <option value="javascript">JavaScript</option>
    <option value="python">Python</option>
    <option value="java">Java</option>
</select>
```

Server-side handling:

```javascript
app.post('/form', (req, res) => {
    const interests = req.body.interests; // Array: ['sports', 'music']
    const skills = req.body.skills; // Array: ['javascript', 'python']
    
    // Handle arrays
    if (Array.isArray(interests)) {
        console.log('Selected interests:', interests.join(', '));
    }
});
```

### File Uploads

For file uploads, you need additional middleware:

```bash
npm install multer
```

```javascript
const multer = require('multer');
const upload = multer({ dest: 'uploads/' });

// Single file
app.post('/upload', upload.single('avatar'), (req, res) => {
    console.log('File uploaded:', req.file);
    res.send('File uploaded successfully');
});

// Multiple files
app.post('/upload-multiple', upload.array('photos', 5), (req, res) => {
    console.log('Files uploaded:', req.files);
    res.send('Files uploaded successfully');
});
```

## Form Validation

### Client-Side Validation

HTML5 provides built-in validation:

```html
<!-- Required field -->
<input type="text" name="name" required>

<!-- Email validation -->
<input type="email" name="email" required>

<!-- Pattern validation -->
<input type="text" name="phone" pattern="[0-9]{3}-[0-9]{3}-[0-9]{4}" 
       placeholder="123-456-7890">

<!-- Length validation -->
<input type="text" name="username" minlength="3" maxlength="20">

<!-- Number range -->
<input type="number" name="age" min="18" max="100">
```

### Server-Side Validation

Always validate on the server, even if you have client-side validation:

```javascript
function validateContactForm(data) {
    const errors = [];
    
    // Required fields
    if (!data.firstName || data.firstName.trim().length < 2) {
        errors.push('First name must be at least 2 characters');
    }
    
    if (!data.lastName || data.lastName.trim().length < 2) {
        errors.push('Last name must be at least 2 characters');
    }
    
    // Email validation
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!data.email || !emailRegex.test(data.email)) {
        errors.push('Please provide a valid email address');
    }
    
    // Phone validation (optional)
    if (data.phone) {
        const phoneRegex = /^\(?([0-9]{3})\)?[-. ]?([0-9]{3})[-. ]?([0-9]{4})$/;
        if (!phoneRegex.test(data.phone)) {
            errors.push('Please provide a valid phone number');
        }
    }
    
    // Message validation
    if (!data.message || data.message.trim().length < 10) {
        errors.push('Message must be at least 10 characters');
    }
    
    return errors;
}

app.post('/contact', (req, res) => {
    const errors = validateContactForm(req.body);
    
    if (errors.length > 0) {
        return res.status(400).send(`
            <h1>Validation Errors</h1>
            <ul>
                ${errors.map(error => `<li>${error}</li>`).join('')}
            </ul>
            <a href="/contact">Try again</a>
        `);
    }
    
    // Process valid form data
    // ... rest of your code
});
```

## Best Practices for Forms

1. **Use semantic HTML** - Choose appropriate input types
2. **Provide clear labels** - Every input should have a descriptive label
3. **Group related fields** - Use fieldsets for logical grouping
4. **Validate on both sides** - Client-side for UX, server-side for security
5. **Provide helpful error messages** - Tell users exactly what's wrong
6. **Use proper HTTP methods** - POST for form submissions
7. **Handle errors gracefully** - Don't lose user data on validation errors
8. **Consider accessibility** - Use proper ARIA labels and keyboard navigation

## Next Steps

Now that you know how to create forms and handle their submissions, it's important to learn how to validate and secure the data you receive. This is crucial for preventing security vulnerabilities in your application.

**[Next: Input Validation and Security →](inputValidation.md)**
