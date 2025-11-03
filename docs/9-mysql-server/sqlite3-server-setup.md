---
layout: default
title: Setting Up SQLite Database
nav_order: 1
parent: Database Server Setup
---

# Setting Up SQLite Database

SQLite is a lightweight, file-based database that doesn't require a separate server process. It's perfect for development, small to medium applications, and learning database concepts. Unlike MySQL, SQLite stores everything in a single file, making it simple to set up and use.

## Introduction

SQLite provides several advantages:
- **No Server Required** - Database is stored in a single file
- **Zero Configuration** - Works immediately after installation
- **Portable** - Database file can be moved anywhere
- **Perfect for Learning** - Simple to understand and use
- **File-Based** - No user management or passwords needed
- **Works on the dang server** - Digital oceans smallest droplet seems to not have the resources nessessary for more complicated databases.

## Prerequisites

Before we begin, make sure you have:
- A Linux server (Ubuntu/Debian recommended for this guide)
- Root or sudo access to your server
- Basic familiarity with terminal commands
- Understanding of basic database concepts

## Step 1: Check for Existing MySQL Installation

If you previously installed MySQL (or it came pre-installed), it may cause conflicts. Let's check and remove it if needed:

```bash
# Check if MySQL is installed
dpkg -l | grep mysql-server

# If MySQL is installed, remove it completely
sudo apt remove --purge mysql-server mysql-client mysql-common mysql-server-core-* mysql-client-core-* -y
sudo apt autoremove -y
sudo apt autoclean
```

This will completely remove MySQL to avoid any conflicts. SQLite doesn't conflict with MySQL, but if MySQL has broken packages, it can prevent other installations.

## Step 2: Update System Packages

First, let's update your system's package list to ensure you're installing the latest available versions:

```bash
sudo apt update
```

You might get tired of doing this every chapter but it's a common practice to always `update` and `upgrade` before every major change.

## Step 3: Install SQLite

Install SQLite using the package manager:

```bash
sudo apt install sqlite3 -y
```

The `-y` flag automatically answers "yes" to any prompts during installation.

## Step 4: Verify SQLite Installation

Check that SQLite was installed correctly:

```bash
sqlite3 --version
```

You should see output showing your SQLite version, something like:
```
3.46.1 2024-08-13 09:16:08 c9c2ab54ba1f5f46360f1b4f35d849cd3f080e6fc2b6c60e91b16c63f69aalt1 (64-bit)
```

## Step 5: Create Your First Database

Unlike MySQL, SQLite doesn't require starting a service or configuring users. You simply create a database file! Let's create a database:

```bash
sqlite3 myapp.db
```

