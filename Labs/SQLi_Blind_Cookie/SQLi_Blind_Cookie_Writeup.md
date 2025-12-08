# Blind SQL Injection Writeup: Cookie Parameter

This report details the successful exploitation of a **Boolean-based Blind SQL Injection** vulnerability against a web application. The attack was performed by manipulating the `TrackingId` HTTP cookie to extract the administrator's password.

---

## Vulnerability Summary

| Category | Type | Risk Level | Target Parameter |
| :--- | :--- | :--- | :--- |
| **Vulnerability** | Blind SQL Injection (Boolean-based) | **Critical** | `TrackingId` Cookie |
| **Goal** | Extract `password` from the `users` database table for user 'administrator'. |
| **Success Indicator** | A difference in HTTP response body length (e.g., 49 bytes for TRUE vs. 46 bytes for FALSE) or the presence of the "Welcome back" message. |
| **Database Response** | No direct output or error messages were displayed. |

---

## Methodology and Exploitation

### 1. Initial Setup and Syntax Confirmation

The initial challenge was establishing the correct injection syntax to bypass the string context and comment out the rest of the query.

* **Original Tracking ID (Placeholder):** `RN1veNE5nFOnpJ1a`
* **Working Prefix:** The string was closed using a single quote (`'`), followed by the logical operator `AND`.
    * **Final Working Payload:** `PLACEHOLDER_ID' AND [CONDITION] --`

### 2. Payload Logic and Automation

The extraction relied on the following SQL functions to perform the attack:

* **Length Determination:** `LENGTH(password)`
    * **Result:** Password length was **19 characters**.
* **Character Extraction:** `SUBSTRING(password, N, 1)` (Extracts the N-th character).

The **Sniper Attack** mode in Burp Intruder was used to iterate over 36 possible characters (a-z, 0-9) for each of the 19 positions.

* **Extraction Payload Template:**
    ```sql
    PLACEHOLDER_ID' AND (SELECT SUBSTRING(password, N, 1) FROM users WHERE username='administrator')='§C§
    ```
    (Where `N` was manually incremented from 1 to 19, and `§C§` was the character payload marker.)

### 3. Final Result

The full password was successfully retrieved character by character.

* **Extracted Administrator Password:** **nl4czzbfq87hltowm0o8**
