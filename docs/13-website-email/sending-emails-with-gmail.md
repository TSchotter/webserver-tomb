---
layout: default
title: Sending Emails with Gmail
nav_order: 1
parent: Website Email
---

# Sending Emails with Gmail

In this subchapter, you'll learn how to send emails from Node.js using Gmail. We'll cover setting up a Google account, generating App Passwords for secure authentication, and using nodemailer to send emails programmatically.

## What You'll Learn

- How to create a Google account (if you don't have one)
- How to enable 2-Step Verification and generate App Passwords
- Installing and configuring nodemailer
- Creating a Node.js script that sends emails
- Sending both plain text and HTML emails
- Securing email credentials with environment variables

## Create a Google Account (If Needed)

If you don't already have a Google account, you'll need to create one:

1. Go to [accounts.google.com/signup](https://accounts.google.com/signup)
2. Fill in your information (name, username, password)
3. Complete the verification process
4. Once your account is created, you'll have access to Gmail

> **Note**: If you already have a Google account, you can skip this step and proceed to Step 2.

## Enable 2-Step Verification

To use App Passwords (which we need for programmatic email access), you must enable 2-Step Verification on your Google account:

1. Go to [myaccount.google.com/security](https://myaccount.google.com/security)
2. Scroll down to "How you sign in to Google"
3. Click on **2-Step Verification**
4. Follow the prompts to set up 2-Step Verification
   - You'll need to verify your phone number
   - Google will send you a verification code via text or call
5. Once enabled, you'll see "2-Step Verification" marked as "On"

> **Why 2-Step Verification?** Google requires 2-Step Verification to be enabled before you can generate App Passwords. This adds an extra layer of security to your account.

## Generate an App Password

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

## Set Up Your Project

Create a new project directory and initialize it:

```bash
mkdir email-server
cd email-server
npm init -y
```

Install the required dependencies:

```bash
npm install nodemailer
```

**Package explanation:**
- **nodemailer**: Popular Node.js library for sending emails. It supports multiple email providers including Gmail, Outlook, and custom SMTP servers.

## Set Up Environment Variables

Create a `.env` file in your project root (this file should NOT be committed to version control). Only sensitive credentials go here:

```bash
# .env
GMAIL_USER=your-email@gmail.com
GMAIL_APP_PASSWORD=your16characterapppassword
```

**Important Security Notes:**
- Never commit the `.env` file to Git
- Add `.env` to your `.gitignore` file
- The App Password should be 16 characters with no spaces
- Only put sensitive credentials (Gmail user and password) in the `.env` file
- Email content (to, subject, text) will be hardcoded in your script

Create a `.gitignore` file if you don't have one:

```bash
# .gitignore
node_modules/
.env
*.log
```

> **Note**: We'll use Node.js's built-in `--env-file` flag (available in Node.js 20.6.0+) to load these environment variables. This eliminates the need for the `dotenv` package.

## Create the Email Configuration Module

Create a `config/email.js` file to handle email configuration:

```javascript
// config/email.js
const nodemailer = require('nodemailer');

// Create a transporter object using Gmail SMTP
// We'll use environment variables for sensitive information
// These will be loaded from .env file using node --env-file=.env
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


module.exports = {
    sendEmail
};
```

## Create the Email Sending Script

Create `send-email.js` - a simple Node.js script that sends an email:

```javascript
// send-email.js
const { sendEmail } = require('./config/email');

// Check that required environment variables are set
if (!process.env.GMAIL_USER || !process.env.GMAIL_APP_PASSWORD) {
    console.error('Error: GMAIL_USER and GMAIL_APP_PASSWORD environment variables must be set');
    console.error('Make sure you have a .env file and run: node --env-file=.env send-email.js');
    process.exit(1);
}

// Example: Send a plain text email
async function main() {
    // Hardcode the email details
    const recipient = 'recipient@example.com';
    const subject = 'Test Email from Node.js';
    const text = 'This is a test email sent from a Node.js script!';
    
    console.log('Sending email...');
    console.log(`To: ${recipient}`);
    console.log(`Subject: ${subject}`);
    
    const result = await sendEmail(recipient, subject, text);
    
    if (result.success) {
        console.log('Email sent successfully!');
        console.log(`Message ID: ${result.messageId}`);
    } else {
        console.error('Failed to send email:', result.error);
        process.exit(1);
    }
}

// Run the script
main().catch(error => {
    console.error('Error:', error);
    process.exit(1);
});
```

## Test Your Email Script

Make sure your `.env` file contains only your Gmail credentials:

```bash
# .env
GMAIL_USER=your-email@gmail.com
GMAIL_APP_PASSWORD=your16characterapppassword
```

Edit `send-email.js` to set the recipient, subject, and message text you want to send, then run the script using Node.js's `--env-file` flag:

```bash
node --env-file=.env send-email.js
```

The `--env-file` flag automatically loads all environment variables from the `.env` file into `process.env`. The email content (to, subject, text) is hardcoded in your script, so you can easily modify it before running.

> **Note**: The `--env-file` flag requires Node.js 20.6.0 or later. You can check your Node.js version with `node --version`.

---

**[Previous: Website Email Overview](index.md)** | **[Return to Table of Contents](../../index.md)**

