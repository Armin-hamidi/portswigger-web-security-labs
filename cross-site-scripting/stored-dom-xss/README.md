# Stored DOM XSS

## Lab Description

**Difficulty:** Practitioner

This lab demonstrates a stored DOM-based Cross-Site Scripting (XSS) vulnerability in the blog comment functionality.

The application stores user-submitted comments and retrieves them using client-side JavaScript. Before displaying the comments, the JavaScript attempts to encode angle brackets using an `escapeHTML()` function.

However, the function is implemented incorrectly. It uses JavaScript's `String.replace()` method with string arguments, which means it replaces only the first occurrence of each character.

Because the application later inserts the processed comment into the page using `innerHTML`, any remaining HTML markup can be interpreted by the browser.

The goal of this lab was to investigate the comment functionality, identify the incomplete HTML encoding, and exploit the vulnerability to execute `alert(1)`.

## Solution

### 1. Testing the Comment Functionality

<p>
<img src="images/screenshot 1.png" alt="Screenshot 1 - Submitting a normal blog comment">
</p>

I started by investigating the blog's comment section because the lab description mentioned a vulnerability in the comment functionality.

I submitted a normal comment using the following values:

- **Comment:** `testcomment`
- **Name:** `testname`
- **Email:** `test@gmail.com`
- **Website:** `https://test.com`

The application accepted the comment and displayed it on the blog post.

I also noticed that the website field required a valid URL beginning with `http:` or `https:`. This showed that the application performed some input validation.

However, validation does not necessarily prevent XSS. I needed to understand how the application stored, retrieved, and processed the submitted data.

My first step was to establish normal behavior before testing for a vulnerability.

### 2. Inspecting the Rendered Comment in the DOM

<p>
<img src="images/screenshot 2.png" alt="Screenshot 2 - Inspecting the rendered comment and its HTML elements">
</p>

After submitting the comment, I opened the browser's developer tools and inspected the live DOM.

I found that the application rendered the submitted information in different places:

- The author's name appeared in the comment header.
- The website was assigned to the `href` attribute of an anchor element.
- The comment body appeared inside a paragraph element.

The rendered structure looked similar to this:

```html
<section class="comment">
    <p>
        <a id="author" href="https://test.com"></a>
        testname | DATE
        <img class="avatar" src="/resources/images/avatarDefault.svg">
    </p>
    <p>testcomment</p>
    <p></p>
</section>
```

`DATE` represents the date displayed by the application; it is a placeholder in this example.

This inspection helped me understand where the submitted values appeared in the page and which elements I should investigate further.

I now had several possible areas to examine, especially the comment body and the author information.

### 3. Testing Special Characters

<p>
<img src="images/screenshot 3.png" alt="Screenshot 3 - Testing special characters in the name and comment fields">
</p>

My next step was to test how the application handled special characters.

I submitted characters such as:

```text
< ' " >
```

I wanted to determine whether the application treated these characters as plain text or allowed them to change the page's HTML structure.

The characters appeared as text on the page. When I inspected the DOM, I found that angle brackets were represented using HTML entities such as `&lt;` and `&gt;`.

This suggested that the application was attempting to prevent HTML injection.

However, I did not assume that this protection was secure simply because my initial test did not create an HTML element.

I needed to identify where the encoding occurred and whether it handled every occurrence of the special characters.

### 4. Finding the JavaScript That Loads Comments

<p>
<img src="images/screenshot 4.png" alt="Screenshot 4 - Finding the JavaScript file responsible for loading comments">
</p>

I inspected the page source and found the following code:

```html
<span id="user-comments">
    <script src="/resources/js/loadCommentsWithVulnerableEscapeHtml.js"></script>
    <script>
        loadComments('/post/comment')
    </script>
</span>
```

Two details caught my attention:

- The application loaded a JavaScript file named `loadCommentsWithVulnerableEscapeHtml.js`.
- It called `loadComments('/post/comment')` to retrieve the comments.

This showed that client-side JavaScript was responsible for loading and displaying the stored comments.

I opened the JavaScript file to investigate how the application processed the returned data.

This was an important step because I needed to trace the data flow rather than focus only on the visible output.

### 5. Identifying the Incomplete HTML Encoding

<p>
<img src="images/screenshot 5.png" alt="Screenshot 5 - Inspecting the vulnerable escapeHTML function">
</p>

Inside the JavaScript file, I found the following function:

```javascript
function escapeHTML(html) {
    return html.replace('<', '&lt;').replace('>', '&gt;');
}
```

At first glance, the function appeared to protect against HTML injection by replacing angle brackets with HTML entities.

However, the problem was how JavaScript's `String.replace()` method behaved.

When the first argument is a string, `replace()` changes only the first occurrence of that string. It does not automatically replace every matching character.

For example, consider this input:

```html
A <B> C <D>
```

