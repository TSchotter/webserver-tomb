---
layout: default
title: HTML Tags
nav_order: 2
parent: HTML & CSS
---

# HTML Tags

[&laquo; Return to HTML & CSS](index.md)

## Overview

HTML tags are the building blocks of web pages. Understanding how tags work and which ones to use is essential for creating well-structured, accessible web content.

## The Basic Webpage Template

Every HTML document follows a standard structure:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Page Title</title>
</head>
<body>
    <!-- Your content goes here -->
</body>
</html>
```

### Template Breakdown

**`<!DOCTYPE html>`** - Tells the browser this is an HTML5 document
**`<html>`** - Root element containing all other elements
**`<head>`** - Contains metadata not displayed on the page
**`<body>`** - Contains all visible content

> **Important for Webserver Developers**
>
> This template structure is what gets sent to the client. The `<head>` section contains information that helps the browser understand how to process the page, while the `<body>` contains what users actually see. 

> *Do I need `<!DOCTYPE html>`? It seems to work without it.*
>
> Most of the time, no. Most browsers will function without the doctype field. However, there are some extremely old browsers (and extremely old systems that run those browsers) where this is necessary. It's backwards compatibility for your webpage.

> *What are those meta tags for?*
>
> The `<meta charset="UTF-8">` tells the browser what character encoding to use (UTF-8 supports all characters including emojis and international text). The `<meta name="viewport"...>` tells mobile devices how to scale the page for different screen sizes.


## Understanding the Tag System

### Tag Structure

HTML tags follow a consistent pattern:

```html
<opening-tag>content</closing-tag>
```

**Opening Tag**: `<tagname>` - Defines the start of an element
**Content**: The text or other elements inside
**Closing Tag**: `</tagname>` - Defines the end of an element

### Self-Closing Tags

Some tags don't have content and are self-closing:

```html
<img src="image.jpg" alt="Description">
<br>
<hr>
<input type="text">
```

> For many of these self-closing tags, you'll see web developers add a forward slash before the closing bracket. 
> 
> `<br/>`  or `<img src="image.jpg" alt="Description"/>` for example
>
> This is done because in XHTML, these self-closing tags need to be marked as void elements (elements with nothing in them intentionally). 
>
> Since `<br/>` is accepted by both HTML and the more strict XHTML, it is often seen as preferable to use it. 

### Attributes

Tags can have attributes that provide additional information:

```html
<a href="https://example.com" target="_blank">Link Text</a>
```

> In the above examples, the `a` tag has two attributes:
> - `href`: which dictates where the link will take the client
> - `target`: which controls how the link opens (`_blank` opens in a new tab, `_self` opens in the same tab) 

> Attributes are often not required as part of the tag. Though many lose most of their functionality if some attributes are not included. An `a` tag doesn't function as a link unless it has an `href`.

## Essential Content Tags

### `<div>` - Generic Container

The `<div>` tag is a generic container used for grouping and styling content:

```html
<div class="header">
    <h1>Website Title</h1>
</div>

<div class="content">
    <p>Main content goes here</p>
</div>

<div class="footer">
    <p>Copyright information</p>
</div>
```

**Key Points:**
- Block-level element (takes full width)
- Used for layout and styling purposes
- No semantic meaning by itself
- Often combined with CSS classes for styling

> *What does it mean (takes full width)?*
> 
> Meaning, if you were to have two div tags like this:
> 
> ```html
> <div>Content1</div>
> <div>Content2</div>
> ```
> 
> Provided there isn't any other styles changing the rules of the `div` tag, these two contents will be on their own line.

> **Webserver Developer Note**
>
> `<div>` tags are commonly used in server-side templating systems to create consistent page layouts. When generating dynamic content, you'll often wrap sections in `<div>` elements with specific classes for styling.

#### Common `<div>` Attributes

The `<div>` tag commonly uses these attributes:

```html
<div class="header" id="main-header" style="background-color: blue;">
    <h1>Website Title</h1>
</div>
```

**Key Attributes:**
- `class` - CSS class name for styling (can be used multiple times on different elements)
- `id` - Unique identifier for the element (must be unique on the page)
- `style` - Inline CSS styling (not recommended for production, use CSS files instead)

> **Attribute Usage Guidelines**
>
> - Use `class` for styling that applies to multiple elements
> - Use `id` for unique elements that need to be targeted by JavaScript or CSS
> - Avoid `style` attribute in production code - use external CSS files instead

### `<p>` - Paragraph

The `<p>` tag defines paragraphs of text:

```html
<p>This is a paragraph of text. It will be displayed as a block element with default spacing.</p>

