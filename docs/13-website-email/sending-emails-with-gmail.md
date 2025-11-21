---
layout: default
title: Sending Emails with Gmail
nav_order: 1
parent: Website Email
---

# Sending Emails with Gmail

In this subchapter, you'll learn how to send emails from your Express server using Gmail. We'll cover setting up a Google account, generating App Passwords for secure authentication, and using nodemailer to send emails programmatically.

## What You'll Learn

- How to create a Google account (if you don't have one)
- How to enable 2-Step Verification and generate App Passwords
- Installing and configuring nodemailer
- Creating an Express server that sends emails
- Sending both plain text and HTML emails
- Securing email credentials with environment variables

## Step 1: Create a Google Account (If Needed)

If you don't already have a Google account, you'll need to create one:

1. Go to [accounts.google.com/signup](https://accounts.google.com/signup)
2. Fill in your information (name, username, password)
3. Complete the verification process
4. Once your account is created, you'll have access to Gmail

> **Note**: If you already have a Google account, you can skip this step and proceed to Step 2.

## Step 2: Enable 2-Step Verification

To use App Passwords (which we need for programmatic email access), you must enable 2-Step Verification on your Google account:

1. Go to [myaccount.google.com/security](https://myaccount.google.com/security)
2. Scroll down to "How you sign in to Google"
3. Click on **2-Step Verification**
4. Follow the prompts to set up 2-Step Verification
   - You'll need to verify your phone number
   - Google will send you a verification code via text or call
5. Once enabled, you'll see "2-Step Verification" marked as "On"

> **Why 2-Step Verification?** Google requires 2-Step Verification to be enabled before you can generate App Passwords. This adds an extra layer of security to your account.

## Step 3: Generate an App Password

App Passwords are special passwords that allow applications (like your Node.js server) to access your Gmail account without using your main Google password. This is more secure because:

- You can revoke App Passwords individually
- They're separate from your main account password
- If compromised, they only affect the specific application

**To generate an App Password:**

1. Go to [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)
2. You may be prompted to sign in again
3. Under "Select app", choose **Mail**
4. Under "Select device", choose **Other (Custom name)**
5. Enter a name like "Express Server" or "Node.js App"
6. Click **Generate**
7. **Important**: Copy the 16-character password that appears (it will look like: `abcd efgh ijkl mnop`)
   - You won't be able to see this password again after you close the window
   - Remove the spaces when using it (it should be 16 characters without spaces)

> **Security Note**: Treat this App Password like your regular password. Don't share it or commit it to version control. We'll use environment variables to keep it secure.

## Step 4: Set Up Your Project

Create a new project directory and initialize it:

```bash
mkdir email-server
cd email-server
npm init -y
```

Install the required dependencies:

```bash
npm install express nodemailer
```

**Package explanations:**
- **express**: Web framework for Node.js (we've used this before)
- **nodemailer**: Popular Node.js library for sending emails. It supports multiple email providers including Gmail, Outlook, and custom SMTP servers.

## Step 5: Set Up Environment Variables

Create a `.env` file in your project root (this file should NOT be committed to version control):

```bash
# .env
GMAIL_USER=your-email@gmail.com
GMAIL_APP_PASSWORD=your16characterapppassword
```

**Important Security Notes:**
- Never commit the `.env` file to Git
- Add `.env` to your `.gitignore` file
- The App Password should be 16 characters with no spaces

Create a `.gitignore` file if you don't have one:

```bash
# .gitignore
node_modules/
.env
*.log
```

## Step 6: Install dotenv (Optional but Recommended)

To automatically load environment variables from the `.env` file, install `dotenv`:

```bash
npm install dotenv
```

The code examples in the following steps will include `require('dotenv').config();` at the top of the files to automatically load your environment variables when the server starts.

## Step 7: Create the Email Configuration Module

Create a `config/email.js` file to handle email configuration:

```javascript
// config/email.js
require('dotenv').config();
const nodemailer = require('nodemailer');

// Create a transporter object using Gmail SMTP
// We'll use environment variables for sensitive information
const transporter = nodemailer.createTransport({
    service: 'gmail',
    auth: {
        user: process.env.GMAIL_USER,        // Your Gmail address
        pass: process.env.GMAIL_APP_PASSWORD // Your App Password (16 characters, no spaces)
    }
});

// What is a transporter?
// The transporter is like a "mail carrier" that knows how to connect to Gmail's email servers
// and deliver your messages. It handles:
// - Connecting to Gmail's SMTP (Simple Mail Transfer Protocol) servers
// - Authenticating with your Gmail account using your credentials
// - Sending emails through Gmail's infrastructure
// - Handling the technical details of email delivery
//
// Once created, you can reuse this transporter object to send multiple emails
// without having to reconnect each time.

// Function to send a plain text email
async function sendEmail(to, subject, text) {
    try {
        const info = await transporter.sendMail({
            from: process.env.GMAIL_USER,  // Sender address
            to: to,                         // Recipient address
            subject: subject,               // Email subject
            text: text                     // Plain text body
        });
        
        console.log('Email sent successfully:', info.messageId);
        return { success: true, messageId: info.messageId };
    } catch (error) {
        console.error('Error sending email:', error);
        return { success: false, error: error.message };
    }
}

// Function to send an HTML email
async function sendHTMLEmail(to, subject, html) {
    try {
        const info = await transporter.sendMail({
            from: process.env.GMAIL_USER,
            to: to,
            subject: subject,
            html: html  // HTML body instead of plain text
        });
        
        console.log('Email sent successfully:', info.messageId);
        return { success: true, messageId: info.messageId };
    } catch (error) {
        console.error('Error sending email:', error);
        return { success: false, error: error.message };
    }
}

// Function to send an email with both plain text and HTML
async function sendEmailWithBoth(to, subject, text, html) {
    try {
        const info = await transporter.sendMail({
            from: process.env.GMAIL_USER,
            to: to,
            subject: subject,
            text: text,  // Plain text version (for email clients that don't support HTML)
            html: html   // HTML version (for modern email clients)
        });
        
        console.log('Email sent successfully:', info.messageId);
        return { success: true, messageId: info.messageId };
    } catch (error) {
        console.error('Error sending email:', error);
        return { success: false, error: error.message };
    }
}

module.exports = {
    sendEmail,
    sendHTMLEmail,
    sendEmailWithBoth
};
```

## Step 8: Create the Express Server

Create `server.js` with routes to send emails:

```javascript
// server.js
require('dotenv').config();
const express = require('express');
const { sendEmail, sendHTMLEmail, sendEmailWithBoth } = require('./config/email');

const app = express();
const PORT = process.env.PORT || 3000;

// Middleware to parse JSON and URL-encoded data
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Route to send a plain text email
app.post('/send-email', async (req, res) => {
    const { to, subject, text } = req.body;
    
    // Validate required fields
    if (!to || !subject || !text) {
        return res.status(400).json({ 
            success: false, 
            error: 'Missing required fields: to, subject, text' 
        });
    }
    
    const result = await sendEmail(to, subject, text);
    
    if (result.success) {
        res.json({ 
            success: true, 
            message: 'Email sent successfully',
            messageId: result.messageId 
        });
    } else {
        res.status(500).json({ 
            success: false, 
            error: result.error 
        });
    }
});

// Route to send an HTML email
app.post('/send-html-email', async (req, res) => {
    const { to, subject, html } = req.body;
    
    if (!to || !subject || !html) {
        return res.status(400).json({ 
            success: false, 
            error: 'Missing required fields: to, subject, html' 
        });
    }
    
    const result = await sendHTMLEmail(to, subject, html);
    
    if (result.success) {
        res.json({ 
            success: true, 
            message: 'Email sent successfully',
            messageId: result.messageId 
        });
    } else {
        res.status(500).json({ 
            success: false, 
            error: result.error 
        });
    }
});

// Route to send an email with both plain text and HTML
app.post('/send-email-both', async (req, res) => {
    const { to, subject, text, html } = req.body;
    
    if (!to || !subject || !text || !html) {
        return res.status(400).json({ 
            success: false, 
            error: 'Missing required fields: to, subject, text, html' 
        });
    }
    
    const result = await sendEmailWithBoth(to, subject, text, html);
    
    if (result.success) {
        res.json({ 
            success: true, 
            message: 'Email sent successfully',
            messageId: result.messageId 
        });
    } else {
        res.status(500).json({ 
            success: false, 
            error: result.error 
        });
    }
});

// Test route
app.get('/', (req, res) => {
    res.send(`
        <h1>Email Server</h1>
        <p>Server is running. Use POST requests to send emails.</p>
        <h2>Endpoints:</h2>
        <ul>
            <li>POST /send-email - Send plain text email</li>
            <li>POST /send-html-email - Send HTML email</li>
            <li>POST /send-email-both - Send email with both text and HTML</li>
        </ul>
    `);
});

app.listen(PORT, () => {
    console.log(`Email server running on http://localhost:${PORT}`);
    console.log('Make sure GMAIL_USER and GMAIL_APP_PASSWORD are set in your environment variables');
});
```

## Step 9: Test Your Email Server

Start your server:

```bash
node server.js
```

**Test with curl:**

```bash
# Send a plain text email
curl -X POST http://localhost:3000/send-email \
  -H "Content-Type: application/json" \
  -d '{
    "to": "recipient@example.com",
    "subject": "Test Email",
    "text": "This is a test email from my Express server!"
  }'

# Send an HTML email
curl -X POST http://localhost:3000/send-html-email \
  -H "Content-Type: application/json" \
  -d '{
    "to": "recipient@example.com",
    "subject": "Test HTML Email",
    "html": "<h1>Hello!</h1><p>This is an <strong>HTML</strong> email.</p>"
  }'
