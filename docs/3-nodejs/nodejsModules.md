---
layout: default
title: NodeJS Modules and NPM
nav_order: 3
parent: NodeJS
---

# NodeJS Modules and NPM

[&laquo; Return to NodeJS Chapter](index.md)

## Understanding Modules

NodeJS uses a module system to organize code into reusable pieces. This helps keep your applications organized and maintainable.

### Built-in Modules

NodeJS comes with many built-in modules that you can use without installing anything:

```javascript
// File system operations
const fs = require('fs');

// HTTP server functionality
const http = require('http');

// URL parsing
const url = require('url');

// Path manipulation
const path = require('path');
```

### Creating Your Own Modules

You can create your own modules by exporting functions, objects, or variables:

```javascript
// math.js
function add(a, b) {
    return a + b;
}

function subtract(a, b) {
    return a - b;
}

// Export functions
module.exports = {
    add: add,
    subtract: subtract
};

// Or export individual functions
module.exports.add = add;
module.exports.subtract = subtract;
```

### Using Your Modules

```javascript
// app.js
const math = require('./math');

console.log(math.add(5, 3));      // 8
console.log(math.subtract(10, 4)); // 6

// Or destructure the imports
const { add, subtract } = require('./math');
console.log(add(5, 3));
```

## NPM: Node Package Manager

NPM is the package manager for NodeJS. It allows you to install, manage, and share packages (libraries) with other developers.

### Package.json

Every NodeJS project should have a `package.json` file that describes your project:

```json
{
  "name": "my-webserver",
  "version": "1.0.0",
  "description": "A simple web server built with NodeJS",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.0"
  },
  "author": "Your Name",
  "license": "MIT"
}
```

Once you've installed packages, they will be listed in the "dependency" related sections. This allows your project to signal the npm which packages to install should someone else want to use your project (or you install it on a different machine).

To install packages for an npm project, it's as simple as running;

```bash
npm install
```
or
```bash
npm i
```


In the directory that has package.json

### Installing NEW Packages

```bash
# Install a package and save it to dependencies
npm install express

# Install a development dependency
npm install --save-dev nodemon

# Install a package globally
npm install -g nodemon
```

### Other Common NPM Commands

```bash
# List installed packages
npm list

# Update packages
npm update

# Remove packages
npm uninstall package-name
```
## Module Resolution

NodeJS looks for modules in this order:

1. **Built-in modules** (fs, http, etc.)
2. **Local files** (./math.js, ../utils/helper.js)
3. **node_modules directory** (installed packages)

```javascript
// These are all valid require statements
const fs = require('fs');                    // Built-in module
const math = require('./math');              // Local file (the './' means current directory)
const express = require('express');          // Installed package
const helper = require('../utils/helper');   // Relative path
```


## Best Practices

### Organize Your Code

```
project/
├── package.json
├── server.js
├── routes/
│   ├── index.js
│   └── api.js
├── models/
│   └── user.js
├── utils/
│   └── helpers.js
└── node_modules/
```

### Use Semantic Versioning

```json
{
  "dependencies": {
    "express": "^4.18.0",    // Compatible with 4.x.x
    "lodash": "~4.17.21",    // Compatible with 4.17.x
    "moment": "2.29.0"       // Exact version
  }
}
```

### Keep Dependencies Updated

```bash
# Check for outdated packages
npm outdated

# Update to latest versions
npm update
```

## Example: Building a Simple Module

Let's create a simple utility module for our web server with some functions:

```javascript
// utils/helpers.js
const fs = require('fs');

// Function to format a date nicely
function formatDate(date) {
    const year = date.getFullYear();
    const month = date.getMonth() + 1;
    const day = date.getDate();
    const hours = date.getHours();
    const minutes = date.getMinutes();
    const seconds = date.getSeconds();
    
    // This part might look strange to you, but in this language this is how you can have variables inside
    // of strings. Make sure you use the `  character for the string, located to the left of the '1'.
    return `${year}-${month}-${day} ${hours}:${minutes}:${seconds}`;
}

// Function to add two numbers
function addNumbers(a, b) {
    return a + b;
}

// Function to check if a file exists
function fileExists(filename) {
    try {
        fs.accessSync(filename);
        return true;
    } catch (error) {
        return false;
    }
}

// Function to get a random number between min and max
function getRandomNumber(min, max) {
    return Math.floor(Math.random() * (max - min + 1)) + min;
}

// Export all the functions
module.exports = {
    formatDate: formatDate,
    addNumbers: addNumbers,
    fileExists: fileExists,
    getRandomNumber: getRandomNumber
};
```

```javascript
// server.js
const helpers = require('./utils/helpers');

// Use the helper functions
const currentDate = helpers.formatDate(new Date());
console.log('Current date:', currentDate);

const sum = helpers.addNumbers(5, 3);
console.log('5 + 3 =', sum);

const hasFile = helpers.fileExists('package.json');
console.log('package.json exists:', hasFile);

const randomNum = helpers.getRandomNumber(1, 100);
console.log('Random number:', randomNum);
```

## Next Steps

Now that you understand modules and NPM, let's learn about Express.js - a popular web framework that makes building NodeJS applications much easier.

**[Next: Express.js →](express.md)**
