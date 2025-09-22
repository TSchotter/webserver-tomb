---
layout: default
title: Filesystem Operations
nav_order: 5
parent: Request Handling
---

# Filesystem Operations

[&laquo; Return to Request Handling Chapter](index.md)

## Introduction to Filesystem Operations

So far, we've learned how to handle requests and validate input, but the data we process disappears when the server restarts. To build persistent applications, we need to store data in files or databases. In this chapter, we'll focus on using the filesystem to store and retrieve data based on user requests.

## Node.js File System Module

Node.js provides the `fs` module for file system operations. We'll use both synchronous and asynchronous methods:

```javascript
const fs = require('fs');
const path = require('path');
```

### Basic File Operations

```javascript
// Writing data to a file
const data = { name: 'John', email: 'john@example.com' };
fs.writeFileSync('data.json', JSON.stringify(data, null, 2));

// Reading data from a file
const fileData = fs.readFileSync('data.json', 'utf8');
const parsedData = JSON.parse(fileData);
console.log(parsedData); // { name: 'John', email: 'john@example.com' }

// Checking if a file exists
if (fs.existsSync('data.json')) {
    console.log('File exists');
}

// Creating a directory
if (!fs.existsSync('data')) {
    fs.mkdirSync('data');
}
```

## Building a Guestbook Application

Let's create a complete guestbook application that demonstrates filesystem operations:

### Project Structure

```
guestbook-app/
├── package.json
├── server.js
├── public/
│   └── guestbook.html
├── data/
│   └── messages.json (created automatically)
└── uploads/ (for future file uploads)
```

### Server Implementation