<p>This is another paragraph. Browsers automatically add spacing between paragraphs.</p>
```

**Key Points:**
- Block-level element
- Automatically adds spacing above and below
- Should contain text content, not other block elements
- Essential for readable content structure

### `<a>` - Links

The `<a>` tag creates hyperlinks to other pages or resources:

```html
<!-- External link -->
<a href="https://example.com">Visit Example</a>

<!-- Internal link -->
<a href="/about.html">About Us</a>

<!-- Link with target -->
<a href="https://example.com" target="_blank">Open in New Tab</a>

<!-- Link to email -->
<a href="mailto:contact@example.com">Contact Us</a>
```

**Key Attributes:**
- `href` - The destination URL (required)
- `target` - How to open the link (`_blank` for new tab)
- `title` - Tooltip text when hovering

> **Absolute URL vs relative URL**
>
> When linking to somewhere on the same site as the link, you can use relative pathing. For example:
>
> ```html
> <!-- Absolute URL - includes full domain -->
> <a href="https://example.com/about.html">About Us</a>
> 
> <!-- Relative URL - assumes same domain -->
> <a href="/about.html">About Us</a>
> <a href="about.html">About Us</a>
> <a href="../pages/contact.html">Contact</a>
> ```
>
> **Relative URL differences:**
> - `/about.html` - starts from the root directory (always goes to the same place regardless of current page)
> - `about.html` - looks for the file in the same directory as the current page
> - `../pages/contact.html` - goes up one directory level (`../`), then into the `pages` folder
> 



> **Security Consideration**
>
> When generating links dynamically, always validate and sanitize URLs to prevent security vulnerabilities like XSS attacks if the URL is from some user source (like if a user is posting a link on a comment post). Never trust user input for href attributes.
>
> **What is an XSS attack?** XSS (Cross-Site Scripting) occurs when malicious JavaScript code is injected into a webpage. For example, if a user enters `javascript:alert('hack')` as a URL, it could execute harmful code when clicked. Always validate URLs to ensure they're safe before using them in href attributes.

### `<script>` - JavaScript

The `<script>` tag embeds or references JavaScript code:

```html
<!-- Inline JavaScript -->
<script>
    alert('Hello, World!');
</script>

<!-- External JavaScript file -->
<script src="script.js"></script>

<!-- Script with type attribute -->
<script type="text/javascript" src="app.js"></script>
```

> Inline Javascript is placed on the HTML page that you want to run the JavaScript. Best used for one-off JavaScript that will not be used anywhere else.
>
> If you intend on some Javascript function to be run on several pages, it's best to have an external JavaScript file that multiple pages refer to.

**Key Points:**
- Can contain JavaScript code directly
- Can reference external JavaScript files
- Executes on the client side (browser)
- Can be placed in `<head>` or `<body>`

> **Critical Security Warning**
>
> JavaScript runs on the client side and is completely accessible to users. Never put sensitive logic, API keys, or server-side validation in JavaScript. Always validate data on the server, even if you also validate on the client.

## Additional Important Tags

### Text Formatting

```html
<h1>Main Heading</h1>
<h2>Subheading</h2>
<strong>Bold text</strong>
<em>Italic text</em>
<span>Inline text container</span>
```

### Lists

```html
<!-- Unordered list -->
<ul>
    <li>First item</li>
    <li>Second item</li>
</ul>

<!-- Ordered list -->
<ol>
    <li>First step</li>
    <li>Second step</li>
</ol>
```

### Images

```html
<img src="image.jpg" alt="Description of image" width="300" height="200">
```

## Best Practices for Webserver Developers

### 1. Semantic HTML
Use tags that describe content meaning:

```html
<!-- Good -->
<header>
    <nav>
        <a href="/">Home</a>
        <a href="/about">About</a>
    </nav>
</header>

<main>
    <article>
        <h1>Article Title</h1>
        <p>Article content...</p>
    </article>
</main>

<!-- Avoid -->
<div>
    <div>
        <a href="/">Home</a>
        <a href="/about">About</a>
    </div>
</div>
<div>
    <div>
        <div>Article Title</div>
        <div>Article content...</div>
    </div>
