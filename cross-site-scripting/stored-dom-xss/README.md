\# Stored DOM XSS



\## Lab Description



\*\*Difficulty:\*\* Practitioner



The goal of this lab was to exploit a stored DOM-based Cross-Site Scripting vulnerability in the blog comment functionality and trigger `alert(1)`.



The website attempted to prevent HTML injection by encoding angle brackets in user-submitted comments. However, the JavaScript function responsible for this protection only replaced the first occurrence of each character.



By investigating the client-side JavaScript, testing my hypothesis, and identifying the unsafe `innerHTML` sink, I was able to bypass the incomplete encoding and execute JavaScript.



\## Solution



\### 1. Testing the Comment Functionality



I started by investigating the blog's comment section because the lab description mentioned a vulnerability in the comment functionality.



I submitted a normal comment with a name, email address, and website to observe how the application handled my input.



The website accepted the website field when I entered a valid URL beginning with `https://`.



My first step was to establish normal behavior before testing for a vulnerability.



<p>

<img src="images/Screenshot 1.png" alt="Screenshot 1 - Submitting a normal blog comment">

</p>



\### 2. Inspecting the Comment in the DOM



After submitting my comment, I inspected the live DOM in the browser's developer tools.



I found that the author name, website, and comment body were rendered in different parts of the comment section.



The website was placed inside an anchor's `href` attribute, while the comment body was displayed inside a paragraph.



This gave me three areas to investigate: the name, the comment body, and the website attribute.



<p>

<img src="images/Screenshot 2.png" alt="Screenshot 2 - Inspecting the rendered comment and its HTML elements">

</p>



\### 3. Testing Special Characters



My next step was to test how the application handled special characters.



I submitted angle brackets and quotation marks in the name and comment fields.



On the page, the characters appeared as text. However, inspecting the DOM showed that the angle brackets had been encoded as `\&lt;` and `\&gt;`.



This suggested that the application was trying to prevent HTML injection.



Instead of assuming that the protection was secure, I decided to investigate how the application processed the input.



<p>

<img src="images/Screenshot 3.png" alt="Screenshot 3 - Testing special characters and checking their representation in the DOM">

</p>



\### 4. Finding the JavaScript That Loads Comments



I inspected the page source and found the following code:



```html

<span id="user-comments">

&#x20;   <script src="/resources/js/loadCommentsWithVulnerableEscapeHtml.js"></script>

&#x20;   <script>loadComments('/post/comment')</script>

</span>

```



This showed that the comment section was populated by client-side JavaScript.



I opened `loadCommentsWithVulnerableEscapeHtml.js` to investigate how the stored comments were retrieved and displayed.



This was an important step because the vulnerability involved DOM manipulation rather than simply reflecting input directly into the server-generated HTML.



<p>

<img src="images/Screenshot 4.png" alt="Screenshot 4 - Finding the JavaScript file responsible for loading comments">

</p>



\### 5. Identifying the Incomplete HTML Encoding



Inside the JavaScript file, I found the following function:



```javascript

function escapeHTML(html) {

&#x20;   return html.replace('<', '\&lt;').replace('>', '\&gt;');

}

```



The function was intended to encode angle brackets to prevent user input from being interpreted as HTML.



However, JavaScript's `replace()` method replaces only the first occurrence when its search argument is a string.



This meant that the function encoded only the first `<` and the first `>`, leaving subsequent occurrences unchanged.



For example, a global regular expression would replace all occurrences:



```javascript

function escapeHTML(html) {

&#x20;   return html.replace(/</g, '\&lt;').replace(/>/g, '\&gt;');

}

```



I now had a hypothesis: if I submitted input containing multiple pairs of angle brackets, later markup might survive the incomplete encoding.



<p>

<img src="images/Screenshot 5.png" alt="Screenshot 5 - Discovering the vulnerable escapeHTML function">

</p>



\### 6. Confirming the Vulnerability With a Harmless Test



Before attempting to execute JavaScript, I tested my hypothesis using harmless HTML-like input:



```html

A <B> C <D>

```



I inspected the resulting comment in the live DOM and observed that the first angle brackets were encoded, while later markup was not consistently encoded.



This confirmed that the escaping function did not process every occurrence.



I also found that the comment body was inserted using `innerHTML`. Unlike `textContent`, `innerHTML` parses the supplied string as HTML.



The combination of incomplete encoding and an HTML-parsing sink made it possible for remaining markup to be interpreted by the browser.



<p>

<img src="images/Screenshot 6.png" alt="Screenshot 6 - Confirming that later angle brackets survive the incomplete encoding">

</p>



\### 7. Exploiting the Stored DOM XSS



After confirming the vulnerability, I submitted the following comment:



```html

<><img src=1 onerror=alert(1)>

```



The function encoded the first pair of angle brackets but left the subsequent image element's angle brackets unchanged.



The application then assigned the processed comment to `innerHTML`, allowing the browser to interpret the remaining image element as HTML.



Because the image source failed to load, its `onerror` event handler executed `alert(1)`.



The alert appeared, confirming that the exploit worked.



<p>

<img src="images/Screenshot 7.png" alt="Screenshot 7 - Successfully triggering alert(1)">

</p>



\## Key Takeaway



This lab taught me to trace user-controlled data from its source to the point where the browser interprets it.



The vulnerable flow was:



1\. The user submitted a comment that was stored by the application.

2\. Client-side JavaScript retrieved the stored comment.

3\. The `escapeHTML()` function encoded only the first occurrence of each angle bracket.

4\. The resulting string was assigned to `innerHTML`.

5\. The browser interpreted the remaining markup, allowing JavaScript execution.



The main lesson is that security functions must handle all relevant input correctly. When displaying plain text, `textContent` is safer than `innerHTML`. If user-supplied HTML must be supported, a reputable HTML sanitizer should be used.



\## Final Payload



```html

<><img src=1 onerror=alert(1)>

```



\*\*Result:\*\* Successfully triggered `alert(1)` and solved the Stored DOM XSS lab.

