\# Reflected XSS into a JavaScript string with angle brackets HTML encoded



\## Lab Description



This lab contains a reflected Cross-Site Scripting (XSS) vulnerability in the search query tracking functionality.



The search input is reflected inside a JavaScript string, while angle brackets such as `<` and `>` are HTML encoded.



My goal was to break out of the JavaScript string and execute `alert(1)`.



\---



\## Solution



\### 1. Identifying the Injection Point



I started by entering a simple value:



```text

test123

```



After searching, I inspected the page source and found:



```javascript

var searchTerms = 'test123';

```



This showed me that my search input was being reflected directly inside a JavaScript string.



So the important part was:



```javascript

var searchTerms = 'USER\_INPUT';

```



This meant the `search` parameter was an interesting injection point to investigate.



<p>

<img src="images/Screenshot 1.png" alt="Search input reflected inside JavaScript string">

</p>



\---



\### 2. Testing Whether I Could Break Out of the String



Since my input was inside a JavaScript string surrounded by single quotes, I tested a single quote:



```text

test'

```



The resulting JavaScript became:



```javascript

var searchTerms = 'test'';

```



The single quote I entered was reflected as an actual `'` character.



This was important because it showed that the single quote was not being encoded.



I could therefore interfere with the JavaScript string syntax and potentially break out of the original string.



<p>

<img src="images/Screenshot 2.png" alt="Testing a single quote to break out of the JavaScript string">

</p>



\---



\### 3. Testing Angle Brackets



I also tested an angle bracket:



```text

<

```



The application changed it to:



```html

\&lt;

```



This showed that angle brackets were being HTML encoded.



Therefore, I could not simply rely on `<script>` or HTML tags to perform the XSS.



More importantly, I was already inside an existing `<script>` block, so I didn't actually need to create a new `<script>` tag.



The vulnerability was in the JavaScript context itself.



<p>

<img src="images/Screenshot 3.png" alt="Angle bracket HTML encoding">

</p>



\---



\### 4. Breaking Out of the JavaScript String



At this point, I knew that my input was inside:



```javascript

var searchTerms = 'USER\_INPUT';

```



I needed to create valid JavaScript outside of the original string.



I started by using a single quote:



```text

'

```



This closes the string created by the application.



Then I needed my JavaScript code to execute.



I used:



```text

\-alert(1)-

```



Finally, I used another single quote to create the closing string.



My complete input was:



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



The important difference is that `alert(1)` is \*\*not inside quotes\*\*, so JavaScript interprets it as a function call rather than as text.



The `-` operators allow the resulting expression to remain valid JavaScript while causing `alert(1)` to be evaluated.



<p>

<img src="images/Screenshot 4.png" alt="Breaking out of the JavaScript string and executing alert">

</p>



\---



\### 5. Confirming the XSS



After loading the resulting URL, the JavaScript executed and the browser displayed:



```text

alert(1)

```



This confirmed that I had successfully performed reflected XSS.



<p>

<img src="images/Screenshot 5.png" alt="Alert confirming successful XSS">

</p>



\---



\## Technical Explanation



The vulnerable code was:



```javascript

var searchTerms = 'USER\_INPUT';

```



My input was placed directly inside a JavaScript string.



The important context was therefore:



\*\*JavaScript string context\*\*



The application did HTML-encode angle brackets, which prevented me from using HTML tags such as `<script>`.



However, the single quote was not encoded.



That allowed me to break out of the JavaScript string and introduce JavaScript syntax.



The final payload was:



```text

'-alert(1)-'

```



Which resulted in:



```javascript

var searchTerms = ''-alert(1)-'';

```



Because `alert(1)` is outside the quotes, the JavaScript engine evaluates it as executable code.



\---



\## Key Takeaway



The most important lesson from this lab was to \*\*identify the context before choosing an XSS payload\*\*.



My input was not being inserted into normal HTML. It was being inserted into a \*\*JavaScript string\*\*.



So instead of trying to inject an HTML `<script>` tag, I needed to:



1\. Identify the JavaScript string.

2\. Test whether I could break out of it.

3\. Confirm that the single quote was not encoded.

4\. Create valid JavaScript syntax.

5\. Execute `alert(1)` outside the string.



The vulnerability was not simply that my input was reflected.



The important combination was:



```text

User input

&#x20;   ↓

Reflected into JavaScript

&#x20;   ↓

Inside a JavaScript string

&#x20;   ↓

Single quote not encoded

&#x20;   ↓

String can be broken

&#x20;   ↓

JavaScript can be injected

&#x20;   ↓

alert(1) executes

```



\---



\## Final Payload



```text

'-alert(1)-'

```



\## Vulnerability Summary



\*\*Vulnerability:\*\* Reflected Cross-Site Scripting (XSS)



\*\*Context:\*\* JavaScript string



\*\*Source:\*\* `search` query parameter



\*\*Injection point:\*\*



```javascript

var searchTerms = 'USER\_INPUT';

```



\*\*Protection observed:\*\* Angle brackets were HTML encoded.



\*\*Missing protection:\*\* The single quote was not safely encoded for the JavaScript string context.



\*\*Result:\*\* I was able to break out of the JavaScript string and execute arbitrary JavaScript in the victim's browser.