</div>
```

> **What is Semantic HTML?**
>
> Semantic HTML uses tags that describe the **meaning** of content, not just how it looks. While `<div>` and `<span>` are generic containers, semantic tags like `<header>`, `<nav>`, `<main>`, and `<article>` tell both humans and computers what the content represents.
>
> **Why does this matter beyond code readability?**
>
> - **Screen readers**: Assistive technology can navigate pages more effectively (e.g., "jump to main content")
> - **Search engines**: Google and other search engines better understand your page structure
> - **Browser features**: Some browsers can automatically generate table of contents from semantic structure
> - **Future-proofing**: Your code is more maintainable and accessible
> - **SEO benefits**: Search engines can better index and rank your content

### 2. Accessibility
- Always include `alt` attributes for images. The `alt` attribute should be a description of what the image is.
- Use proper heading hierarchy (h1 -> h2 -> h3)
- Ensure links have descriptive text

### 3. Performance
- Minimize the number of tags
- Use appropriate tags for content type
- Avoid unnecessary nesting

## Common HTML Mistakes

### 1. Missing Closing Tags

**Mistake:**
```html
<div>
    <p>This is a paragraph
    <p>This is another paragraph
</div>
```

**Correct:**
```html
<div>
    <p>This is a paragraph</p>
    <p>This is another paragraph</p>
</div>
```

> **Why this matters:** Just like it's easy to forget to add a semi-colon to the end of each sentence in Java or C, you'll find it's easy to forget to add the closing tag.
> 
> Highly recommend you immediately add the closing tag whenever you create a tag (most IDE's do this for you automatically).

### 2. Improper Nesting

**Mistake:**
```html
<p>This is a paragraph <strong>with bold text</p></strong>
```

**Correct:**
```html
<p>This is a paragraph <strong>with bold text</strong></p>
```

> **Why this matters:** Tags must be closed in the reverse order they were opened.

### 3. Using `<div>` for Everything

**Mistake:**
```html
<div>
    <div>Website Title</div>
    <div>
        <div>Navigation Link 1</div>
        <div>Navigation Link 2</div>
    </div>
</div>
```

**Correct:**
```html
<header>
    <h1>Website Title</h1>
    <nav>
        <a href="/">Navigation Link 1</a>
        <a href="/about">Navigation Link 2</a>
    </nav>
</header>
```

> **Why this matters:** Semantic HTML improves accessibility, SEO, and code maintainability.

### 4. Missing Alt Attributes on Images

**Mistake:**
```html
<img src="photo.jpg">
```

**Correct:**
```html
<img src="photo.jpg" alt="Description of the photo">
```

> **Why this matters:** Alt attributes are essential for screen readers and accessibility compliance.

### 5. Incorrect Heading Hierarchy

**Mistake:**
```html
<h1>Main Title</h1>
<h3>Subtitle</h3>
<h2>Another Title</h2>
```

**Correct:**
```html
<h1>Main Title</h1>
<h2>Subtitle</h2>
<h3>Another Title</h3>
```

> **Why this matters:** Proper heading hierarchy helps screen readers navigate and improves SEO.

### 6. Putting Block Elements Inside Inline Elements

**Mistake:**
```html
<a href="/page">
    <div>Click here</div>
    <p>More content</p>
</a>
```

**Correct:**
```html
<div>
    <a href="/page">Click here</a>
    <p>More content</p>
</div>
```

> **Why this matters:** HTML has rules about which elements can contain other elements. Block elements cannot be inside inline elements.

### 7. Forgetting to Quote Attribute Values

**Mistake:**
```html
<div class=container id=main-content>
```

**Correct:**
```html
<div class="container" id="main-content">
```

> **Why this matters:** Attributes values need to be a string. While some browsers are forgiving, unquoted attributes can cause parsing errors and are invalid HTML.

### 8. Using `<br>` for Spacing

**Mistake:**
```html
<p>First paragraph</p>
<br><br><br>
<p>Second paragraph</p>
```

**Correct:**
```html
<p>First paragraph</p>
<p>Second paragraph</p>
```

> **Why this matters:** Use CSS for spacing, not HTML. `<br>` should only be used for line breaks within content, not for layout spacing.

## Next Steps

Now that you understand HTML tags and how they function, learn about [CSS](CSSstart.md) to style your HTML content.

---