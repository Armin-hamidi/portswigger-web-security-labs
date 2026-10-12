

\# DOM XSS in AngularJS Expression with Angle Brackets and Double Quotes HTML-Encoded



\## Lab Description



This lab demonstrates a DOM-based Cross-Site Scripting (XSS) vulnerability in AngularJS.



The application reflects the search input into the page, where AngularJS evaluates expressions inside double curly braces `{{ }}`. This can allow an attacker to execute JavaScript through an AngularJS expression without injecting an HTML element.



The goal is to execute `alert(1)` using the vulnerable search functionality.



\## Solution



\### 1. Identifying AngularJS



<p>

<img src="images/Screenshot 1.png" alt="Screenshot 1 - Identifying AngularJS with Wappalyzer">

</p>



I opened the lab and started by checking the technologies used by the website with the Wappalyzer browser extension.



Wappalyzer identified \*\*AngularJS 1.7.7\*\*.



I then inspected the page source and noticed AngularJS-related markup. For example, the search form contained these classes:



```html

<form action="/" method="GET" class="ng-pristine ng-valid">

```



These classes are added by AngularJS to describe the form's state. They were useful clues that AngularJS was managing the form, but they did not prove the vulnerability by themselves.



I needed to investigate how AngularJS was processing the search input.



\### 2. Testing AngularJS Expression Evaluation



<p>

<img src="images/Screenshot 2.png" alt="Screenshot 2 - Testing AngularJS expression evaluation">

</p>



The lab description mentioned AngularJS expressions, so I decided to test whether the application evaluated expressions supplied through the search field.



I entered the following harmless test:



```text

{{7\*7}}

```



The page displayed `49` in the search results heading. However, when I viewed the original page source, I could still see the expression `{{7\*7}}`.



This difference was important:



\- \*\*View Page Source\*\* showed the original HTML returned by the server.

\- \*\*Inspect → Elements\*\* showed the live DOM after AngularJS had processed the page.



AngularJS evaluated the expression and updated the displayed result from `{{7\*7}}` to `49`.



This confirmed that my input was reaching an AngularJS expression context.



I then considered how I could make an expression do more than a calculation. In ordinary JavaScript, the `Function` constructor can create a function from a string of code:



```javascript

const myFunction = Function("alert(1)");

myFunction();

```



This example helped me understand the concept behind the exploit. I still needed to find a way to reach the function constructor through the AngularJS expression context.



\### 3. Exploiting the AngularJS Expression



<p>

<img src="images/Screenshot 3.png" alt="Screenshot 3 - Executing the AngularJS expression payload">

</p>



After investigating how JavaScript functions and their `constructor` property work, I used the following payload in the search field:



```javascript

{{$on.constructor('alert(1)')()}}

```



The payload uses AngularJS expression evaluation to reach a function constructor and execute JavaScript.



Here is how each part works.



\#### Part 1: `{{ ... }}`



The double curly braces tell AngularJS to evaluate the expression inside them.



I had already confirmed this behavior with my earlier test:



```text

{{7\*7}}

```



AngularJS evaluated it and displayed `49`.



\#### Part 2: `$on`



`$on` is a method available on an AngularJS scope. In this expression context, it provides a reference to a function.



The payload uses this function as a starting point to access its constructor.



\#### Part 3: `.constructor`



JavaScript functions have a `constructor` property. For the function used here, this property provides access to JavaScript's built-in `Function` constructor.



The `Function` constructor can create a new function from a string containing code.



\#### Part 4: `('alert(1)')`



The string `'alert(1)'` is passed to the constructor as the body of the new function.



Conceptually, the result is similar to creating a function like this:



```javascript

function generatedFunction() {

&#x20;   alert(1);

}

```



This is a conceptual equivalent, not the literal code inserted into the page.



\#### Part 5: The final `()`



The constructor returns a function. The final `()` calls that function, causing `alert(1)` to execute.



The complete payload therefore uses AngularJS expression evaluation to reach JavaScript function construction and invoke the resulting function.



\### 4. Understanding the Vulnerability



The important discovery was not just the final payload. It was identifying how the application processed my input.



My investigation showed the following:



1\. The search input was submitted through the `search` URL parameter.

2\. The search value appeared in the results heading.

3\. The page used AngularJS and had an `ng-app` directive on the body.

4\. The expression `{{7\*7}}` was evaluated by AngularJS.

5\. The AngularJS expression context could be exploited to execute JavaScript.



The lab also encoded angle brackets and double quotes. However, the exploit did not need to inject a new HTML element. Instead, it abused AngularJS expression evaluation.



This demonstrates why HTML encoding alone is not a complete defense when untrusted input is interpreted as a client-side template expression.



\## Key Takeaway



This lab taught me to investigate XSS systematically instead of guessing payloads.



I started by identifying the JavaScript framework, inspected how the search value reached the page, and used `{{7\*7}}` to confirm that AngularJS evaluated user-controlled expressions.



Once I understood the execution context, I investigated the JavaScript function constructor and used it to execute `alert(1)`.



The main lesson is that \*\*untrusted input must never be evaluated as an AngularJS expression or compiled as a template\*\*. Applications should keep user data separate from executable expressions and use appropriate output handling.



\## Final Payload



```javascript

{{$on.constructor('alert(1)')()}}

```



