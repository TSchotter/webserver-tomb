---
layout: default
title: Connecting SQLite to Node.js Express
nav_order: 2
parent: Database Server Setup
---

# Connecting SQLite to Node.js Express

Now that you have SQLite set up, let's connect it to a Node.js application. We'll start by learning how to use SQLite with plain Node.js before integrating it with Express.

## Prerequisites

Before we begin, make sure you have:
- SQLite installed on your system (from the previous chapter)
- Node.js installed
- A SQLite database file (you can create one following the previous chapter)
- Basic understanding of JavaScript and Node.js modules

## Step 1: Install Required Packages

Install the `better-sqlite3` package for Node.js in your project:

```bash
npm install better-sqlite3
```

`better-sqlite3` is a fast, synchronous SQLite3 library that's perfect for most Node.js applications.

## Step 2: Connect to SQLite and Retrieve Data (Before Express)

Let's start by creating a simple Node.js script that connects to SQLite and retrieves data without using Express. This will help you understand how the database connection works.

Create a file called `test-database.js`:

```javascript
const Database = require('better-sqlite3');
const path = require('path');

// Connect to database file
const dbPath = path.join(__dirname, 'myapp.db');
const db = new Database(dbPath);

// Create tables if they don't exist
db.exec(`
  CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
  )
`);

// Insert some sample data if the table is empty
const userCount = db.prepare('SELECT COUNT(*) as count FROM users').get();
if (userCount.count === 0) {
  console.log('Inserting sample data...');
  // Alternative way to insert multiple pieces of data compared to what was done in class (more safe too)
  const insert = db.prepare('INSERT INTO users (name, email) VALUES (?, ?)');
  insert.run('Mario', 'mario@example.com');
  insert.run('Princess Peach', 'peach@example.com');
  insert.run('Luigi', 'luigi@example.com');
  console.log('Sample data inserted!');
}

// Retrieve and print all users
console.log('\n--- All Users ---');
const users = db.prepare('SELECT * FROM users').all();
console.log('Total users:', users.length);
users.forEach((user) => {
  console.log(`ID: ${user.id}, Name: ${user.name}, Email: ${user.email}, Created: ${user.created_at}`);
});

// Retrieve and print a specific user
console.log('\n--- User by ID ---');
const user = db.prepare('SELECT * FROM users WHERE id = ?').get(1);
if (user) {
  console.log(`Found user: ${user.name} (${user.email})`);
} else {
  console.log('User not found');
}

// Retrieve and print users by condition
console.log('\n--- Users matching condition ---');
// The "%" symbol acts like a wildcard
const filteredUsers = db.prepare('SELECT * FROM users WHERE name LIKE ?').all('%Mario%');
console.log(`Found ${filteredUsers.length} user(s) with "Mario" in their name:`);
filteredUsers.forEach((user) => {
  console.log(`     - ${user.name} (${user.email})`);
});

// Close the database connection
db.close();
console.log('\nDatabase connection closed.');
```

Run this script:

```bash
node test-database.js
```

You should see output like:

```
Inserting sample data...
Sample data inserted!

--- All Users ---
Total users: 3
ID: 1, Name: Mario, Email: mario@example.com, Created: 2025-11-03 11:57:49
ID: 2, Name: Princess Peach, Email: peach@example.com, Created: 2025-11-03 11:57:49
ID: 3, Name: Luigi, Email: luigi@example.com, Created: 2025-11-03 11:57:49

--- User by ID ---
Found user: Mario (mario@example.com)

--- Users matching condition ---
Found 1 user(s) with "Mario" in their name:
     - Mario (mario@example.com)

Database connection closed.
```

### Understanding the Code

- **`Database = require('better-sqlite3')`** - Imports the SQLite library
- **`new Database(dbPath)`** - Opens a connection to the database file and creates an object for the connection.
- **`db.exec()`** - Executes SQL statements that don't return data (like CREATE TABLE)
- **`db.prepare()`** - Prepares a SQL statement for execution
- **`.all()`** - Returns all matching rows as an array
- **`.get()`** - Returns a single row (or undefined if not found)
- **`.run()`** - Executes a statement and returns information about the operation
- **`db.close()`** - Closes the database connection

## Step 3: Create a Database Connection Module

Now let's create a reusable database connection module that we can use with Express. Create `database.js`:

```javascript
const Database = require('better-sqlite3');
const path = require('path');

// Connect to database file
const dbPath = path.join(__dirname, 'myapp.db');
const db = new Database(dbPath);

// Create tables if they don't exist
db.exec(`
  CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
  )
`);

module.exports = db;
```

## Step 4: Create Your Express App with SQLite Routes

Now let's create your Express application with routes that use the database. Here's an example `server.js`:

