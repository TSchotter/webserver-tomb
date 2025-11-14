---
layout: default
title: Password Requirements and Encryption
nav_order: 1
parent: Complex Database Integration
---

# Password Requirements and Encryption

In this subchapter, we'll learn how to implement password validation with minimum requirements and encrypt passwords using argon2. This is the foundation of secure authentication - before we can store passwords in a database, we need to ensure they meet security standards and are properly encrypted.

## What You'll Learn

- Installing and using argon2 for password encryption
- Creating password validation with minimum requirements
- Understanding one-way hashing and why it's secure
- Building a reusable password utilities module

## Step 1: Install Required Dependencies

Create a new project directory and install the necessary packages:

```bash
mkdir auth-system
cd auth-system
npm init -y
npm install argon2
```

**Package explanation:**
- **argon2**: Modern password hashing library that won the Password Hashing Competition in 2015. It's considered the current best practice for password hashing, offering better security than bcrypt while being resistant to GPU-based attacks. It uses a one-way hashing algorithm that cannot be reversed, making it perfect for storing passwords securely.

## Step 2: Create Password Utilities Module

Create `modules/password-utils.js` to handle password hashing and validation:

```javascript
// modules/password-utils.js
const argon2 = require('argon2');

// Argon2 configuration options
// These values provide a good balance of security and performance
const ARGON2_OPTIONS = {
  type: argon2.argon2id,  // Uses a hybrid approach (best for most cases)
  memoryCost: 65536,      // 64 MB memory cost
  timeCost: 3,            // Number of iterations
  parallelism: 4          // Number of parallel threads
};

/*
validatePassword takes a password and checks to see if
it passes some standard requirements (like length, an uppercase, etc)
*/
function validatePassword(password) {
  const errors = [];
  
  if (!password) {
    errors.push('Password is required');
    return { valid: false, errors };
  }
  
  if (password.length < 8) {
    errors.push('Password must be at least 8 characters long');
  }
  
  if (!/[A-Z]/.test(password)) {
    errors.push('Password must contain at least one uppercase letter');
  }
  
  if (!/[a-z]/.test(password)) {
    errors.push('Password must contain at least one lowercase letter');
  }
  
  if (!/[0-9]/.test(password)) {
    errors.push('Password must contain at least one number');
  }
  
  if (!/[!@#$%^&*(),.?":{}|<>]/.test(password)) {
    errors.push('Password must contain at least one special character');
  }
  
  return {
    valid: errors.length === 0,
    errors: errors
  };
}

// simple function that will hash a password.
async function hashPassword(password) {
  return await argon2.hash(password, ARGON2_OPTIONS);
}


// Compares a plain text password with a hashed password

async function comparePassword(password, hash) {
  return await argon2.verify(hash, password);
}

module.exports = {
  validatePassword,
  hashPassword,
  comparePassword
};
```

## Understanding Password Validation

The `validatePassword` function checks that passwords meet minimum security requirements:

