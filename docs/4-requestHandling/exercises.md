---
layout: default
title: Exercises
nav_order: 10
parent: Request Handling
---

# In-Class Exercises

[&laquo; Return to Request Handling Chapter](index.md)

## Overview

This experiemental chapter will have exercises that are attempted in-class to give you (the students) an opporuntity to work on a mini-assignment covering content from the course (and to make it clear that this web-textbook exists for help material).



## Available Exercises

### Exercise 1: Handlebars Header with Navigation

**Objective:** Practice creating a reusable header partial with dynamic navigation and page titles using Handlebars

**Requirements:**
- Install and configure the `hbs` package
- Create a header partial that includes:
  - A navigation bar with multiple links
  - Dynamic page title that changes based on the current page
  - A welcome message that can be customized
- Set up multiple routes that use the same header
- Pass different data to each route to demonstrate dynamic content

**Starter Code:**
```bash
npm init
npm install express
npm install hbs
```

```javascript
const express = require('express');
const hbs = require('hbs');
const path = require('path');

const app = express();
const PORT = 3000;

// Configure Handlebars
app.set('view engine', 'hbs');
app.set('views', path.join(__dirname, 'views'));

// Register partials directory
hbs.registerPartials(path.join(__dirname, 'views', 'partials'));

// Middleware
app.use(express.urlencoded({ extended: true }));
app.use(express.json());
app.use(express.static('public'));

// Your routes here

app.listen(PORT, () => {
    console.log(`Server running on ${PORT}`);
});
```

**Required File Structure:**
```
your-project/
├── views/
│   ├── partials/
│   │   └── header.hbs
│   ├── home.hbs
│   ├── about.hbs
│   └── contact.hbs
├── public/
└── server.js
```

**Expected Output:**
- Route: `/` should show home page with "Home" in navigation and "Welcome to Our Site" as title
- Route: `/about` should show about page with "About" highlighted in navigation and "About Us" as title  
- Route: `/contact` should show contact page with "Contact" highlighted in navigation and "Contact Us" as title
- All pages should share the same header with consistent navigation

**Header Partial Requirements:**
The header should include:
- Navigation links for Home, About, and Contact
- Dynamic page title that changes based on the current page
- A welcome message that can be customized per page
- Use Handlebars conditionals to highlight the current page in navigation

**Bonus Challenges:**
- Add a custom Handlebars helper to format the current date in the header
- Create a footer partial and include it on all pages




**[Return to Request Handling Chapter →](index.md)**

