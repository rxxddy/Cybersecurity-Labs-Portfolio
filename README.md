# Blind SQL Injection Writeup (Boolean-Based)

This repository section documents the successful exploitation of a **Blind SQL Injection** vulnerability (Boolean-based) found in the `TrackingId` cookie of a web application. The goal was to extract the password for the administrator user from the database.

---

## Vulnerability Summary

| Category | Type | Risk Level | Target |
| :--- | :--- | :--- | :--- |
| **Vulnerability** | Blind SQL Injection (Boolean-based) | Critical | `TrackingId` Cookie Parameter |
| **Goal** | Extract `password` from the `users` table for user 'administrator'. |
| **Database Response** | No direct data output or error messages were returned. The page included a "Welcome back" message only if the injected query returned any rows (i.e., the condition was TRUE). |

---

## Tools Used

* Burp Suite Community Edition (Repeater, Intruder - Sniper mode).
* Operating System: Kali Linux.

---

## Exploitation Methodology (Proof of Concept)

The attack required injecting a subquery into the `TrackingId` cookie and comparing its result to a known value. The indicator of a TRUE condition was a difference in the HTTP response body size (Response received: 49 vs 46).

### 1. Finding the Working Syntax

The working syntax was found to be:

Initial Working Syntax: `Dk0C4X2x5SPdmdPo' AND [CONDITION] --`

### 2. Determining Password Length

The `LENGTH()` function was used to determine the total number of characters.

* Payload Example: `Dk0C4X2x5SPdmdPo' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>18)='a`
* Result: The password length was determined to be **19 characters**.

### 3. Character Extraction using Intruder

Burp Intruder was used in **Sniper Attack** mode to automate the character guessing process. The `SUBSTRING()` function was employed, and a Simple List payload set (a-z, 0-9) was configured.

* **Extraction Payload Template:**
    ```
    Dk0C4X2x5SPdmdPo' AND (SELECT SUBSTRING(password, N, 1) FROM users WHERE username='administrator')='§C§
    ```
    (Where `N` is the character position, and `C` is the payload being tested).

---

## Final Result

The full password was successfully extracted by iterating the position **N** from 1 to 19.

**Extracted Administrator Password:** nl4czzbfq87hltowm0o8

**Proof:** Successfully logged in as the administrator user using the extracted password.

* [Screenshot of Burp Intruder results showing the TRUE condition for the correct character.]
* [Screenshot of the successful login confirmation page.]