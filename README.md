# FileLocker
Single-file Python CLI that encrypts files with a password. AES-128-CBC + HMAC-SHA256 via Fernet, PBKDF2-HMAC-SHA256 key derivation, random per-file salt. ~175 lines, one external dependency.
