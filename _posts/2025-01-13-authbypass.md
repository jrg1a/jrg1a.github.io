---
title: Bypassing Website Authentication
date: 2025-01-05 20:00:00 +0100
categories: [Red Team, Web Application Security]
tags: [authentication, brute force, username enumeration, logic flaws, cookie tampering, encoding, hashing]
authors: jrg1a
image:
  path: /assets/images/RedTeam/authb.png
---

# Bypassing Website Authentication

> Authentication is a critical part of web application security, but poorly implemented authentication mechanisms can often be bypassed, defeated, or broken. This guide explores common methods used by attackers to exploit authentication vulnerabilities, and demonstrates tools and techniques to identify and exploit these flaws.

---

## Username Enumeration

A common step when identifying authentication vulnerabilities is creating a list of valid usernames. Website error messages can be a valuable resource for this, as they may inadvertently disclose whether a username exists in the system.

### Example
If you try entering the username **admin** in a signup form, and the error message reads:
```plaintext
An account with this username already exists.
```
This error message confirms the existence of the username admin. Attackers can automate this process using tools like ffuf to identify other valid usernames.

### Automating Username Enumeration with ffuf
The following ffuf command uses a wordlist of common usernames to check if they exist on the target system:
```bash
ffuf -w /usr/share/wordlists/SecLists/Usernames/Names/names.txt -X POST -d "username=FUZZ&email=x&password=x&cpassword=x" -H "Content-Type: application/x-www-form-urlencoded" -u http://target-machine.com/customers/signup -mr "username already exists"
```
Explanation of Flags:

    -w: Specifies the wordlist to use for fuzzing.
    -X: Specifies the HTTP request method (e.g., POST).
    -d: Defines the data sent in the request, where FUZZ represents the fuzzing parameter.
    -H: Adds headers (e.g., "Content-Type").
    -u: Sets the target URL.
    -mr: Matches specific response text to validate results.

The ffuf tool comes pre-installed on AttackBox or can be downloaded from GitHub.

## Brute Force
Once a list of valid usernames is identified, it can be used to perform a brute force attack on the login page. This attack automates the process of testing common password combinations against each username.

**Example**
```bash
ffuf -w valid_usernames.txt:W1,/usr/share/wordlists/SecLists/Passwords/Common-Credentials/10-million-password-list-top-100.txt:W2 -X POST -d "username=W1&password=W2" -H "Content-Type: application/x-www-form-urlencoded" -u http://10.10.116.166/customers/login -fc 200
```
Command Details:

    W1 represents the list of usernames.
    W2 represents the list of passwords.
    -fc: Filters out responses with a specific HTTP status code (e.g., 200).

## Cookie Tampering
Cookies are often used for access control and session management. Examining and modifying cookie values can lead to unauthorized access or privilege escalation.

**Example**
Let's say we have these cookies:
```plaintext
Set-Cookie: logged_in=true; Max-Age=3600; Path=/
Set-Cookie: admin=false; Max-Age=3600; Path=/
```
We can modify the cookies to:
```plaintext
curl -H "Cookie: logged_in=true; admin=true" http://10.10.116.166/cookie-test
```
and try to escalate our privileges.

**Example**
Cookies can also be encoded in Base64. Here is an example of how that could look like: 
```plaintext
Set-Cookie: session=eyJpZCI6MSwiYWRtaW4iOmZhbHNlfQ==; Max-Age=3600; Path=/
```
If we decode this string, we get:
```plaintext
{"id":1,"admin":false}
```


## Logic Flaws

Logic flaws occur when the intended logical flow of an application can be bypassed, circumvented, or manipulated by an attacker. These flaws are common in poorly implemented authentication processes.

**Example: Case-Sensitivity in URL Checks

```php
if( url.substr(0,6) === '/admin') {
    // Code to check if user is admin
} else {
    // Show page to user
}
```

The above code checks if the URL starts with /admin, but it fails to account for case variations. A user accessing /adMin would bypass the authentication check.

## Testing for Authentication Bypass
If you want to test for authentication bypass, you can use tools such as:
- **Burp Suite**: Intruder module for brute force attacks and parameter tampering.
- **Hydra**: Automated brute force on various protocols.
- **ffuf**: Fuzz endpoint for valid usernames and passwords.
- **Mitmproxy**: Intercept and modify HTTP/HTTPS Traffic.
- **Curl**: Manually craft and test HTTP requests!

## Avoiding Authentication Bypass
Preventing authentication bypass vulnerabilities involves following core security principles and leveraging available resources for best practices:

1. **Secure Password Policies**  
   Use strong, unique passwords enforced by policies that require complexity and periodic updates.  
   👉 [NIST Password Guidelines](https://pages.nist.gov/800-63-3/)

2. **Multi-Factor Authentication (MFA)**  
   Add an extra layer of security by requiring multiple forms of verification.  
   👉 [Microsoft MFA Best Practices](https://learn.microsoft.com/en-us/azure/active-directory/authentication/concept-mfa-howitworks)

3. **Proper Session Management**  
   Secure session cookies using attributes like `HttpOnly`, `Secure`, and `SameSite`.  
   👉 [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)

4. **Rate Limiting and Account Lockout**  
   Implement throttling and temporary account locks after repeated failed login attempts.  
   👉 [OWASP Rate Limiting Guidelines](https://owasp.org/www-community/controls/Blocking_Brute_Force_Attacks)

5. **Secure Authentication Protocols**  
   Use trusted standards like OAuth 2.0 or OpenID Connect instead of custom solutions.  
   👉 [OAuth 2.0 Overview](https://oauth.net/2/)

6. **Regular Security Audits**  
   Perform code reviews, penetration testing, and automated scans to identify vulnerabilities.  
   👉 [OWASP Top 10](https://owasp.org/www-project-top-ten/)

7. **User Education**  
   Teach users how to spot phishing attempts and use password managers for better security.  
   👉 [StaySafeOnline by NCSA](https://staysafeonline.org/)

By following these practices and guidelines, you can reduce the risk of authentication bypass vulnerabilities and enhance the security of your web application.