1. **Minimum length**: At least 8 characters (longer passwords are harder to crack)
2. **Uppercase letter**: At least one capital letter (A-Z)
3. **Lowercase letter**: At least one lowercase letter (a-z)
4. **Number**: At least one digit (0-9)
5. **Special character**: At least one special character (!@#$%^&*(),.?":{}|<>)

**Why these requirements?**
- **Length**: Longer passwords take exponentially more time to brute-force
- **Mixed case**: Increases the possible character combinations
- **Numbers and special characters**: Further increases complexity, making dictionary attacks less effective

## Understanding argon2 Password Encryption

### What is argon2?

argon2 is a modern password hashing function that won the Password Hashing Competition in 2015. It's designed to be memory-hard, making it resistant to both CPU and GPU-based attacks. It's considered the current best practice for password hashing, recommended by OWASP and security experts.

### Key Concepts

**One-way hashing:**
- Passwords are **hashed**, not encrypted
- A hash cannot be reversed to get the original password
- Even if someone accesses your database, they cannot retrieve the original passwords

**Salt:**
- Each password gets a unique random "salt" before hashing. You don't need to do this yourself, argon2 does this automatically.
- This means identical passwords produce different hashes
- Prevents rainbow table attacks (pre-computed hash tables)

**Memory-hard design:**
- argon2 requires significant memory to compute hashes
- This makes it resistant to GPU-based attacks (GPUs have limited memory)
- The memory cost parameter controls how much memory is used

**Time cost and parallelism:**
- **timeCost**: Number of iterations (higher = more secure but slower)
- **parallelism**: Number of parallel threads (can utilize multiple CPU cores)
- **type**: argon2id (recommended) uses a hybrid approach combining resistance to both side-channel and GPU attacks

**How it works:**
1. When you hash a password, argon2 generates a random salt
2. The salt is combined with the password
3. The combination is processed through multiple iterations using memory-hard operations
4. The result is a hash that includes the salt and the hashed password

### Why is argon2.hash() Async?

argon2 is **slow by design** - With the default configuration, hashing a password takes approximately 100-300 milliseconds. This might not seem like much, but in a web server handling many requests, it adds up.

**The Problem with Synchronous Operations:**

Node.js uses a **single-threaded event loop**. This means it can only do one thing at a time. If you use a synchronous (blocking) operation, the entire server stops and waits for that operation to complete. During that time, no other requests can be processed.

### What Does the `async` Keyword Actually Do?

**Important:** The `async` keyword does **NOT** create a new thread! This is a common misconception.

**What `async` actually does:**

1. **Makes a function return a Promise**: An `async` function always returns a Promise, even if you don't explicitly return one
2. **Allows `await` inside**: You can only use `await` inside `async` functions
3. **Pauses execution, doesn't block**: When you `await`, the function pauses and yields control back to the event loop

**How it works (simplified):**

```javascript
// Without async - blocks everything
function hashPasswordSync(password) {
  // This runs on the main thread
  // Nothing else can run while this executes
  // Note: argon2 doesn't have a sync version, this is just for illustration
  return argon2.hashSync(password); // Would take ~200ms, blocks everything
}

// With async - doesn't block
async function hashPasswordAsync(password) {
  // This also runs on the main thread
  // But when we hit await, we pause and let other code run
  return await argon2.hash(password, ARGON2_OPTIONS); // Takes ~200ms, but doesn't block
}
```

**The Event Loop (Simplified Explanation):**

Node.js has a single main thread with an "event loop" that manages tasks:

```
Main Thread (Event Loop)
├── Request 1: hashPassword() → await → [paused, waiting]
├── Request 2: hashPassword() → await → [paused, waiting]  
├── Request 3: simple route handler → [executing now]
└── When argon2 finishes → resume Request 1 or 2
```

**Key Points:**

- **No threads created**: Everything runs on the same main thread
- **Non-blocking**: When `await` pauses, other code can run
- **Still single-threaded**: Only one piece of JavaScript runs at a time
- **I/O operations**: argon2 uses native code (C) that can run in parallel, but JavaScript execution is still single-threaded

**Why This Matters:**

```javascript
// BAD: Would block the entire server (argon2 doesn't have sync, but if it did)
app.post('/register', (req, res) => {
  const hash = argon2.hashSync(password); // Would block for 200ms
  // During those 200ms, NO other requests can be processed
});

// GOOD: Yields control back to event loop
app.post('/register', async (req, res) => {
  const hash = await argon2.hash(password, ARGON2_OPTIONS); // Pauses for 200ms
  // During those 200ms, OTHER requests CAN be processed
});
```

**The Magic:**

When you `await`, Node.js:
1. Pauses your function
2. Returns control to the event loop
3. Processes other requests/tasks
4. When argon2 finishes (in native code), it resumes your function

This is why async is powerful - it allows concurrency without threads!

### Alternative: Using `.then()` Callbacks

While `async/await` is the modern and recommended way to handle asynchronous operations, you can also use `.then()` callbacks. Both approaches work with Promises, but they have different syntax.

**Understanding Promises:**

When an async function is called, it returns a Promise. A Promise represents a value that will be available in the future. You can handle this value in two ways:

1. **Using `await`** (what we've been doing)
2. **Using `.then()`** (the older callback style)

**Using `.then()` instead of `await`:**

```javascript
// Using async/await (modern approach)
async function hashPasswordWithAwait(password) {
  const hash = await argon2.hash(password, ARGON2_OPTIONS);
  console.log('Hash:', hash);
  return hash;
}

// Using .then() (alternative approach)
function hashPasswordWithThen(password) {
  return argon2.hash(password, ARGON2_OPTIONS)
    .then(hash => {
      console.log('Hash:', hash);
      return hash;
    });
}
```

**Chaining Multiple Operations:**

Both approaches can handle multiple async operations:

```javascript
// Using async/await
async function registerUser(username, password) {
  const hash = await hashPassword(password);
  const user = await saveUserToDatabase(username, hash);
  return user;
}

// Using .then() - same functionality, different syntax
function registerUserWithThen(username, password) {
  return hashPassword(password)
    .then(hash => saveUserToDatabase(username, hash));
}
```

**Error Handling:**

Both approaches handle errors differently:

```javascript
// Using async/await with try-catch
async function hashPasswordSafely(password) {
  try {
    const hash = await argon2.hash(password, ARGON2_OPTIONS);
    return hash;
  } catch (error) {
    console.error('Hashing failed:', error);
    throw error;
  }
}

// Using .then() with .catch()
function hashPasswordSafelyWithThen(password) {
  return argon2.hash(password, ARGON2_OPTIONS)
    .then(hash => hash)
    .catch(error => {
      console.error('Hashing failed:', error);
      throw error;
    });
}
```

**In Express Route Handlers:**

```javascript
// Using async/await (recommended)
app.post('/register', async (req, res) => {
  try {
    const hash = await hashPassword(req.body.password);
    await saveUser(req.body.username, hash);
    res.send('User registered');
  } catch (error) {
    res.status(500).send('Registration failed');
  }
});

// Using .then() (alternative)
app.post('/register', (req, res) => {
  hashPassword(req.body.password)
    .then(hash => saveUser(req.body.username, hash))
    .then(() => res.send('User registered'))
    .catch(error => {
      res.status(500).send('Registration failed');
    });
});
```

**When to Use Which?**

- **Use `async/await`**: 
  - More readable, especially with multiple async operations
  - Easier error handling with try-catch
  - Modern JavaScript standard
  - Recommended for new code

- **Use `.then()`**:
  - When working with older codebases
  - When you prefer functional programming style
  - When chaining simple operations
  - Both are equivalent - choose based on preference and team standards

**Important:** Both approaches are non-blocking and work the same way under the hood. The choice is primarily about code style and readability.

### Example: How Hashing Works

```javascript
const { hashPassword, comparePassword } = require('./modules/password-utils');

// Hash a password
const password = 'MySecure123!';
const hash = await hashPassword(password);
console.log('Hash:', hash);
// Output: $argon2id$v=19$m=65536,t=3,p=4$salt$hashedpassword
// format: $ algorithm $ version $ memoryCost,timeCost,parallelism $ salt $ hash        

// Compare passwords
const isMatch = await comparePassword('MySecure123!', hash);
console.log('Password matches:', isMatch); // true

const isWrong = await comparePassword('WrongPassword', hash);
console.log('Wrong password matches:', isWrong); // false
```

**Important:** Notice that even if you hash the same password twice, you'll get different hashes because of the random salt. But `comparePassword` will still correctly identify that they match!

## Step 3: Testing Password Utilities

Create a test file to verify your password utilities work correctly:

```javascript
// test-password-utils.js
const { validatePassword, hashPassword, comparePassword } = require('./modules/password-utils');

async function testPasswordUtils() {
  console.log('=== Testing Password Validation ===\n');
  
  // Test valid password
  const validResult = validatePassword('MySecure123!');
  console.log('Valid password test:', validResult.valid); // true
  console.log('Errors:', validResult.errors); // []
  
  // Test invalid passwords
  const weakPasswords = [
    'short',           // Too short
    'nouppercase123!', // No uppercase
    'NOLOWERCASE123!', // No lowercase
    'NoNumbersHere!',  // No numbers
    'NoSpecialChars123' // No special characters
  ];
  
  weakPasswords.forEach(pwd => {
    const result = validatePassword(pwd);
    console.log(`\nPassword: "${pwd}"`);
    console.log('Valid:', result.valid);
    console.log('Errors:', result.errors);
  });
  
  console.log('\n=== Testing Password Hashing ===\n');
  
  // Test hashing
  const password = 'TestPassword123!';
  const hash1 = await hashPassword(password);
  const hash2 = await hashPassword(password);
  
  console.log('Original password:', password);
  console.log('Hash 1:', hash1);
  console.log('Hash 2:', hash2);
  console.log('Hashes are different (due to salt):', hash1 !== hash2);
  
  // Test comparison
  console.log('\n=== Testing Password Comparison ===\n');
  
  const correctMatch = await comparePassword(password, hash1);
  const wrongMatch = await comparePassword('WrongPassword', hash1);
  
  console.log('Correct password matches:', correctMatch); // true
  console.log('Wrong password matches:', wrongMatch); // false
}

testPasswordUtils().catch(console.error);
```

Run the test:

```bash
node test-password-utils.js
```

## Understanding Security: Why Not Store Plain Text?

**Never store passwords in plain text!** Here's why:

1. **Database breaches**: If your database is compromised, attackers get all passwords
2. **Insider threats**: Anyone with database access can see all passwords
3. **Reuse attacks**: Many users reuse passwords across sites - exposing one exposes many

**With argon2:**
- Even if the database is breached, attackers only get hashes
- Hashes cannot be reversed to get original passwords
- Brute-forcing argon2 hashes is extremely slow (by design)
- Memory-hard design makes GPU-based attacks impractical
- Each password has a unique salt, preventing rainbow table attacks

## Best Practices

1. **Always validate before hashing**: Check password requirements before storing
2. **Use async/await**: argon2 functions are asynchronous
3. **Never log passwords**: Even in development, never log plain text passwords
4. **Choose appropriate parameters**: The default configuration is good for most apps, increase memoryCost and timeCost for high-security applications
5. **Handle errors**: Always wrap argon2 operations in try-catch blocks
6. **Use argon2id**: The hybrid type (argon2id) is recommended as it provides the best balance of security against different attack vectors

---

**[Previous: Complex Database Integration](index.md)** | **[Next: Putting the Server on the Map](../11-putting-server-on-map/index.md)**