```

**Test with a simple HTML form:**

Create `public/test-email.html`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Test Email Form</title>
</head>
<body>
    <h1>Send Test Email</h1>
    <form action="/send-email" method="POST">
        <label>To:</label><br>
        <input type="email" name="to" required><br><br>
        
        <label>Subject:</label><br>
        <input type="text" name="subject" required><br><br>
        
        <label>Message:</label><br>
        <textarea name="text" rows="5" cols="40" required></textarea><br><br>
        
        <button type="submit">Send Email</button>
    </form>
</body>
</html>
```

Don't forget to add static file serving to your server:

```javascript
// Add this to server.js
app.use(express.static('public'));
```

## Common Issues and Solutions

### Issue: "Invalid login" or "Authentication failed"

**Solutions:**
1. Make sure you're using the App Password (16 characters, no spaces), not your regular Gmail password
2. Verify that 2-Step Verification is enabled on your Google account
3. Double-check that the email address in `GMAIL_USER` matches the account where you generated the App Password
4. Ensure there are no extra spaces in your `.env` file

### Issue: "Less secure app access" error

**Solution:** Google no longer supports "Less secure app access". You must use App Passwords with 2-Step Verification enabled. Make sure you've completed Steps 2 and 3 above.

### Issue: Environment variables not loading

