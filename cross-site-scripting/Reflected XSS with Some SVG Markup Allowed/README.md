\# Reflected XSS with Some SVG Markup Allowed



\## Lab Description



This lab contains a reflected Cross-Site Scripting (XSS) vulnerability. The application blocks many common HTML tags but allows some SVG markup and event handlers.



My goal was to find an allowed tag and event combination, understand how the browser interpreted my input, and trigger a JavaScript `alert()`.



\## Solution



\### 1. Testing the Search Input



I started by searching for `test` and then tested special characters such as `<`, `>`, `"`, and `'`.



My input appeared in the search results heading without those characters being visibly encoded. Next, I tried a `<script>` tag and received a message saying that the tag was not allowed.



This suggested that the application reflected my input but also applied filtering to certain HTML tags.



<p>

&#x20; <img src="images/Screenshot 1.png" alt="Screenshot 1 - Testing special characters and the script tag">

</p>



\### 2. Finding Allowed HTML Tags



I used Burp Suite Intruder to test different HTML tags and check which ones the application accepted.



Some tags returned a `200` status code, including `svg` and `animatetransform`. I checked the responses to identify which tags were preserved instead of relying only on the status code.



<p>

&#x20; <img src="images/Screenshot 2.png" alt="Screenshot 2 - Finding allowed SVG tags with Burp Suite Intruder">

</p>



\### 3. Finding an Allowed Event Handler



After identifying allowed tags, I tested different event handler attributes using Intruder.



I found that `onbegin` returned a `200` status code in my tests. This gave me a possible event handler to investigate further.



However, an allowed event handler does not automatically execute JavaScript. The corresponding event must actually occur.



<p>

&#x20; <img src="images/Screenshot 3.png" alt="Screenshot 3 - Testing event handlers with Burp Suite Intruder">

</p>



\### 4. Testing the Animation Element



I initially tried using `animatetransform` with the `onbegin` event handler, expecting it to trigger an alert.



```html

<animatetransform onbegin="alert(1)">test</animatetransform>

```



Although the markup appeared in the response, I did not get an alert.



My mistake was assuming that the animation element would work by itself. I needed to understand what kind of context the browser required to interpret it as an SVG animation element.



<p>

&#x20; <img src="images/Screenshot 4.png" alt="Screenshot 4 - Testing animateTransform with the onbegin event">

</p>



\### 5. Investigating the HTML Structure



While testing my input, I noticed that the search form disappeared from the rendered page.



This made me investigate how my input affected the surrounding HTML. I suspected that I might be changing the way the browser interpreted the existing markup.



<p>

&#x20; <img src="images/Screenshot 5.png" alt="Screenshot 5 - Observing the changed page structure">

</p>



\### 6. Confirming the Context Breakout



I inspected the response source and investigated how my input was inserted into the search results heading.



I discovered that I could use the characters `">` to change how the browser parsed the surrounding markup in this context.



This was an important step: instead of only inserting a tag inside the existing text, I could escape the surrounding context and introduce new markup.



<p>

&#x20; <img src="images/Screenshot 6.png" alt="Screenshot 6 - Inspecting the response source and HTML context">

</p>



\### 7. Introducing SVG Markup



Next, I combined the context breakout with an SVG element.



```html

"><svg>

```



Think of `<svg>` as a box that provides the correct environment for SVG elements. The `animateTransform` element defines a transformation animation, and its `onbegin` event handler can execute JavaScript when that animation begins.



This helped me understand why finding an allowed tag and event handler was not enough: I also needed the browser to interpret them in the correct context.



<p>

&#x20; <img src="images/Screenshot 7.png" alt="Screenshot 7 - Testing the SVG markup in the reflected input">

</p>



\### 8. Triggering the JavaScript Alert



Finally, I combined the context breakout, the SVG element, the animation element, and the event handler.



My final payload was:



```html

"><svg><animatetransform onbegin=alert(1)>

```



The browser interpreted the injected markup as SVG content, and the animation's `onbegin` event triggered the JavaScript alert.



<p>

&#x20; <img src="images/Screenshot 8.png" alt="Screenshot 8 - Successful JavaScript alert">

</p>



\## Final Payload



```html

"><svg><animatetransform onbegin=alert(1)>

```



\## Key Takeaway



This lab taught me that finding an allowed HTML tag is only one part of discovering an XSS vulnerability.



1\. \*\*Test the input:\*\* Observe how special characters are reflected.

2\. \*\*Identify the filtering rules:\*\* Use Burp Suite Intruder to test which tags and event handlers are accepted.

3\. \*\*Understand the HTML context:\*\* Determine how the browser interprets the injected input.

4\. \*\*Use the correct environment:\*\* SVG animation elements need to be interpreted in an appropriate SVG context.

5\. \*\*Find a trigger:\*\* An event handler only executes when its corresponding event occurs.



The biggest lesson was to stop guessing payloads and start understanding the browser's behavior. By investigating the filtering, HTML context, SVG markup, and event trigger separately, I was able to understand how the XSS worked.

