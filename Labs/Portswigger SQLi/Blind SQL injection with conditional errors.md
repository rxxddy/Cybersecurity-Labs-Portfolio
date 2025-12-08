# Blind SQL Injection Writeup: Conditional Error (Oracle)

This report details the successful exploitation of a **Blind SQL Injection** vulnerability against an Oracle database. The attack was performed by manipulating the `TrackingId` HTTP cookie to force the application to return the administrator's password within a server-side error message.

---

## 🛡️ Vulnerability and Indicators

### Summary

| Category | Type | Risk Level | Target Parameter |
| :--- | :--- | :--- | :--- |
| **Vulnerability** | Blind SQL Injection (Error-based) | **Critical** | `TrackingId` Cookie |
| **Database** | **Oracle** (Confirmed via lab hint) |
| **Database Response** | No direct output or time delay. Server intercepts error details, but returns a generic **HTTP 500 Internal Server Error** upon any SQL failure. |

### Error-Based Logic Learned

In this scenario, traditional **Boolean-based** (Welcome Back/Length change) and **Time-based** (Sleep) attacks were ineffective due to strict server-side filtering. This required using a **Conditional Error** approach:

* **IF TRUE (Condition Met):** The query executes a command designed to **force a fatal error** (Division by Zero). Result: **HTTP 500 Internal Server Error**.
* **IF FALSE (Condition Not Met):** The query executes safely (NULL), resulting in a **200 OK** response.

This methodology required seeking the **500 Internal Server Error** response to confirm a matching character.

---

## 🛠️ Exploitation Methodology

### 1. Syntax Confirmation and Error Vector

The first step was confirming the working syntax to successfully execute arbitrary SQL, and finding a reliable Oracle function to generate a conditional error.

* **Working Prefix:** `hz2A7AgX8OyAxohu'`
* **Error Generation Function:** The expression `TO_CHAR(1/0)` was identified as a reliable method to generate a division-by-zero error in the Oracle environment.
* **Initial Test Payload (Confirmation):**
    ```sql
    hz2A7AgX8OyAxohu' AND 1=TO_CHAR('abc'/0) --
    ```
    This successfully returned a **500 Internal Server Error**, confirming the working syntax and error vector.

### 2. Payload Logic: Conditional Error Generation

The core payload used the `CASE` statement to conditionally trigger the error only when a character matched:

* **Function Used:** `SUBSTR(password, N, 1)` (Oracle equivalent of SUBSTRING).

* **Extraction Payload Template:**
    ```sql
    ' AND (SELECT CASE WHEN (SUBSTR(password, N, 1)='C') THEN TO_CHAR(1/0) ELSE NULL END FROM users WHERE username='administrator') IS NULL --
    ```

    1.  **If `SUBSTR(...)='C'` is TRUE:** The query attempts `TO_CHAR(1/0)`, causing the **500 Internal Server Error**.
    2.  **If `SUBSTR(...)='C'` is FALSE:** The query returns `NULL`, causing the **200 OK** status.

### 3. Automated Character Extraction

The attack was automated using **Burp Suite Intruder** (Sniper Attack mode) to iterate through 36 possible alphanumeric characters (a-z, 0-9) for each position ($N$).

* **Indicator of Success:** The desired character was confirmed upon receiving an **HTTP 500 Internal Server Error** status code.
* **Password Length:** Determined to be **19 characters** (through iteration).

### 4. Final Result

The full password was successfully retrieved by logging the Payload ($C$) that caused the 500 status code for each position ($N=1$ to $N=19$).

* **Extracted Administrator Password:** **6zdk4cxbyeju3qmu4cjo**