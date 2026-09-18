# DefCamp-CTF-aiscrimination-Write-up

# CTF Write-up: AIscrimination — Escaping the Parser via CSS Preprocessor Injection (LFI)

**Vulnerability Type:** Server-Side CSS Injection / Local File Inclusion (LFI)  
**Severity:** Critical (CVSS 3.1: 9.8)

In this write-up, we'll dive into an interesting web challenge called AIscrimination. What initially looked like a standard input reflection or potential template injection turned out to be a much cooler attack vector: breaking out of a CSS string to force a server-side preprocessor into reading arbitrary local files.

Here is the step-by-step breakdown of the attack chain.

## 1. Reconnaissance & Target Mapping
The target application generates a public "inclusion card" based on user input. Initial DOM analysis revealed that the "Inclusion statement" parameter was not rendered directly in the HTML markup. Instead, a span with the class `imported-fragment` was returned empty, but a dynamically generated stylesheet was linked in the header:

```html
<link rel="stylesheet" href="/assets/cards/[hash]/identity.css">
```

Furthermore, the UI noted that the card was "compiled from locally imported design fragments." This behavior and phrasing heavily indicated that the backend was utilizing a CSS Preprocessor (likely LESS or SASS) to dynamically compile stylesheets based on user input.

## 2. Vulnerability Identification (CSS String Breakout)
By analyzing the dynamically generated `identity.css` file via an HTTP proxy (Burp Suite), the exact injection point was discovered. It resided inside the `content` property of a pseudo-element:

```css
.identity-card::after {
  content: "USER_INPUT_HERE";
}
```

The backend failed to sanitize or escape double quotes (`"`), allowing for a CSS String Breakout. By injecting `"; }`, it was possible to terminate the `content` property and close the `.identity-card` CSS block entirely. This granted the ability to inject arbitrary CSS rules or, more importantly, Preprocessor directives into the compilation pipeline.

## 3. Exploitation (The Kill Chain)
To escalate this injection into an Arbitrary File Read (LFI), I leveraged the LESS `@import` directive, specifically `@import (inline)`. This directive forces the compiler to read a local file and embed its raw contents directly into the final CSS output.

To prevent the compiler from crashing and throwing a 500 Internal Server Error, a dummy CSS class was appended to balance the remaining syntax in the backend template.

### Step 1: Architecture Mapping via `/etc/passwd`
Initial attempts to read files using absolute paths resulted in a `/* design fragment unavailable */` error, indicating the compiler either blocked absolute paths or couldn't locate them. A deep path traversal payload was used to traverse up to the system root:

**Payload:**
```css
"; } @import (inline) "../../../../../../../../etc/passwd"; .dummy { content: "
```

The compiled CSS successfully leaked the contents of `/etc/passwd`. Because the output is rendered in CSS, newlines were hex-escaped as `\A`. This leaked data confirmed the directory traversal worked and revealed the system layout, including a `ctf` user account.

### Step 2: Flag Exfiltration
Following the system mapping, the payload was adjusted to target the specific challenge flag located in the root directory.

**Payload:**
```css
"; } @import (inline) "../../../../../../../../flag.txt"; .dummy { content: "
```

**Response Output (`identity.css`):**
```css
#card-3e9d663c3a9d4fbbb4394dc5858a785d .imported-fragment::after {
  content: "CTF{readcted}\A ";
  display: inline-block;
  margin-top: .7rem;
  color: #d8f96e;
  font: 500 .74rem 'DM Mono', monospace;
  letter-spacing: .02em;
}
 .dummy { content: "";
  display: block;
  color: #d9d4ff;
  font-size: .93rem;
  line-height: 1.45;
  margin-top: .65rem;
}
```
*Flag successfully exfiltrated!*

