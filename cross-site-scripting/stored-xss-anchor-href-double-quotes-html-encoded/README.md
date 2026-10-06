\# Stored XSS into anchor `href` Attribute with Double Quotes HTML-Encoded



\## Lab Description



\*\*Platform:\*\* PortSwigger Web Security Academy

\*\*Vulnerability:\*\* Stored Cross-Site Scripting (XSS)

\*\*Context:\*\* HTML `href` attribute

\*\*Status:\*\* Completed



This lab contains a stored cross-site scripting vulnerability in the comment functionality.



The objective was to submit a comment that causes the `alert()` function to execute when the comment author's name is clicked.



The important part of this lab is that the application HTML-encodes double quotes inside the `href` attribute. Therefore, instead of breaking out of the attribute, I needed to find another way to make the existing `href` execute JavaScript.



\---



\## Solution



\### 1. Testing the Comment Functionality



The lab contains a blog post with a comment section.



Because the lab description specifically mentions a stored XSS vulnerability in the comment functionality, I started by testing the comment form with harmless values.



I submitted:



```text

Comment: testcomment1

Name: testname1

Email: testemail@gmail.com

Website: test.com

```



After submitting the comment, I returned to the blog post and checked how my information was rendered.



<p>

<img src="images/Screenshot 1.png" alt="Submitting a test comment">

</p>



The comment was successfully stored and displayed on the page.



This confirmed that user-controlled data submitted through the comment form was being stored by the application and later displayed when the blog post was viewed.



This is an important characteristic of \*\*stored XSS\*\*:



```text

User input

&#x20;   ↓

Application stores the input

&#x20;   ↓

Later request to the page

&#x20;   ↓

Stored input is rendered

&#x20;   ↓

Browser processes the input

```



\---



\### 2. Identifying the Injection Point



After the comment was displayed, I noticed that the comment author's name was a hyperlink.



Inspecting the HTML showed:



```html

<a id="author" href="https://www.test.com">testname1</a>

```



<p>

<img src="images/Screenshot 2.png" alt="Comment author stored inside href attribute">

</p>



This was the important discovery.



My input from the \*\*Website\*\* field was being stored and inserted into the `href` attribute of the author's name:



```html

<a id="author" href="USER\_INPUT">testname1</a>

```



Therefore, the injection point was not the comment text or the author's name.



The user-controlled value was being inserted into an HTML `href` attribute.



At this point, I needed to understand what I could do with the value inside the `href`.



\---



\### 3. Testing Whether I Could Escape the `href` Attribute



My first idea was to try to break out of the existing `href` attribute.



The original structure was:



```html

<a id="author" href="USER\_INPUT">testname1</a>

```



If the double quote was accepted by the application, I might be able to change the HTML structure by injecting another attribute.



I therefore tested characters including:



```text

https://:?<.com

```



The important part of this test was the double quote.



The application returned the double quote as:



```html

\&quot;

```



<p>

<img src="images/Screenshot 3.png" alt="Double quote HTML encoded inside href attribute">

</p>



This showed that the application was HTML-encoding double quotes.



For example:



```text

"

↓

\&quot;

```



Therefore, I could not simply inject something like:



```html

" onclick="alert(1)

```



because the quote was not interpreted as the end of the existing `href` attribute.



This ruled out the attribute-breakout approach.



\---



\### 4. Looking at the `href` Context



Since I could not escape the `href` attribute, I changed my approach.



Instead of asking:



> How can I escape the `href` attribute?



I asked:



> What can an `href` value itself contain?



A normal link might look like:



```html

<a href="https://www.example.com">testname1</a>

```



The browser interprets `https://` as a URL scheme and navigates to the specified resource.



However, an `href` can also use other URL schemes.



One special scheme is:



```text

javascript:

```



A `javascript:` URL is interpreted by the browser as JavaScript rather than as a normal HTTP/HTTPS destination.



This meant that I did not need to escape the `href` attribute.



Instead, I could control the value that was already inside the attribute.



The structure could remain:



```html

<a id="author" href="USER\_INPUT">testname1</a>

```



while the value of `USER\_INPUT` could be interpreted as JavaScript.



\---



\### 5. Injecting a JavaScript URL



I therefore submitted the following value in the \*\*Website\*\* field:



```text

javascript:alert(1)

```



The application stored the value and rendered it as:



```html

<a id="author" href="javascript:alert(1)">testname1</a>

```



This was the key result.



I had not escaped the `href` attribute.



Instead, I controlled the value inside the existing attribute and changed its meaning by using the `javascript:` URL scheme.