This command:
- Creates a new database file called `myapp.db` (if it doesn't exist)
- Opens SQLite's command-line interface

If the file already exists, it will open that database. You should see the SQLite prompt:

```
SQLite version 3.45.3
Enter ".help" for usage hints.
sqlite>
```

## Step 6: Create Your First Table

Now let's create a simple table to get started:

```sql
-- Create a users table
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

SQLite uses slightly different data types than MySQL:
- `INTEGER` instead of `INT`
- `TEXT` instead of `VARCHAR` for any text based input.
- `AUTOINCREMENT` instead of `AUTO_INCREMENT`
- `DATETIME` for timestamps

## Step 7: Insert and Query Data

Let's add some test data and query it:

```sql
-- Insert test data
INSERT INTO users (name, email) VALUES 
    ('Mario Luigi', 'mario@example.com'),
    ('Jane Smith', 'jane@example.com');

-- Query the data
SELECT * FROM users;

-- Exit SQLite
.quit
```

Note: In SQLite, you use `.quit` or `.exit` to leave the command-line interface, not `EXIT` like in MySQL.

## Step 8: Working with SQLite Databases

### Opening an Existing Database

To work with an existing database file:

```bash
sqlite3 myapp.db
```

### Listing Tables

Once inside SQLite, you can list all tables:

```sql
.tables
```

### Viewing Table Structure

To see the structure of a table:

```sql
.schema users
```

Or for all tables:

```sql
.schema
```

### Useful SQLite Dot Commands

SQLite has special commands that start with a dot (`.`) for database management:

```sql
-- List all tables
.tables

-- Show table structure
.schema table_name

-- Show all schemas
.schema

-- Show database information
.databases

-- Exit SQLite
.quit
-- or
.exit

-- Show help
.help

-- Set output mode (more readable)
.headers on
.mode column

-- Export data to CSV
.mode csv
.output data.csv
SELECT * FROM users;
```

## Step 9: Organizing Your Databases

Since SQLite databases are just files, you can organize them however you like. Here's a recommended structure:

```bash
# Create a directory for your databases
mkdir -p ~/databases

# Create databases in that directory
cd ~/databases
sqlite3 myapp.db

# Or specify full path
sqlite3 ~/databases/myapp.db
```

## SQLite vs MySQL: Key Differences

### Data Types

| MySQL | SQLite |
|-------|--------|
| INT, INT AUTO_INCREMENT | INTEGER PRIMARY KEY AUTOINCREMENT |
| VARCHAR(n) | TEXT |
| DATETIME | DATETIME or TEXT |
| BOOLEAN | INTEGER (0 or 1) |

### No User Management

SQLite doesn't have users, passwords, or permissions. The database file itself controls access (via file system permissions).

### No Server Process

SQLite doesn't run as a service. It's accessed directly through the file system.

## Common SQLite Commands Reference

### SQL Commands (Standard SQL)

```sql
-- Create a table
CREATE TABLE table_name (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    column_name TEXT
);

-- Insert data
INSERT INTO table_name (column_name) VALUES ('value');

-- Query data
SELECT * FROM table_name;
SELECT * FROM table_name WHERE column_name = 'value';

-- Update data
UPDATE table_name SET column_name = 'new_value' WHERE id = 1;

-- Delete data
DELETE FROM table_name WHERE id = 1;

-- Delete table
DROP TABLE table_name;
```

### SQLite-Specific Commands (Dot Commands)

```sql
-- Help
.help

-- List tables
.tables

-- Show schema
.schema
.schema table_name

-- Show databases (usually just main)
.databases

-- Exit
.quit
.exit

-- Format output
.headers on
.mode column
.mode csv
.mode json
```

## Working with SQLite from Command Line

You can also run SQL commands directly from the command line without entering interactive mode:

```bash
# Create table
sqlite3 myapp.db "CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT);"

# Insert data
sqlite3 myapp.db "INSERT INTO users (name) VALUES ('John Doe');"

# Query data
sqlite3 myapp.db "SELECT * FROM users;"

# Execute SQL from a file
sqlite3 myapp.db < script.sql
```

> ### That last command is important, it will allow you to create a setup file to initialize your database with tables.

Use of the ```<``` isn't something unique to sqlite or any program specifically. It's a linux commandline tool to take the text from a file and use it as input. You're effectively running SQL commands directly from the command line, but using a file to say what you would normally type.

The `>` operator works in the opposite direction - it redirects output from a command into a file. For example, `sqlite3 myapp.db .dump > backup.sql` takes all the output from the `.dump` command (which shows all your database contents as SQL commands) and saves it to a file called `backup.sql` instead of displaying it on the screen. This is called output redirection and is a fundamental Linux command-line feature. You can think of `<` as "read from file" and `>` as "write to file". 


## Best Practices

1. **Backup Regularly** - Just copy the `.db` file:
   ```bash
   cp myapp.db myapp_backup_$(date +%Y%m%d).db
   ```

2. **Use Descriptive Names** - Name your database files clearly:
   - `myapp.db` not `db1.db`
   - `users_db.db` not `data.db`

3. **Organize in Directories** - Keep databases organized:
   ```
   ~/databases/
     ├── production/
     │   └── app.db
     ├── development/
     │   └── app.db
     └── backups/
         └── app_20240115.db
   ```


## Next Steps

Now that you have SQLite set up and working, you can:

- Learn to connect Node.js applications to SQLite - see [Connecting SQLite to Node.js Express](sqlite3-nodejs-express.md)
- Create multiple databases for different projects
- Practice SQL queries and database design
- Learn about SQLite-specific features
- Set up automated backups (simple file copies!)

SQLite is perfect for learning and development. When you need more advanced features like user management, concurrent writes at scale, or network access, you can consider migrating to MySQL or PostgreSQL.

---

**[Previous: Database Server Setup](index.md)** | **[Next: Connecting SQLite to Node.js Express](sqlite3-nodejs-express.md)**
