---
layout: default
title: Module Basics
nav_order: 1
parent: Creating Your Own Node.js Modules
---

# Module Basics

In this lesson, you'll learn the fundamentals of creating your own Node.js modules. Modules are a way to organize code into reusable pieces that can be imported and used in other files.

## Why Create Your Own Modules?

Creating modules helps you:
- **Organize code** - Break large files into smaller, manageable pieces
- **Reuse code** - Write once, use many times
- **Maintain code** - Easier to find and fix bugs
- **Share code** - Use the same code across multiple projects
- **Test code** - Easier to test isolated functions

## Understanding module.exports

In Node.js, each file is treated as a separate module. To make functions, objects, or values available to other files, you use `module.exports` or `exports`.

## Step 1: Create Your First Module

Let's create a simple utility module. Create a file called `math-utils.js`:

```javascript
// math-utils.js

// Simple function to add two numbers
function add(a, b) {
  return a + b;
}

// Simple function to multiply two numbers
function multiply(a, b) {
  return a * b;
}

// Export the functions so other files can use them
module.exports = {
  add: add,
  multiply: multiply
};

// Alternative shorter syntax (ES6):
// module.exports = { add, multiply };
```

## Step 2: Use Your Module

Now create a file called `app.js` that uses your module:

```javascript
// app.js

// Import the module (note: no .js extension needed)
const mathUtils = require('./math-utils');

// Use the functions
const sum = mathUtils.add(5, 3);
const product = mathUtils.multiply(4, 7);

console.log(`Sum: ${sum}`);        // Output: Sum: 8
console.log(`Product: ${product}`); // Output: Product: 28
```

Run it:
```bash
node app.js
```

## Step 3: Exporting Individual Functions

You can also export functions individually:

```javascript
// string-utils.js

// Export each function directly
exports.capitalize = function(str) {
  return str.charAt(0).toUpperCase() + str.slice(1);
};

exports.reverse = function(str) {
  return str.split('').reverse().join('');
};

exports.trim = function(str) {
  return str.trim();
};
```

Then use them:

```javascript
// app.js
const strUtils = require('./string-utils');

console.log(strUtils.capitalize('hello'));  // Output: Hello
console.log(strUtils.reverse('world'));     // Output: dlrow
console.log(strUtils.trim('  space  '));     // Output: space
```

## Step 4: Exporting a Single Function

Sometimes you want to export just one main function:

```javascript
// greet.js

function greet(name) {
  return `Hello, ${name}!`;
}

// Export the function directly
module.exports = greet;
```

Use it:

```javascript
// app.js
const greet = require('./greet');

console.log(greet('Mario')); // Output: Hello, Mario!
```

## Step 5: Exporting a Class

You can export classes for object-oriented programming:

```javascript
// user.js

class User {
  constructor(name, email) {
    this.name = name;
    this.email = email;
    this.createdAt = new Date();
  }
  
  greet() {
    return `Hello, I'm ${this.name}!`;
  }
  
  getInfo() {
    return {
      name: this.name,
      email: this.email,
      createdAt: this.createdAt
    };
  }
}

module.exports = User;
```

Use it:

```javascript
// app.js
const User = require('./user');

const mario = new User('Mario', 'mario@example.com');
console.log(mario.greet());     // Output: Hello, I'm Mario!
console.log(mario.getInfo());   // Output: { name: 'Mario', email: 'mario@example.com', ... }
```

## Step 6: Creating a Configuration Module

Modules are great for configuration:

```javascript
// config.js

const config = {
  port: process.env.PORT || 3000,
  database: {
    host: process.env.DB_HOST || 'localhost',
    name: process.env.DB_NAME || 'myapp'
  },
  secret: process.env.SECRET || 'default-secret-change-me'
};

module.exports = config;
```

Use it:

```javascript
// app.js
const config = require('./config');

console.log(`Server will run on port ${config.port}`);
```

## Understanding module.exports vs exports

You can use either `module.exports` or `exports`, but there's a subtle difference:

```javascript
// These are equivalent:
module.exports.myFunction = function() { ... };
exports.myFunction = function() { ... };

// But this works:
module.exports = { myFunction: function() { ... } };

// While this does NOT work:
exports = { myFunction: function() { ... } }; // This won't work!
```

**Rule of thumb**: Use `module.exports` when assigning a new object. Use `exports` when adding properties to the exports object.

## Step 7: Real-World Example - Database Module

Let's create a practical module for database operations:

```javascript
// database.js
const Database = require('better-sqlite3');
const path = require('path');

// Create a module that exports a configured database connection
const dbPath = path.join(__dirname, 'myapp.db');
const db = new Database(dbPath);

// Create tables if they don't exist
db.exec(`
  CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL
  )
`);

// Export helper functions
function getAllUsers() {
  return db.prepare('SELECT * FROM users').all();
}

function getUserById(id) {
  return db.prepare('SELECT * FROM users WHERE id = ?').get(id);
}

function createUser(name, email) {
  const stmt = db.prepare('INSERT INTO users (name, email) VALUES (?, ?)');
  const result = stmt.run(name, email);
  return { id: result.lastInsertRowid, name, email };
}

// Export both the db connection and helper functions
module.exports = {
  db: db,
  getAllUsers: getAllUsers,
  getUserById: getUserById,
  createUser: createUser
};
```

Use it:

```javascript
// app.js
const { getAllUsers, createUser, getUserById } = require('./database');

// Create a user
const newUser = createUser('Luigi', 'luigi@example.com');
console.log('Created user:', newUser);

// Get all users
const users = getAllUsers();
console.log('All users:', users);

// Get user by ID
const user = getUserById(1);
console.log('User by ID:', user);
```

## Best Practices

1. **One responsibility per module** - Each module should do one thing well
2. **Clear naming** - Name modules based on what they do (`math-utils.js`, `database.js`)
3. **Export what's needed** - Only export what other files need
4. **Document your modules** - Add comments explaining what functions do
5. **Organize by functionality** - Group related functions together

## Common Patterns

### Pattern 1: Utility Functions
```javascript
// utils.js
module.exports = {
  function1: function() { ... },
  function2: function() { ... }
};
```

### Pattern 2: Configuration Object
```javascript
// config.js
module.exports = {
  setting1: 'value1',
  setting2: 'value2'
};
```

### Pattern 3: Single Function
```javascript
// validator.js
module.exports = function(value) {
  // validation logic
};
```

### Pattern 4: Class
```javascript
// model.js
module.exports = class Model {
  constructor() { ... }
  method() { ... }
};
```

## Practice Exercise

Create a module called `calculator.js` with the following functions:
- `add(a, b)` - Add two numbers
- `subtract(a, b)` - Subtract two numbers
- `divide(a, b)` - Divide two numbers (handle division by zero)
- `multiply(a, b)` - Multiply two numbers

Then create an `app.js` file that uses your calculator module to perform calculations.

## Next Steps

Now that you understand module basics, you're ready to learn about advanced module patterns in the next lesson.

---

**[Previous: Creating Your Own Node.js Modules](index.md)** | **[Next: Advanced Module Patterns](advanced-modules.md)**