```javascript
const express = require('express');
const app = express();
const db = require('./database');

// Middleware
app.use(express.json()); // Parse JSON bodies

// GET all users
app.get('/users', (req, res) => {
  try {
    const users = db.prepare('SELECT * FROM users').all();
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// GET user by ID
app.get('/users/:id', (req, res) => {
  try {
    const user = db.prepare('SELECT * FROM users WHERE id = ?').get(req.params.id);
    if (user) {
      res.json(user);
    } else {
      res.status(404).json({ error: 'User not found' });
    }
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// POST create new user
app.post('/users', (req, res) => {
  try {
    const { name, email } = req.body;
    const stmt = db.prepare('INSERT INTO users (name, email) VALUES (?, ?)');
    const result = stmt.run(name, email);
    res.status(201).json({ 
      id: result.lastInsertRowid,
      name,
      email 
    });
  } catch (error) {
    if (error.message.includes('UNIQUE constraint')) {
      res.status(400).json({ error: 'Email already exists' });
    } else {
      res.status(500).json({ error: error.message });
    }
  }
});

// PUT update user
app.put('/users/:id', (req, res) => {
  try {
    const { name, email } = req.body;
    const stmt = db.prepare('UPDATE users SET name = ?, email = ? WHERE id = ?');
    const result = stmt.run(name, email, req.params.id);
    
    if (result.changes > 0) {
      res.json({ message: 'User updated successfully' });
    } else {
      res.status(404).json({ error: 'User not found' });
    }
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// DELETE user
app.delete('/users/:id', (req, res) => {
  try {
    const stmt = db.prepare('DELETE FROM users WHERE id = ?');
    const result = stmt.run(req.params.id);
    
    if (result.changes > 0) {
      res.json({ message: 'User deleted successfully' });
    } else {
      res.status(404).json({ error: 'User not found' });
    }
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Start server
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### Important Notes

1. **Database File Location**: Make sure your database file path is correct. Use `path.join(__dirname, 'myapp.db')` to create the database in your project directory.

2. **Error Handling**: Always handle database errors properly. SQLite will throw errors for constraint violations (like duplicate emails).

3. **Prepared Statements**: Always use prepared statements (with `?` placeholders) to prevent SQL injection. SQL injection is a serious security vulnerability where attackers can execute malicious SQL code.

   **Vulnerable Code (DON'T DO THIS):**
   ```javascript
   // This is vulnerable to SQL injection!
   const email = req.body.email;
   const user = db.prepare(`SELECT * FROM users WHERE email = '${email}'`).get();
   ```

   **Example SQL Injection Attack:**
   
   If an attacker submits `' OR '1'='1` as the email, the SQL query becomes:
   ```sql
   SELECT * FROM users WHERE email = '' OR '1'='1'
   ```
   
   This would return ALL users because `'1'='1'` is always true! An attacker could even do more damage:
   
   ```javascript
   // Attacker sends: "'; DROP TABLE users; --"
   // This would delete your entire users table!
   const maliciousInput = "'; DROP TABLE users; --";
   db.prepare(`SELECT * FROM users WHERE email = '${maliciousInput}'`).get();
   // Results in: SELECT * FROM users WHERE email = ''; DROP TABLE users; --'
   ```

   **Safe Code (USE THIS INSTEAD):**
   ```javascript
   // Good - uses prepared statement (safe from SQL injection)
   const email = req.body.email;
   const user = db.prepare('SELECT * FROM users WHERE email = ?').get(email);
   ```
   
   **How the `?` Placeholder Ensures Safety:**
   
   Prepared statements work in two separate phases:
   
   1. **Preparation Phase**: The database parses and validates the SQL structure (the query template with `?` placeholders). At this point, the SQL structure is fixed - it cannot be changed.
   
   2. **Execution Phase**: The actual data values are bound to the `?` placeholders. The database engine automatically:
      - Escapes special characters (like quotes, semicolons, etc.)
      - Treats the entire input as a single data value
      - Prevents the input from being interpreted as SQL code
   
   When you use `db.prepare('SELECT * FROM users WHERE email = ?').get(email)`, even if someone sends `'; DROP TABLE users; --`, the database will:
   - Treat the entire string `'; DROP TABLE users; --` as the email value to search for
   - Properly escape it so it's interpreted as literal text, not SQL commands
   - Search for a user with that exact email string (which probably doesn't exist)
   
   The key is **separation**: the SQL structure is determined once during preparation, and data is inserted later in a controlled, safe manner. This makes it impossible for user input to alter the SQL structure itself.

4. **Connection Closing**: With `better-sqlite3`, you can close the connection when shutting down:
   ```javascript
   process.on('SIGINT', () => {
     db.close();
     process.exit(0);
   });
   ```

5. **Production Considerations**: 
   - Ensure the database file has proper permissions
   - Set up regular backups (just copy the `.db` file)
   - Consider database migrations for schema changes

### Import/Export Data

```bash
# Export to SQL
sqlite3 myapp.db .dump > backup.sql

# Export to CSV
sqlite3 myapp.db .headers on .mode csv .output users.csv SELECT * FROM users

# Import from SQL file
sqlite3 myapp.db < backup.sql
```

## Next Steps

Now that you have SQLite connected to your Node.js Express application, you can:

- Create more complex database queries and relationships
- Implement user authentication with SQLite
- Set up database migrations for schema changes
- Create REST APIs that interact with your database
- Implement data validation and error handling

---

**[Previous: Setting Up SQLite Database](sqlite3-server-setup.md)** | **[Next: Database Server Setup](index.md)**