The browser therefore interprets:



```text

javascript:alert(1)

```



as a JavaScript URL.



When the link is activated, the browser executes:



```javascript

alert(1)

```



\---



\### 6. Successful XSS



After clicking the comment author's name, the browser executed the JavaScript and displayed the alert dialog.



<p>

<img src="images/Screenshot 4.png" alt="Successful stored XSS alert">

</p>



The successful execution of:



```javascript

alert(1)

```



confirmed that the stored XSS vulnerability was successfully exploited.



The complete attack flow was:



```text

Website input

&#x20;     ↓

Application stores the input

&#x20;     ↓

Stored value is rendered into:

<a href="USER\_INPUT">

&#x20;     ↓

Double quotes are HTML-encoded

&#x20;     ↓

Cannot escape the href attribute

&#x20;     ↓

Analyze the href context

&#x20;     ↓

Use the javascript: URL scheme

&#x20;     ↓

href="javascript:alert(1)"

&#x20;     ↓

User clicks the author's name

&#x20;     ↓

JavaScript executes

&#x20;     ↓

alert(1)

```



\---



\## Technical Explanation



The vulnerability exists because the application stores user-controlled input from the \*\*Website\*\* field and later inserts it into the `href` attribute of the comment author's name.



The original HTML structure is:



```html

<a id="author" href="USER\_INPUT">testname1</a>

```



The application does protect the attribute from a traditional attribute-breakout attack by HTML-encoding double quotes:



```text

" → \&quot;

```



Therefore, an attack such as:



```html

" onclick="alert(1)

```



does not create a new HTML attribute.



However, the application still allows the attacker to control the actual value of the `href` attribute.



A normal value such as:



```text

https://www.test.com

```



causes the browser to navigate to that website.



A `javascript:` URL is interpreted differently:



```text

javascript:alert(1)

```



The browser treats the value after the `javascript:` scheme as JavaScript code.



Therefore, the final HTML becomes:



```html

<a id="author" href="javascript:alert(1)">testname1</a>

```



When the user clicks the author's name, the browser processes the `javascript:` URL and executes:



```javascript

alert(1)

```



This demonstrates that HTML-encoding the double quote alone was not sufficient to prevent XSS.



The application needed to safely handle the value according to the security requirements of the `href` context and prevent dangerous URL schemes such as `javascript:`.



\---



\## Impact



In a real-world application, stored XSS can be more dangerous than reflected XSS because the malicious input is stored by the application and can be served to other users.



Depending on the application's functionality and security controls, stored XSS could potentially allow an attacker to:



\* Execute JavaScript in another user's browser.

\* Manipulate the application's page content.

\* Perform actions using the victim's authenticated session.

\* Conduct phishing or UI manipulation attacks.

\* Access data available to JavaScript within the vulnerable origin.



The exact impact depends on the application's architecture, authentication model, browser protections, and other security controls.



\---



\## Key Takeaway



The main lesson from this lab is that \*\*HTML context matters\*\*, and escaping the HTML attribute is not always necessary for XSS.



My initial approach was to try to escape the `href` attribute using a double quote:



```text

"

```



However, the application encoded the character:



```text

" → \&quot;

```



This prevented a traditional attribute-breakout attack.



Instead of continuing to search for a way to escape the attribute, I looked at what the `href` value itself could represent.



The important discovery was that an `href` can use different URL schemes, including:



```text

javascript:

```



This allowed me to keep the existing HTML structure intact while controlling the behavior of the `href` value.



The final attack therefore became:



```text

Website input

&#x20;     ↓

Stored by application

&#x20;     ↓

Inserted into href

&#x20;     ↓

javascript: URL scheme

&#x20;     ↓

User clicks author name

&#x20;     ↓

JavaScript executes

```



This lab reinforced an important XSS mindset:



> \*\*When testing XSS, do not only ask how to escape the current context. Ask what the browser does with the value while you are inside that context.\*\*



\---



\## Final Payload



```text

javascript:alert(1)

```



\## Vulnerability Summary



\*\*Source:\*\*



```text

Website field in the comment form

```



\*\*Storage:\*\*



```text

Comment data stored by the application

```



\*\*Injection point:\*\*



```html

<a id="author" href="USER\_INPUT">testname1</a>

```



\*\*Protection observed:\*\*



```text

Double quotes are HTML-encoded:

" → \&quot;

```



\*\*Bypass:\*\*



```text

Use the javascript: URL scheme instead of escaping the href attribute.

```



\*\*Result:\*\*



```text

Stored XSS through a JavaScript URL in the anchor href attribute.

```