The function replaces the first `<` with `&lt;`. It then replaces the first `>` in the resulting string with `&gt;`.

The later angle brackets remain unchanged.

A global regular expression would replace all occurrences of these characters:

```javascript
function escapeHTML(html) {
    return html.replace(/</g, '&lt;').replace(/>/g, '&gt;');
}
```

This demonstrates the specific problem with the original implementation. However, for displaying plain text, using `textContent` is generally safer than writing a custom HTML-escaping function.

I had now identified a potential weakness: if I submitted content containing multiple pairs of angle brackets, some markup might survive the encoding process.

My next step was to test that hypothesis without immediately attempting JavaScript execution.

### 6. Confirming the Vulnerability With a Harmless Test

<p>
<img src="images/Screenshot 6.png" alt="Screenshot 6 - Testing whether later angle brackets survive the incomplete encoding">
</p>

Before attempting to execute JavaScript, I tested my hypothesis using harmless HTML-like input:

```html
A <B> C <D>
```

I then inspected the resulting comment in the live DOM.

The output showed that the first angle brackets were encoded, while later markup was not consistently encoded. The browser could therefore interpret some of the remaining characters as HTML syntax.

This confirmed that the `escapeHTML()` function did not handle every occurrence of `<` and `>`.

I also found that the application used `innerHTML` to insert the processed comment into the page. For example, the comment body was handled like this:

```javascript
commentBodyPElement.innerHTML = escapeHTML(comment.body);
```

This was the most important discovery.

Unlike `textContent`, `innerHTML` parses the supplied string as HTML. If the string contains markup that has not been safely handled, the browser can create elements from that markup.

The vulnerability therefore depended on two problems working together:

1. The escaping function did not encode every angle bracket.
2. The resulting content was inserted through an HTML-parsing sink.

I had confirmed the root cause and could now test whether the remaining markup could trigger JavaScript execution.

### 7. Exploiting the Stored DOM XSS

<p>
<img src="images/Screenshot 7.png" alt="Screenshot 7 - Successfully triggering alert(1) with the stored DOM XSS payload">
</p>

After confirming the vulnerability, I submitted the following payload as a blog comment:

```html
<><img src=1 onerror=alert(1)>
```

The payload took advantage of the incomplete encoding.

#### Part 1: `<>`

The initial empty angle-bracket pair was used to consume the first occurrences of `<` and `>` processed by the vulnerable function.

After the replacements, the beginning of the string became:

```text
&lt;&gt;
```

The browser displayed this part as text rather than treating it as an HTML element.

#### Part 2: `<img src=1 onerror=alert(1)>`

The later image element's angle brackets remained unescaped because the function replaced only the first occurrence of each character.

When the application assigned the resulting string to `innerHTML`, the browser interpreted the remaining image markup as an actual HTML element.

#### Part 3: `src=1`

The image's `src` value caused the browser to attempt to load an image resource. In this test, the resource failed to load.

#### Part 4: `onerror=alert(1)`

When the image failed to load, the browser triggered the `error` event. The inline `onerror` handler executed `alert(1)`.

The alert appeared, confirming that I had successfully exploited the stored DOM XSS vulnerability.

Because the comment was stored by the application and processed again when the blog comments were loaded, the malicious input remained available for client-side processing.

## Understanding the Root Cause

My investigation revealed the following data flow:

1. I submitted a comment through the blog's comment form.
2. The application stored the comment.
3. Client-side JavaScript retrieved the stored comments through an XMLHttpRequest.
4. The `escapeHTML()` function replaced only the first occurrence of each angle bracket.
5. The resulting string was assigned to `innerHTML`.
6. The browser interpreted the remaining markup, allowing the image's `onerror` handler to execute JavaScript.

The vulnerability was stored because the malicious comment was saved and processed again when comments were loaded. It was DOM-based because the unsafe HTML interpretation occurred in client-side JavaScript.

The core issue was **incomplete HTML encoding combined with inserting user-controlled content through `innerHTML`**.

## Key Takeaway

This lab taught me to trace user-controlled data from its source to the point where the browser interprets it, instead of relying on the appearance of the rendered page alone.

I started with a normal comment, inspected how the data appeared in the DOM, tested special characters, and located the JavaScript responsible for loading stored comments.

The key discovery was that `String.replace()` with a string argument replaces only the first occurrence. Because the application then used `innerHTML`, the remaining markup could still be interpreted by the browser.

**The main lesson:** never rely on incomplete escaping to make untrusted HTML safe. When displaying plain text, prefer `textContent` instead of `innerHTML`. If user-supplied HTML must be supported, use a reputable HTML sanitizer.

## Final Payload

```html
<><img src=1 onerror=alert(1)>
```

**Result:** Successfully triggered `alert(1)` and solved the Stored DOM XSS lab.
