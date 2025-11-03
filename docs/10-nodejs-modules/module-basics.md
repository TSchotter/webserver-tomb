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

## Simple Example: Exporting a Single Function

Let's start with the simplest case - exporting a single function. Create a file called `greet.js`:

```javascript
// greet.js

function greet(name) {
  return `Hello, ${name}!`;
}

// Export the function directly
module.exports = greet;
```

Now create a file called `app.js` to use it:

```javascript
// app.js
const greet = require('./greet');

console.log(greet('World')); // Output: Hello, World!
```

When you export a single function like this, you can use it directly after requiring the module. Notice that `greet` is the function itself, not an object containing the function.

## Create Your First Module

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
module.exports = { add, multiply };
```

Here you're creating two functions, then exporting them. The export is important. You can't import them unless the file exports them.

## Use Your Module

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

## Exporting Individual Functions

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

Notice that, if you export multiple functions you'll need to call that specific function after the require.



## Exporting a Class

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

## Extending a Class

Classes can extend other classes using the `extends` keyword. This allows you to create a new class that inherits properties and methods from a parent class:

```javascript
// user.js - Base class
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

Now create a class that extends the User class:

```javascript
// admin.js
const User = require('./user');

class Admin extends User {
  constructor(name, email, permissions) {
    // Call the parent class constructor using super()
    super(name, email);
    this.permissions = permissions;
    this.role = 'admin';
  }
  
  // Add new methods specific to Admin
  deleteUser(userId) {
    return `User ${userId} deleted by ${this.name}`;
  }
  
  // Override the parent's greet method
  greet() {
    return `Hello, I'm ${this.name}, and I'm an administrator!`;
  }
}

module.exports = Admin;
```

Use it:

```javascript
// app.js
const User = require('./user');
const Admin = require('./admin');

// Create a regular user
const mario = new User('Mario', 'mario@example.com');
console.log(mario.greet());     // Output: Hello, I'm Mario!

// Create an admin user
const admin = new Admin('Luigi', 'luigi@example.com', ['delete', 'modify']);
console.log(admin.greet());     // Output: Hello, I'm Luigi, and I'm an administrator!
console.log(admin.getInfo());   // Output: { name: 'Luigi', email: 'luigi@example.com', ... }
console.log(admin.deleteUser(5)); // Output: User 5 deleted by Luigi
console.log(admin.role);        // Output: admin
```

### Key Points About Class Inheritance

1. **`extends` keyword**: Use `extends` to inherit from a parent class
2. **`super()`**: Call `super()` in the constructor to invoke the parent class constructor
3. **Method inheritance**: Child classes inherit all methods from the parent class
4. **Method overriding**: You can override parent methods by defining a method with the same name
5. **Additional properties**: Child classes can add their own properties and methods

### Multiple Levels of Inheritance

You can extend a class that itself extends another class:

```javascript
// user.js
class User {
  constructor(name, email) {
    this.name = name;
    this.email = email;
  }
  
  login() {
    return `${this.name} logged in`;
  }
}

module.exports = User;
```

```javascript
// admin.js
const User = require('./user');

class Admin extends User {
  constructor(name, email) {
    super(name, email);
    this.role = 'admin';
  }
  
  manageUsers() {
    return `${this.name} is managing users`;
  }
}

module.exports = Admin;
```

```javascript
// super-admin.js
const Admin = require('./admin');

class SuperAdmin extends Admin {
  constructor(name, email) {
    super(name, email);
    this.permissions = 'all';
  }
  
  deleteDatabase() {
    return `${this.name} deleted the database`;
  }
}

module.exports = SuperAdmin;
```

```javascript
// app.js
const SuperAdmin = require('./super-admin');

const boss = new SuperAdmin('Peach', 'peach@example.com');
console.log(boss.login());        // Inherited from User
console.log(boss.manageUsers());  // Inherited from Admin
console.log(boss.deleteDatabase()); // Defined in SuperAdmin
```

## Creating a Configuration Module

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

## Extending JavaScript Objects

When working with modules, you often need to extend or combine objects. JavaScript provides several ways to do this:

### Using the Spread Operator

The spread operator (`...`) is the modern way to extend objects by copying properties from one object into another:

```javascript
// base-config.js
const baseConfig = {
  port: 3000,
  host: 'localhost',
  timeout: 5000
};

module.exports = baseConfig;
```

```javascript
// app.js
const baseConfig = require('./base-config');

// Extend the base config with additional properties
const extendedConfig = {
  ...baseConfig,
  database: 'myapp',
  debug: true
};

// If you add a property with the same name, it will override the original
const overriddenConfig = {
  ...baseConfig,
  port: 8080  // This overrides the port from baseConfig
};

console.log(extendedConfig);
// Output: { port: 3000, host: 'localhost', timeout: 5000, database: 'myapp', debug: true }

console.log(overriddenConfig);
// Output: { port: 8080, host: 'localhost', timeout: 5000 }
```

Properties that appear later in the spread operation will override earlier ones. You can spread multiple objects:

```javascript
const config1 = { port: 3000, host: 'localhost' };
const config2 = { database: 'myapp' };
const config3 = { debug: true, port: 8080 }; // port will override config1's port

const combined = {
  ...config1,
  ...config2,
  ...config3
};

console.log(combined);
// Output: { port: 8080, host: 'localhost', database: 'myapp', debug: true }
```

### Real-World Example: Extending Configuration

This pattern is especially useful for configuration modules where you want to merge default settings with user-specific settings:

```javascript
// default-config.js
module.exports = {
  port: 3000,
  host: 'localhost',
  database: {
    host: 'localhost',
    name: 'myapp'
  }
};
```

```javascript
// user-config.js
const defaultConfig = require('./default-config');

// User can override specific settings
const userConfig = {
  ...defaultConfig,
  port: process.env.PORT || defaultConfig.port,
  database: {
    ...defaultConfig.database,
    name: process.env.DB_NAME || defaultConfig.database.name
  }
};

module.exports = userConfig;
```




## Real-World Example - Database Module

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

## Next Steps

---

**[Previous: Creating Your Own Node.js Modules](index.md)**

