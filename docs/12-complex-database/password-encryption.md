---
layout: default
title: Password Requirements and Encryption
nav_order: 1
parent: Complex Database Integration
---

# Password Requirements and Encryption

In this subchapter, we'll learn how to implement password validation with minimum requirements and encrypt passwords using bcrypt. This is the foundation of secure authentication - before we can store passwords in a database, we need to ensure they meet security standards and are properly encrypted.

## What You'll Learn

- Installing and using bcrypt for password encryption
- Creating password validation with minimum requirements
- Understanding one-way hashing and why it's secure
- Building a reusable password utilities module

## Step 1: Install Required Dependencies

Create a new project directory and install the necessary packages:

```bash
mkdir auth-system
cd auth-system
npm init -y
npm install bcrypt
```

**Package explanation:**
- **bcrypt**: Industry-standard password hashing library. It uses a one-way hashing algorithm that cannot be reversed, making it perfect for storing passwords securely.

## Step 2: Create Password Utilities Module

Create `modules/password-utils.js` to handle password hashing and validation:

```javascript
// modules/password-utils.js
const bcrypt = require('bcrypt');

// Number of salt rounds (higher = more secure but slower)
// 10 is a good balance for most applications
const SALT_ROUNDS = 10;

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
  return await bcrypt.hash(password, SALT_ROUNDS);
}


// Compares a plain text password with a hashed password

async function comparePassword(password, hash) {
  return await bcrypt.compare(password, hash);
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

## Understanding bcrypt Password Encryption

### What is bcrypt?

bcrypt is a password hashing function designed by Niels Provos and David Mazières. It's specifically designed to be slow, making brute-force attacks impractical.

### Key Concepts

**One-way hashing:**
- Passwords are **hashed**, not encrypted
- A hash cannot be reversed to get the original password
- Even if someone accesses your database, they cannot retrieve the original passwords

**Salt:**
- Each password gets a unique random "salt" before hashing. You don't need to do this yourself, bcrypt does this.
- This means identical passwords produce different hashes
- Prevents rainbow table attacks (pre-computed hash tables)

**Salt rounds:**
- The number of times bcrypt hashes the password
- Higher rounds = more secure but slower
- 10 rounds is a good balance (takes ~100ms to hash)

**How it works:**
1. When you hash a password, bcrypt generates a random salt
2. The salt is combined with the password
3. The combination is hashed multiple times (salt rounds)
4. The result is a hash that includes the salt and the hashed password

### Why is bcrypt.hash() Async?

bcrypt is **slow** - With 10 salt rounds, hashing a password takes approximately 100-200 milliseconds. This might not seem like much, but in a web server handling many requests, it adds up.

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
  return bcrypt.hashSync(password, 10); // Takes ~100ms, blocks everything
}

// With async - doesn't block
async function hashPasswordAsync(password) {
  // This also runs on the main thread
  // But when we hit await, we pause and let other code run
  return await bcrypt.hash(password, 10); // Takes ~100ms, but doesn't block
}
```

**The Event Loop (Simplified Explanation):**

Node.js has a single main thread with an "event loop" that manages tasks:

```
Main Thread (Event Loop)
├── Request 1: hashPassword() → await → [paused, waiting]
├── Request 2: hashPassword() → await → [paused, waiting]  
├── Request 3: simple route handler → [executing now]
└── When bcrypt finishes → resume Request 1 or 2
```

**Key Points:**

- **No threads created**: Everything runs on the same main thread
- **Non-blocking**: When `await` pauses, other code can run
- **Still single-threaded**: Only one piece of JavaScript runs at a time
- **I/O operations**: bcrypt uses native code (C++) that can run in parallel, but JavaScript execution is still single-threaded

**Why This Matters:**

```javascript
// BAD: Blocks the entire server
app.post('/register', (req, res) => {
  const hash = bcrypt.hashSync(password, 10); // Blocks for 100ms
  // During those 100ms, NO other requests can be processed
});

// GOOD: Yields control back to event loop
app.post('/register', async (req, res) => {
  const hash = await bcrypt.hash(password, 10); // Pauses for 100ms
  // During those 100ms, OTHER requests CAN be processed
});
```

**The Magic:**

When you `await`, Node.js:
1. Pauses your function
2. Returns control to the event loop
3. Processes other requests/tasks
4. When bcrypt finishes (in native code), it resumes your function

This is why async is powerful - it allows concurrency without threads!

### Example: How Hashing Works

```javascript
const { hashPassword, comparePassword } = require('./modules/password-utils');

// Hash a password
const password = 'MySecure123!';
const hash = await hashPassword(password);
console.log('Hash:', hash);
// Output: $2b$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
// format: $ algorithm version $ salt rounds $ password + salt        

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

**With bcrypt:**
- Even if the database is breached, attackers only get hashes
- Hashes cannot be reversed to get original passwords
- Brute-forcing bcrypt hashes is extremely slow (by design)
- Each password has a unique salt, preventing rainbow table attacks

## Best Practices

1. **Always validate before hashing**: Check password requirements before storing
2. **Use async/await**: bcrypt functions are asynchronous
3. **Never log passwords**: Even in development, never log plain text passwords
4. **Choose appropriate salt rounds**: 10 is good for most apps, increase for high-security
5. **Handle errors**: Always wrap bcrypt operations in try-catch blocks

---

**[Previous: Complex Database Integration](index.md)** | **[Next: Putting the Server on the Map](../11-putting-server-on-map/index.md)**