```javascript
const express = require('express');
const fs = require('fs');
const path = require('path');
const app = express();
const PORT = 3000;

// Middleware
app.use(express.urlencoded({ extended: true }));
app.use(express.json());
app.use(express.static('public'));

// File paths
const DATA_DIR = path.join(__dirname, 'data');
const MESSAGES_FILE = path.join(DATA_DIR, 'messages.json');
const UPLOADS_DIR = path.join(__dirname, 'uploads');

// Initialize directories and files
function initializeApp() {
    // Create data directory if it doesn't exist
    if (!fs.existsSync(DATA_DIR)) {
        fs.mkdirSync(DATA_DIR);
        console.log('Created data directory');
    }
    
    // Create uploads directory if it doesn't exist
    if (!fs.existsSync(UPLOADS_DIR)) {
        fs.mkdirSync(UPLOADS_DIR);
        console.log('Created uploads directory');
    }
    
    // Initialize messages file if it doesn't exist
    if (!fs.existsSync(MESSAGES_FILE)) {
        const initialData = {
            messages: [],
            lastId: 0,
            createdAt: new Date().toISOString()
        };
        fs.writeFileSync(MESSAGES_FILE, JSON.stringify(initialData, null, 2));
        console.log('Created messages.json file');
    }
}

// File operations
class FileManager {
    static readMessages() {
        try {
            const data = fs.readFileSync(MESSAGES_FILE, 'utf8');
            return JSON.parse(data);
        } catch (error) {
            console.error('Error reading messages:', error);
            return { messages: [], lastId: 0 };
        }
    }
    
    static writeMessages(data) {
        try {
            fs.writeFileSync(MESSAGES_FILE, JSON.stringify(data, null, 2));
            return true;
        } catch (error) {
            console.error('Error writing messages:', error);
            return false;
        }
    }
    
    static addMessage(messageData) {
        const data = this.readMessages();
        
        const newMessage = {
            id: ++data.lastId,
            ...messageData,
            createdAt: new Date().toISOString(),
            ip: messageData.ip || 'unknown'
        };
        
        data.messages.unshift(newMessage); // Add to beginning
        
        if (this.writeMessages(data)) {
            return newMessage;
        } else {
            throw new Error('Failed to save message');
        }
    }
    
    static getAllMessages() {
        const data = this.readMessages();
        return data.messages;
    }
    
    static getMessageById(id) {
        const data = this.readMessages();
        return data.messages.find(msg => msg.id === parseInt(id));
    }
    
    static deleteMessage(id) {
        const data = this.readMessages();
        const messageIndex = data.messages.findIndex(msg => msg.id === parseInt(id));
        
        if (messageIndex === -1) {
            return false;
        }
        
        data.messages.splice(messageIndex, 1);
        return this.writeMessages(data);
    }
    
    static searchMessages(query) {
        const messages = this.getAllMessages();
        const searchTerm = query.toLowerCase();
        
        return messages.filter(msg => 
            msg.name.toLowerCase().includes(searchTerm) ||
            msg.message.toLowerCase().includes(searchTerm) ||
            msg.email.toLowerCase().includes(searchTerm)
        );
    }
}

// Input validation (from previous chapter)
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
    
    static validateName(name) {
        if (typeof name !== 'string') return false;
        const sanitized = this.sanitizeHTML(name);
        return sanitized.length >= 2 && sanitized.length <= 50 && /^[a-zA-Z\s'-]+$/.test(sanitized);
    }
    
    static validateEmail(email) {
        if (typeof email !== 'string') return false;
        const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        return emailRegex.test(email) && email.length <= 254;
    }
    
    static validateMessage(message) {
        if (typeof message !== 'string') return false;
        const sanitized = this.sanitizeHTML(message);
        return sanitized.length >= 5 && sanitized.length <= 500;
    }
}

// Validation middleware
function validateGuestbookEntry(req, res, next) {
    const errors = [];
    const sanitizedData = {};
    
    const { name, email, message } = req.body;
    
    // Validate name
    if (!name) {
        errors.push('Name is required');
    } else if (!InputValidator.validateName(name)) {
        errors.push('Name must be 2-50 characters and contain only letters, spaces, hyphens, and apostrophes');
    } else {
        sanitizedData.name = InputValidator.sanitizeHTML(name);
    }
    
    // Validate email
    if (!email) {
        errors.push('Email is required');
    } else if (!InputValidator.validateEmail(email)) {
        errors.push('Please provide a valid email address');
    } else {
        sanitizedData.email = email.toLowerCase().trim();
    }
    
    // Validate message
    if (!message) {
        errors.push('Message is required');
    } else if (!InputValidator.validateMessage(message)) {
        errors.push('Message must be 5-500 characters long');
    } else {
        sanitizedData.message = InputValidator.sanitizeHTML(message);
    }
    
    if (errors.length > 0) {
        return res.status(400).json({
            success: false,
            errors: errors
        });
    }
    
    req.sanitizedData = sanitizedData;
    next();
}

// Routes
app.get('/', (req, res) => {
    res.sendFile(path.join(__dirname, 'public', 'guestbook.html'));
});

// Get all messages (API endpoint)
app.get('/api/messages', (req, res) => {
    try {
        const messages = FileManager.getAllMessages();
        res.json({
            success: true,
            count: messages.length,
            messages: messages
        });
    } catch (error) {
        res.status(500).json({
            success: false,
            error: 'Failed to retrieve messages'
        });
    }
});

// Get specific message
app.get('/api/messages/:id', (req, res) => {
    try {
        const message = FileManager.getMessageById(req.params.id);
        if (message) {
            res.json({
                success: true,
                message: message
            });
        } else {
            res.status(404).json({
                success: false,
                error: 'Message not found'
            });
        }
    } catch (error) {
        res.status(500).json({
            success: false,
            error: 'Failed to retrieve message'
        });
    }
});

// Search messages
app.get('/api/messages/search/:query', (req, res) => {
    try {
        const results = FileManager.searchMessages(req.params.query);
        res.json({
            success: true,
            count: results.length,
            query: req.params.query,
            messages: results
        });
    } catch (error) {
        res.status(500).json({
            success: false,
            error: 'Search failed'
        });
    }
});

// Add new message
app.post('/api/messages', validateGuestbookEntry, (req, res) => {
    try {
        const messageData = {
            ...req.sanitizedData,
            ip: req.ip || req.connection.remoteAddress
        };
        
        const newMessage = FileManager.addMessage(messageData);
        
        res.status(201).json({
            success: true,
            message: 'Message added successfully',
            data: newMessage
        });
    } catch (error) {
        res.status(500).json({
            success: false,
            error: 'Failed to save message'
        });
    }
});

// Delete message (admin function)
app.delete('/api/messages/:id', (req, res) => {
    try {
        const success = FileManager.deleteMessage(req.params.id);
        if (success) {
            res.json({
                success: true,
                message: 'Message deleted successfully'
            });
        } else {
            res.status(404).json({
                success: false,
                error: 'Message not found'
            });
        }
    } catch (error) {
        res.status(500).json({
            success: false,
            error: 'Failed to delete message'
        });
    }
});

// Get statistics
app.get('/api/stats', (req, res) => {
    try {
        const messages = FileManager.getAllMessages();
        const data = FileManager.readMessages();
        
        const stats = {
            totalMessages: messages.length,
            firstMessage: data.createdAt,
            lastMessage: messages.length > 0 ? messages[0].createdAt : null,
            uniqueEmails: new Set(messages.map(msg => msg.email)).size,
            messagesToday: messages.filter(msg => {
                const today = new Date().toDateString();
                const messageDate = new Date(msg.createdAt).toDateString();
                return today === messageDate;
            }).length
        };
        
        res.json({
            success: true,
            stats: stats
        });
    } catch (error) {
        res.status(500).json({
            success: false,
            error: 'Failed to retrieve statistics'
        });
    }
});

// Error handling middleware
app.use((err, req, res, next) => {
    console.error('Error:', err);
    res.status(500).json({
        success: false,
        error: 'Internal server error'
    });
});

// Initialize app and start server
initializeApp();

app.listen(PORT, () => {
    console.log(`Guestbook server running on http://localhost:${PORT}`);
    console.log('Visit http://localhost:3000 to see the guestbook');
    console.log('API endpoints:');
    console.log('  GET  /api/messages - Get all messages');
    console.log('  GET  /api/messages/:id - Get specific message');
    console.log('  GET  /api/messages/search/:query - Search messages');
    console.log('  POST /api/messages - Add new message');
    console.log('  DELETE /api/messages/:id - Delete message');
    console.log('  GET  /api/stats - Get statistics');
});
```

### HTML Frontend (`public/guestbook.html`)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Guestbook</title>
    <style>
        * {
            box-sizing: border-box;
        }
        
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
            background-color: #f5f5f5;
        }
        
        .container {
            background-color: white;
            padding: 30px;
            border-radius: 10px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        
        h1 {
            color: #333;
            text-align: center;
            margin-bottom: 30px;
        }
        
        .form-section, .messages-section {
            margin-bottom: 40px;
        }
        
        .form-group {
            margin-bottom: 20px;
        }
        
        label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
            color: #555;
        }
        
        input, textarea {
            width: 100%;
            padding: 12px;
            border: 2px solid #ddd;
            border-radius: 5px;
            font-size: 16px;
            transition: border-color 0.3s;
        }
        
        input:focus, textarea:focus {
            border-color: #007bff;
            outline: none;
        }
        
        button {
            background-color: #007bff;
            color: white;
            padding: 12px 30px;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            font-size: 16px;
            transition: background-color 0.3s;
        }
        
        button:hover {
            background-color: #0056b3;
        }
        
        .message {
            border: 1px solid #ddd;
            padding: 20px;
            margin-bottom: 20px;
            border-radius: 8px;
            background-color: #f9f9f9;
        }
        
        .message-header {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-bottom: 10px;
        }
        
        .message-name {
            font-weight: bold;
            color: #007bff;
            font-size: 18px;
        }
        
        .message-date {
            color: #666;
            font-size: 14px;
        }
        
        .message-content {
            line-height: 1.6;
            color: #333;
        }
        
        .message-email {
            color: #666;
            font-size: 14px;
            margin-top: 10px;
        }
        
        .search-box {
            margin-bottom: 20px;
        }
        
        .search-box input {
            width: 70%;
            display: inline-block;
        }
        
        .search-box button {
            width: 28%;
            margin-left: 2%;
        }
        
        .stats {
            background-color: #e9ecef;
            padding: 15px;
            border-radius: 5px;
            margin-bottom: 20px;
        }
        
        .stats h3 {
            margin-top: 0;
            color: #495057;
        }
        
        .error {
            color: #dc3545;
            background-color: #f8d7da;
            border: 1px solid #f5c6cb;
            padding: 10px;
            border-radius: 5px;
            margin-bottom: 20px;
        }
        
        .success {
            color: #155724;
            background-color: #d4edda;
            border: 1px solid #c3e6cb;
            padding: 10px;
            border-radius: 5px;
            margin-bottom: 20px;
        }
        
        .loading {
            text-align: center;
            color: #666;
            font-style: italic;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Guestbook</h1>
        
        <!-- Statistics Section -->
        <div class="stats">
            <h3>Guestbook Statistics</h3>
            <div id="stats">
                <div class="loading">Loading statistics...</div>
            </div>
        </div>
        
        <!-- Search Section -->
        <div class="search-box">
            <input type="text" id="searchInput" placeholder="Search messages...">
            <button onclick="searchMessages()">Search</button>
        </div>
        
        <!-- Form Section -->
        <div class="form-section">
            <h2>Leave a Message</h2>
            <form id="guestbookForm">
                <div class="form-group">
                    <label for="name">Your Name *</label>
                    <input type="text" id="name" name="name" required>
                </div>
                
                <div class="form-group">
                    <label for="email">Your Email *</label>
                    <input type="email" id="email" name="email" required>
                </div>
                
                <div class="form-group">
                    <label for="message">Your Message *</label>
                    <textarea id="message" name="message" rows="4" required 
                             placeholder="Share your thoughts..."></textarea>
                </div>
                
                <button type="submit">Post Message</button>
            </form>
        </div>
        
        <!-- Messages Section -->
        <div class="messages-section">
            <h2>Recent Messages</h2>
            <div id="messages">
                <div class="loading">Loading messages...</div>
            </div>
        </div>
    </div>

    <script>
        // Load statistics
        async function loadStats() {
            try {
                const response = await fetch('/api/stats');
                const data = await response.json();
                
                if (data.success) {
                    const stats = data.stats;
                    document.getElementById('stats').innerHTML = `
                        <p><strong>Total Messages:</strong> ${stats.totalMessages}</p>
                        <p><strong>Unique Contributors:</strong> ${stats.uniqueEmails}</p>
                        <p><strong>Messages Today:</strong> ${stats.messagesToday}</p>
                        <p><strong>First Message:</strong> ${new Date(stats.firstMessage).toLocaleDateString()}</p>
                    `;
                }
            } catch (error) {
                console.error('Error loading stats:', error);
            }
        }
        
        // Load messages
        async function loadMessages() {
            try {
                const response = await fetch('/api/messages');
                const data = await response.json();
                
                if (data.success) {
                    displayMessages(data.messages);
                } else {
                    document.getElementById('messages').innerHTML = 
                        '<div class="error">Error loading messages</div>';
                }
            } catch (error) {
                console.error('Error loading messages:', error);
                document.getElementById('messages').innerHTML = 
                    '<div class="error">Error loading messages</div>';
            }
        }
        
        // Display messages
        function displayMessages(messages) {
            const messagesDiv = document.getElementById('messages');
            
            if (messages.length === 0) {
                messagesDiv.innerHTML = '<p>No messages yet. Be the first to leave one!</p>';
                return;
            }
            
            messagesDiv.innerHTML = messages.map(msg => `
                <div class="message">
                    <div class="message-header">
                        <div class="message-name">${msg.name}</div>
                        <div class="message-date">${new Date(msg.createdAt).toLocaleString()}</div>
                    </div>
                    <div class="message-content">${msg.message}</div>
                    <div class="message-email">${msg.email}</div>
                </div>
            `).join('');
        }
        
        // Search messages
        async function searchMessages() {
            const query = document.getElementById('searchInput').value.trim();
            
            if (!query) {
                loadMessages();
                return;
            }
            
            try {
                const response = await fetch(`/api/messages/search/${encodeURIComponent(query)}`);
                const data = await response.json();
                
                if (data.success) {
                    displayMessages(data.messages);
                } else {
                    document.getElementById('messages').innerHTML = 
                        '<div class="error">Search failed</div>';
                }
            } catch (error) {
                console.error('Error searching messages:', error);
                document.getElementById('messages').innerHTML = 
                    '<div class="error">Search failed</div>';
            }
        }
        
        // Handle form submission
        document.getElementById('guestbookForm').addEventListener('submit', async function(e) {
            e.preventDefault();
            
            const formData = new FormData(this);
            const data = {
                name: formData.get('name'),
                email: formData.get('email'),
                message: formData.get('message')
            };
            
            try {
                const response = await fetch('/api/messages', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json'
                    },
                    body: JSON.stringify(data)
                });
                
                const result = await response.json();
                
                if (result.success) {
                    // Show success message
                    const successDiv = document.createElement('div');
                    successDiv.className = 'success';
                    successDiv.textContent = 'Message posted successfully!';
                    document.querySelector('.form-section').insertBefore(successDiv, document.querySelector('h2'));
                    
                    // Clear form
                    this.reset();
                    
                    // Reload messages and stats
                    loadMessages();
                    loadStats();
                    
                    // Remove success message after 3 seconds
                    setTimeout(() => {
                        successDiv.remove();
                    }, 3000);
                } else {
                    // Show error messages
                    const errorDiv = document.createElement('div');
                    errorDiv.className = 'error';
                    errorDiv.innerHTML = '<strong>Validation Errors:</strong><ul>' + 
                        result.errors.map(error => `<li>${error}</li>`).join('') + '</ul>';
                    document.querySelector('.form-section').insertBefore(errorDiv, document.querySelector('h2'));
                    
                    // Remove error message after 5 seconds
                    setTimeout(() => {
                        errorDiv.remove();
                    }, 5000);
                }
            } catch (error) {
                console.error('Error submitting form:', error);
                const errorDiv = document.createElement('div');
                errorDiv.className = 'error';
                errorDiv.textContent = 'Error submitting message. Please try again.';
                document.querySelector('.form-section').insertBefore(errorDiv, document.querySelector('h2'));
                
                setTimeout(() => {
                    errorDiv.remove();
                }, 5000);
            }
        });
        
        // Search on Enter key
        document.getElementById('searchInput').addEventListener('keypress', function(e) {
            if (e.key === 'Enter') {
                searchMessages();
            }
        });
        
        // Load initial data
        loadStats();
        loadMessages();
    </script>
</body>
</html>
```

