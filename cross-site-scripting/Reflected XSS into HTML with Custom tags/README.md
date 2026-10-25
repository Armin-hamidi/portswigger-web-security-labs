# Reflected XSS into HTML Context with All Tags Blocked Except Custom Ones

## Lab Description

This lab contains a reflected Cross-Site Scripting (XSS) vulnerability. Most standard HTML tags are blocked, but custom HTML tags are allowed.

My goal was to find an allowed tag, identify an event handler that could execute JavaScript, and trigger an alert using the exploit server.

## Solution

### 1. Testing the Search Input

I started by searching for `test` and testing special characters such as `"`, `'`, `>`, and `<` to see how the application handled my input.

This helped me investigate how the search term was reflected in the page and whether the input was being filtered or encoded.

<p>
<img src="images/Screenshot 1.png" alt="Screenshot 1 - Testing special characters in the search input">
</p>

### 2. Finding Allowed HTML Tags

Next, I used Burp Suite Intruder to test different HTML tags.

Some tags returned a `200` status code, which helped me identify tags that the application might allow. A successful status code alone does not prove that a tag is allowed, so I also checked the response to see whether the tag was preserved.

<p>
<img src="images/Screenshot 2.png" alt="Screenshot 2 - Testing HTML tags with Burp Suite Intruder">
</p>

### 3. Testing Event Handler Attributes

After identifying potentially allowed tags, I tested event handler attributes to see which ones the application accepted.

Some responses returned `200` and preserved the attributes. I then needed to test whether the browser would actually trigger the event handler. Being present in the response does not automatically mean JavaScript will execute.

<p>
<img src="images/Screenshot 3.png" alt="Screenshot 3 - Testing event handler attributes">
</p>

### 4. Creating a Focusable Custom Element

I tested the following payload in the search input:

```html
<xss id="x" tabindex="1" onfocus="alert(1)">hello</xss>
```

This payload uses a custom HTML element named `xss`.

- `id="x"` gives the element an identifier.
- `tabindex="1"` makes the element focusable and places it in the keyboard's Tab navigation order.
- `onfocus="alert(1)"` runs JavaScript when the element receives focus.
- `hello` is the visible text inside the element.

The payload alone did not automatically trigger the alert. I needed a way to make the browser focus the element.

<p>
<img src="images/Screenshot 4.png" alt="Screenshot 4 - Testing the custom HTML element and onfocus handler">
</p>

### 5. Creating the Exploit on the Exploit Server

I used the exploit server to create a page containing a script that navigated the browser to the vulnerable application's search URL. The URL included my payload and ended with the fragment `#x`.

The important part was the URL fragment:

```text
#x
```

Because the injected element had `id="x"` and was focusable, navigating to the fragment could focus that element. When the element received focus, its `onfocus` handler executed.

The navigation script did not directly trigger the alert. Instead, it sent the browser to the crafted URL, where the fragment navigation provided the focus trigger.

<p>
<img src="images/Screenshot 5.png" alt="Screenshot 5 - Setting up the exploit server">
</p>

### 6. Confirming the XSS

Finally, I delivered the exploit and confirmed that the JavaScript alert appeared.

This demonstrated that I could use an allowed custom HTML element and an event handler to execute JavaScript despite the application's restrictions on standard HTML tags.

<p>
<img src="images/Screenshot 6.png" alt="Screenshot 6 - Successful JavaScript alert">
</p>

## Final Payload

```html
<xss id="x" tabindex="1" onfocus="alert(1)">hello</xss>
```

The payload is triggered by navigating to the vulnerable page with the `#x` fragment, which targets the focusable element.

## Key Takeaway

This lab taught me to investigate an XSS vulnerability step by step instead of relying on a familiar payload.

1. **Test the input:** Check how special characters are reflected.
2. **Find allowed tags:** Use Burp Suite Intruder to test candidate HTML tags.
3. **Test event handlers:** Check whether attributes survive filtering, then verify whether the browser executes them.
4. **Find a trigger:** An event handler needs the corresponding event to occur.
5. **Use browser behavior:** A URL fragment can target an element by its ID and, when the element is focusable, can trigger its focus event.

The main lesson is that blocking common HTML tags does not necessarily prevent XSS. An allowed custom element, a usable event handler, and a reliable way to trigger the event can still create an exploitable vulnerability.
