# Reflected XSS in Search Functionality

## 1. Lab / Target

**Platform:** PortSwigger Web Security Academy
**Vulnerability:** Reflected Cross-Site Scripting (XSS)
**Target:** PortSwigger Web Security Academy Lab

The objective of this lab was to identify and exploit a reflected Cross-Site Scripting (XSS) vulnerability in the application's search functionality.

---

## 2. Initial Reconnaissance

When I opened the lab, I noticed a **search bar** on the page.

Since search functionality commonly processes user-controlled input and reflects it back into the page, I suspected that the search parameter might be vulnerable to XSS.

I started with a harmless test value:

```text
test123
```

After submitting the search, the URL changed to:

```text
https://0ac200240341f2ac805dd05d00b500d9.web-security-academy.net/?search=test123
```

This showed that the search input was being sent to the server through the `search` parameter.

### Evidence

At this point, I captured a screenshot showing:

1. The search bar containing `test123`.
2. `test123` being displayed on the page.
3. The corresponding HTML containing the same value.

---

## 3. Identifying the Injection Point

Inspecting the HTML showed that the input was reflected into the page as HTML content:

```html
<h1>
    <span>0 search results for '</span>
    <span id="searchMessage">test123</span>
    <span>'</span>
</h1>
```

The important observation was that my input was inserted directly into the HTML response inside the `searchMessage` element.

This indicated that the application was reflecting user-controlled input without sufficiently encoding it for the HTML context.

Therefore, instead of being treated only as text, the browser could potentially interpret HTML supplied through the `search` parameter.

---

## 4. Testing for XSS

After confirming that my input was reflected into the HTML, I tested whether I could inject an HTML element capable of executing JavaScript.

I used the following payload:

```html
<img src=1 onerror=alert(1)>
```

The payload intentionally uses an invalid image source. When the browser attempts to load the image and the load fails, the `onerror` event handler executes:

```javascript
alert(1)
```

The resulting URL was URL-encoded by the browser:

```text
https://0ac200240341f2ac805dd05d00b500d9.web-security-academy.net/?search=%3Cimg+src%3D1+onerror%3Dalert%281%29%3E
```

---

## 5. PoC / Evidence

After submitting the payload, the application reflected it into the HTML as an actual HTML element:

```html
<img src="1" onerror="alert(1)">
```

Because the browser interpreted the injected input as HTML, it created the `<img>` element.

The image source was invalid, causing the `error` event to fire and execute the JavaScript contained in the `onerror` attribute.

The successful execution of:

```javascript
alert(1)
```

confirmed the presence of reflected XSS.

### Screenshots

I included screenshots demonstrating:

**Screenshot 1 — Search Request**

The URL contains the injected payload in the `search` parameter.

**Screenshot 2 — HTML Injection**

The application's HTML contains:

```html
<img src="1" onerror="alert(1)">
```

showing that the payload was interpreted as HTML rather than displayed as plain text.

**Screenshot 3 — XSS Execution**

The JavaScript payload successfully executes and produces the `alert(1)` dialog.

---

## 6. Technical Explanation

The vulnerability occurs because user-controlled input from the `search` parameter is reflected into the application's HTML response without appropriate output encoding.

The original request:

```text
?search=test123
```

resulted in the application inserting:

```html
<span id="searchMessage">test123</span>
```

Because the application does not safely encode HTML metacharacters in the reflected input, I was able to replace the harmless text with an HTML element:

```html
<img src=1 onerror=alert(1)>
```

The browser parsed the injected string as HTML and created an `<img>` element.

The `src` value points to an invalid resource, so the browser triggers the `error` event. The JavaScript inside the event handler then executes:

```javascript
alert(1)
```

This demonstrates that arbitrary JavaScript can be executed in the context of the vulnerable page.

An important point is that the payload works because the injection occurs in an **HTML context**. The fact that the input appears between `<span>` elements does not itself make XSS possible; rather, the input is being inserted as HTML content, allowing the injected `<img>` element to be parsed by the browser.

---

## 7. Impact

In a real-world application, reflected XSS can allow an attacker to execute JavaScript in another user's browser under the security context of the vulnerable website.

Depending on the application's functionality and security controls, potential consequences can include:

* Manipulation of page content.
* Performing actions on behalf of the victim.
* Accessing data available to JavaScript within the application's origin.
* Phishing or UI manipulation.
* Abuse of authenticated functionality available to the victim.

The exact impact depends on the application's architecture, authentication model, browser protections, and other security controls.

---

## 8. Conclusion / Takeaway

This lab helped me understand how reflected XSS can occur when user-controlled input is inserted into an HTML response without appropriate output encoding.

My approach was like this:

```text
Identify user input
        ↓
Test how the input is reflected
        ↓
Inspect the HTML context
        ↓
Determine whether HTML can be injected
        ↓
Test an HTML element with an event handler
        ↓
Confirm JavaScript execution
```

The key lesson I took from this lab is that when testing for XSS, it is important to first understand **where the input is reflected and in what context**. The appropriate payload depends on that context.