**Solutions:**
1. Make sure you've installed `dotenv`: `npm install dotenv`
2. Add `require('dotenv').config();` at the very top of your files (before other requires)
3. Restart your server after creating or modifying the `.env` file
4. Verify the `.env` file is in the same directory as your `server.js`

### Issue: Email not received

**Solutions:**
1. Check your spam/junk folder
2. Verify the recipient email address is correct
3. Check the server console for error messages
4. Make sure your Gmail account isn't restricted or suspended

## Best Practices

1. **Never commit credentials**: Always use environment variables and add `.env` to `.gitignore`
2. **Use App Passwords**: Never use your main Google account password in code
3. **Handle errors gracefully**: Always wrap email sending in try-catch blocks
4. **Validate input**: Check that required fields are present before sending emails
5. **Rate limiting**: Consider implementing rate limiting to prevent abuse (we'll cover this in future chapters)
6. **Email templates**: For production, consider using email templating libraries like `handlebars` or `ejs`

## Next Steps

Now that you can send emails from your Express server, you can:
- Send welcome emails when users register
- Send password reset links
- Send notification emails for important events
- Create email newsletters
- Send order confirmations for e-commerce sites

---

**[Previous: Website Email Overview](index.md)** | **[Return to Table of Contents](../../index.md)**