## Advanced Filesystem Operations

### File Backup System

```javascript
class BackupManager {
    static createBackup() {
        const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
        const backupFile = path.join(DATA_DIR, `backup-${timestamp}.json`);
        
        try {
            const data = fs.readFileSync(MESSAGES_FILE, 'utf8');
            fs.writeFileSync(backupFile, data);
            console.log(`Backup created: ${backupFile}`);
            return backupFile;
        } catch (error) {
            console.error('Backup failed:', error);
            return null;
        }
    }
    
    static restoreFromBackup(backupFile) {
        try {
            const backupData = fs.readFileSync(backupFile, 'utf8');
            fs.writeFileSync(MESSAGES_FILE, backupData);
            console.log(`Restored from backup: ${backupFile}`);
            return true;
        } catch (error) {
            console.error('Restore failed:', error);
            return false;
        }
    }
    
    static listBackups() {
        try {
            const files = fs.readdirSync(DATA_DIR);
            return files.filter(file => file.startsWith('backup-') && file.endsWith('.json'));
        } catch (error) {
            console.error('Error listing backups:', error);
            return [];
        }
    }
}
```

### Data Export/Import

```javascript
class DataManager {
    static exportToCSV() {
        const messages = FileManager.getAllMessages();
        const csvHeader = 'ID,Name,Email,Message,Created At,IP\n';
        const csvData = messages.map(msg => 
            `${msg.id},"${msg.name}","${msg.email}","${msg.message.replace(/"/g, '""')}","${msg.createdAt}","${msg.ip}"`
        ).join('\n');
        
        return csvHeader + csvData;
    }
    
    static importFromCSV(csvData) {
        const lines = csvData.split('\n');
        const headers = lines[0].split(',');
        const messages = [];
        
        for (let i = 1; i < lines.length; i++) {
            if (lines[i].trim()) {
                const values = lines[i].split(',');
                messages.push({
                    name: values[1].replace(/"/g, ''),
                    email: values[2].replace(/"/g, ''),
                    message: values[3].replace(/"/g, ''),
                    createdAt: values[4].replace(/"/g, ''),
                    ip: values[5].replace(/"/g, '')
                });
            }
        }
        
        return messages;
    }
}
```

## Best Practices for Filesystem Operations

1. **Always handle errors** - File operations can fail
2. **Use proper file paths** - Use `path.join()` for cross-platform compatibility
3. **Validate file data** - Check data integrity when reading files
4. **Implement backups** - Regular backups prevent data loss
5. **Use appropriate file formats** - JSON for structured data, CSV for exports
6. **Consider file permissions** - Ensure proper read/write permissions
7. **Monitor disk space** - Large files can fill up disk space
8. **Use asynchronous operations** - For better performance with large files

## Testing Your Application

1. **Start the server:**
   ```bash
   node server.js
   ```

2. **Test the functionality:**
   - Visit `http://localhost:3000`
   - Add some messages
   - Search for messages
   - Check the `data/messages.json` file

3. **Test the API endpoints:**
   ```bash
   # Get all messages
   curl http://localhost:3000/api/messages
   
   # Add a message
   curl -X POST http://localhost:3000/api/messages \
     -H "Content-Type: application/json" \
     -d '{"name":"Test User","email":"test@example.com","message":"Hello World!"}'
   
   # Search messages
   curl http://localhost:3000/api/messages/search/hello
   
   # Get statistics
   curl http://localhost:3000/api/stats
   ```

## Next Steps

Congratulations! You've now learned how to:
- Handle GET and POST requests
- Create and process HTML forms
- Validate and sanitize user input
- Store and retrieve data from files
- Build a complete web application

You now have the foundation to build more complex web applications. Consider learning about:
- Databases (SQLite, MySQL, MongoDB)
- User authentication and sessions
- File uploads and handling
- RESTful API design
- Frontend frameworks (React, Vue, Angular)

**[Return to Request Handling Chapter →](index.md)**
