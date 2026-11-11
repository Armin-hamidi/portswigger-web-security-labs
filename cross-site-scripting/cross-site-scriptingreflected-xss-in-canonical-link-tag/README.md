# Reflected XSS in Canonical Link Tag

## Lab Description

This lab reflects user input inside a canonical link tag and escapes angle brackets.

The goal was to perform a reflected XSS attack by injecting an additional HTML attribute that calls the `alert()` function.

The interesting part of this lab is that I couldn't inject a new HTML tag because angle brackets were encoded. Instead, I had to find a way to inject new attributes into an existing HTML element.

## Solution

### 1. Checking the normal page

I started by opening the lab and looking at the normal home page.

<p>
  <img src="images/Screenshot 1.png" alt="Screenshot 1 - Normal lab home page">
</p>

### 2. Inspecting the page source

I opened the page source and searched for the canonical link tag.

I found this HTML:

```html
<link rel="canonical" href='https://0af900e603bad6d1803f3a3e00a900b7.web-security-academy.net'/>
```

I noticed that the `href` attribute was surrounded by **single quotes**.

This was important because it gave me a place to investigate whether I could escape from the existing attribute.

<p>
  <img src="images/Screenshot 2.png" alt="Screenshot 2 - Canonical link in page source">
</p>

### 3. Checking whether my input was reflected

I added `?test123` to the URL and inspected the page source again.

I found that my input appeared inside the canonical link's `href` attribute.

This confirmed that the application reflected my input into the HTML response.

<p>
  <img src="images/Screenshot 3.png" alt="Screenshot 3 - Input reflected inside canonical href">
</p>

### 4. Testing the characters

Next, I tested different characters:

```text
?test123 ' test234 > test345
```

I noticed two important things:

- The single quote was not encoded, so I could potentially close the existing `href` value.
- The angle bracket `>` was URL-encoded as `%3e`.

This meant that injecting a completely new HTML tag was not the approach I needed.

However, the unencoded single quote gave me another possibility: injecting additional attributes into the existing `<link>` element.

<p>
  <img src="images/Screenshot 4.png" alt="Screenshot 4 - Testing special characters">
</p>

### 5. Testing attribute injection

I wanted to confirm whether I could create a new HTML attribute.

I tried:

```text
?test123'x='y
```

The resulting HTML showed:

```html
<link rel="canonical" href='https://0af900e603bad6d1803f3a3e00a900b7.web-security-academy.net/?test123'x='y'/>
```

This showed that:

- `x` was interpreted as a new attribute name.
- `y` was its attribute value.

I had successfully found a way to break out of the existing `href` attribute and introduce another attribute without creating a new HTML element.

<p>
  <img src="images/Screenshot 5.png" alt="Screenshot 5 - Testing a new HTML attribute">
</p>

### 6. Injecting accesskey and onclick

Now I needed a way to execute JavaScript.

The lab description mentioned that the simulated user would press one of several keyboard combinations. This was a clue to investigate the HTML `accesskey` attribute.

I learned that:

- `accesskey='x'` assigns the X key as an access key.
- `onclick='alert(1)'` executes JavaScript when the element is activated.

I also considered other event handlers:

- `onkeydown` requires the element to receive keyboard focus.
- `onfocus` requires the element to receive focus.
- `ontrigger` is not a standard HTML event handler.
- `onclick` was suitable because activating the access key can trigger the element's click behavior in this lab.

I used the following input:

```text
?test123'accesskey='x'onclick='alert(1)
```

The resulting HTML was:

```html
<link rel="canonical" href='https://0af900e603bad6d1803f3a3e00a900b7.web-security-academy.net/?test123'accesskey='x'onclick='alert(1)'/>
```

Now the existing canonical link contained the two attributes I needed.

<p>
  <img src="images/Screenshot 6.png" alt="Screenshot 6 - Injecting accesskey and onclick">
</p>

### 7. Triggering the XSS

Since I assigned `x` as the access key, I pressed the appropriate keyboard combination:

```text
Alt + X
```

The browser activated the element, triggering the `onclick` event handler.

The JavaScript executed successfully, and the alert appeared.

This completed the lab.

<p>
  <img src="images/Screenshot 7.png" alt="Screenshot 7 - Alert successfully triggered">
</p>

## Final Payload

```text
?test123'accesskey='x'onclick='alert(1)
```

## Key Takeaway

This lab taught me that HTML-encoding angle brackets does not automatically prevent XSS.

Even when I cannot inject a new HTML tag, I may still be able to escape from an existing attribute and inject new attributes.

I also learned how `accesskey` can be combined with an event handler to trigger JavaScript through keyboard interaction.

**Always identify the reflection context first. Understanding where input is reflected and how the browser interprets it is essential when finding XSS vulnerabilities.**
