# DOM XSS in AngularJS Expression with Angle Brackets and Double Quotes HTML-Encoded

## Lab Description

This lab demonstrates a DOM-based Cross-Site Scripting (XSS) vulnerability caused by AngularJS evaluating user-controlled expressions.

The application reflects the search input into the page. Because AngularJS processes expressions inside double curly braces `{{ }}`, an attacker can potentially execute JavaScript without injecting an HTML element.

The lab's goal is to exploit the vulnerable search functionality and execute `alert(1)`.

## Solution

### 1. Identifying AngularJS

<p>
<img src="images/Screenshot 1.png" alt="Screenshot 1 - Identifying AngularJS with Wappalyzer">
</p>

I started by identifying the technologies used by the application with the Wappalyzer browser extension.

Wappalyzer detected **AngularJS 1.7.7**. This was an important clue because AngularJS supports expression evaluation through interpolation syntax such as `{{ expression }}`.

I then inspected the page markup and found AngularJS-related attributes and classes. For example, the search form contained:

```html
<form action="/" method="GET" class="ng-pristine ng-valid">
```

The `ng-pristine` and `ng-valid` classes are AngularJS form-state indicators. They suggested that AngularJS was managing the form, but they did not prove an XSS vulnerability by themselves.

My next step was to investigate whether the application evaluated expressions originating from the search input.

### 2. Confirming AngularJS Expression Evaluation

<p>
<img src="images/Screenshot 2.png" alt="Screenshot 2 - Confirming AngularJS expression evaluation">
</p>

Instead of immediately trying an exploit payload, I started with a harmless expression:

```text
{{7*7}}
```

After submitting it through the search field, the page displayed `49` in the results heading.

This was a significant discovery. The application was not simply displaying my input as text. AngularJS was evaluating the expression and rendering its result.

I also compared the original page source with the live DOM:

- **View Page Source** showed the original HTML response, including the expression.
- **Inspect → Elements** showed the live DOM after the browser and AngularJS had processed the page.

The original expression could still appear in the source while the rendered page displayed `49`. This helped me distinguish the server's HTML response from the result of client-side expression evaluation.

I had now confirmed that user-controlled input was reaching an AngularJS expression context.

### 3. Investigating JavaScript Function Construction

Once I confirmed expression evaluation, I needed to understand how an expression could execute JavaScript rather than just perform a calculation.

One relevant JavaScript feature is the built-in `Function` constructor, which can create a function from a string:

```javascript
const myFunction = Function("alert(1)");
myFunction();
```

The constructor creates a function whose body contains `alert(1)`. Calling the generated function executes that code.

The remaining challenge was to reach this constructor from the AngularJS expression context. This was the key idea behind the final payload.

### 4. Executing the AngularJS Expression

<p>
<img src="images/Screenshot 3.png" alt="Screenshot 3 - Using the AngularJS expression payload">
</p>

I used the following payload in the search field:

```javascript
{{$on.constructor('alert(1)')()}}
```

The payload takes advantage of AngularJS expression evaluation to access JavaScript's function constructor and invoke the resulting function.

Here is how it works.

#### Part 1: `{{ ... }}`

The double curly braces define an AngularJS interpolation expression.

I had already confirmed that AngularJS evaluated expressions using this harmless test:

```text
{{7*7}}
```

The same expression-processing mechanism is used by the exploit payload.

#### Part 2: `$on`

`$on` is an AngularJS scope method associated with registering event listeners.

In this payload, it serves as a reference to a function. That function reference provides a route to its `constructor` property.

#### Part 3: `.constructor`

In JavaScript, functions have a `constructor` property that normally references the built-in `Function` constructor.

Accessing it allows the expression to construct a new function from a string of JavaScript code.

#### Part 4: `('alert(1)')`

The string `'alert(1)'` is passed to the constructor as the body of the new function.

Conceptually, this is similar to creating a function like:

```javascript
function generatedFunction() {
    alert(1);
}
```

This is a conceptual equivalent used to explain the behavior, not literal code inserted into the page.

#### Part 5: The final `()`

The constructor returns a function. The final `()` invokes that function, causing `alert(1)` to execute.

The complete payload therefore combines AngularJS expression evaluation, access to JavaScript's function constructor, and function invocation.

### 5. Understanding the Root Cause

The key discovery was the way the application processed my input, rather than the payload alone.

My investigation established the following chain:

1. The search form submitted the input through the `search` URL parameter.
2. The search value appeared in the page's results heading.
3. The page used AngularJS, with an `ng-app` directive enabling AngularJS processing.
4. The test expression `{{7*7}}` evaluated to `49`.
5. The vulnerable expression context allowed a crafted AngularJS expression to execute JavaScript.

The lab encoded angle brackets and double quotes, but the successful approach did not depend on injecting an HTML element. Instead, it exploited AngularJS expression evaluation.

This is why I needed to identify the **execution context**, not just check whether HTML special characters were encoded. HTML encoding can help prevent HTML injection, but it does not make it safe to evaluate untrusted input as a client-side template expression.

## Key Takeaway

This lab taught me to investigate XSS methodically instead of guessing payloads.

I identified the JavaScript framework, traced how the search input appeared in the page, and used the harmless expression `{{7*7}}` to confirm that AngularJS evaluated user-controlled expressions.

After identifying the vulnerable context, I investigated how JavaScript's `Function` constructor could be reached through an AngularJS expression and used that understanding to build the final payload.

**The main lesson:** never treat untrusted user input as an AngularJS expression or compile it as a template. Keep user data separate from executable expressions, and use safe rendering techniques that do not evaluate user-controlled content.

## Final Payload

```javascript
{{$on.constructor('alert(1)')()}}
```
