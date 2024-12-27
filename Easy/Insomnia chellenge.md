# Insomnia Challenge: Source Code Analysis and Exploitation

## Overview

This challenge involved bypassing a flawed registration system to access the `administrator` profile and retrieve the flag.

---

## Step-by-Step Analysis

### 1. Initial SQLi Testing

The challenge presented a login form, and the initial approach was to test for **SQL Injection (SQLi)** vulnerabilities. However, analyzing the source code revealed that no SQL queries were used in the authentication process.

---

### 2. Key Files

The challenge source code included two critical files:

#### **`ProfileController.php`**

The important section in this file:

```php
if ($username == "administrator") {
    return view("ProfilePage", [
        "username" => $username,
        "content" => $flag,
    ]);
}
```

**Observation**:

- The flag is displayed on the profile page when the logged-in username is `administrator`.

#### **`UserController.php`**

The critical section in this file:

```php
if (!count($json_data) == 2) {
    return $this->respond("Please provide username and password", 404);
}
```

**Observation**:

- The `if` statement incorrectly validates the JSON payload.
- It uses `!count($json_data) == 2`, which evaluates incorrectly, allowing for unexpected behavior when fewer than two keys are sent in the JSON payload.

---

## Exploitation

### Registration Behavior

The registration system expects a JSON payload with `username` and `password`:

```json
{
    "username": "test",
    "password": "test"
}
```

However, due to the flawed validation logic in `UserController.php`, sending a single key bypasses the intended validation.

### Exploit Payload

To exploit this vulnerability and register as `administrator`, the following payload was sent:

```json
{
    "username": "administrator"
}
```

### Steps to Exploit

1. Use a tool like **Postman** or **cURL** to send the payload to the registration endpoint.
2. Log in as `administrator` using the username from the payload (no password required).
3. Access the profile page to retrieve the flag.

---

## Key Lessons

### Takeaways

1. **Validation Logic**:
   
   - Always validate inputs rigorously.
   - Use strict equality checks (`!==`) for robust validation.

2. **Code Review Importance**:
   
   - Conduct a thorough source code review before testing complex attack vectors.

3. **JSON Payload Validation**:
   
   - Ensure JSON payloads are parsed and validated properly to prevent logical bypasses.

### Mitigation Strategies

1. Correct the flawed logic:
   
   ```php
   if (count($json_data) !== 2) {
      return $this->respond("Please provide username and password", 404);
   }
   ```

2. Implement stricter checks for required fields in the JSON payload.

3. Conduct rigorous testing for edge cases in input validation.

---

## Skills Gained

1. Identifying and exploiting **input validation flaws**.
2. Understanding **JSON payload validation** logic.
3. Recognizing the importance of **code reviews** in vulnerability analysis.
