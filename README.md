<p align="center">
  <img src="logo.svg" alt="FileLocker logo" width="180">
</p>

<h1 align="center">FileLocker</h1>

<p align="center">
  A small command-line tool that encrypts any file behind a password, and decrypts it again when you provide the same password.
</p>

---

## What it does

You give it a file and a password. It writes a new file with `.locked` appended to the name. The original bytes are no longer readable without the password. When you run `unlock`, it asks for the password and writes the original file back, byte-for-byte identical to before.

It works on any file type — PDFs, images, videos, archives, source code, executables. The script reads files as raw bytes, so the format does not matter.

## Installation

```bash
pip install cryptography
```

That is the only third-party dependency. Everything else is from the Python standard library.

Tested on Python 3.9 and newer.

## Usage

```bash
# Encrypt
python file_locker.py lock secret.pdf
# -> secret.pdf.locked

# Decrypt
python file_locker.py unlock secret.pdf.locked
# -> secret.pdf

# Custom output path
python file_locker.py lock report.docx -o /backup/report.encrypted
python file_locker.py unlock /backup/report.encrypted -o report.docx
```

You will be asked for the password interactively. The terminal does not echo the characters you type. When locking, it asks twice to catch typos, because a typo at lock time means the file is unrecoverable.

Exit codes: `0` on success, `1` on a recoverable error (file not found, wrong password, etc.), `130` if you cancel with Ctrl+C.

## How it works

The locked file has this layout:

```
[ "LOCKEDv1" magic header, 8 bytes ]
[ random salt,            16 bytes ]
[ Fernet ciphertext,         rest  ]
```

When you lock a file:

1. A fresh 16-byte salt is drawn from `os.urandom`, the OS-provided cryptographically secure random source.
2. The salt and your password are fed to PBKDF2-HMAC-SHA256 with 200,000 iterations. The output is a 32-byte key.
3. That key drives Fernet, which performs AES-128-CBC encryption and adds an HMAC-SHA256 authentication tag.
4. The magic header, salt, and Fernet output are written to disk.

When you unlock, the same steps run in reverse: read salt, derive the key from the password you typed, ask Fernet to decrypt. If the password is wrong, the HMAC check fails and Fernet raises `InvalidToken`, which the script reports as "wrong password or file is corrupted".

The salt being random per-file means that two files locked with the same password produce different keys and unrelated ciphertext. The salt being stored next to the ciphertext means you do not need a separate key file — only the password is secret.

## Dependencies

### `cryptography` (third-party)

The only external package. Maintained by the Python Cryptographic Authority (PyCA), which is the closest thing the Python ecosystem has to a "default" cryptography library. It is widely audited and used by major projects like Django and AWS CLI.

We use three pieces from it:

- **`Fernet`** — a high-level "recipe" for symmetric encryption. It bundles AES-128-CBC, a random IV, a timestamp, and an HMAC-SHA256 tag into a single output, with one call to `encrypt` and one to `decrypt`. The reason to use Fernet instead of calling AES primitives directly is that doing your own AEAD construction (encrypt-then-MAC, padding, IV handling) is exactly the part of cryptography most often gotten wrong. Fernet eliminates those mistakes by not letting you make them.
- **`PBKDF2HMAC`** — a key derivation function. Passwords typed by humans have far less entropy than a 256-bit key, so you cannot use them directly. PBKDF2 turns a password and salt into a key, and is deliberately slow (controlled by `iterations`) so that an attacker who steals a locked file cannot brute-force the password quickly even with a GPU farm. We use the HMAC-SHA256 variant.
- **`hashes.SHA256`** — the underlying hash function for PBKDF2. SHA-256 is standard, fast, and supported in hardware on every modern CPU.

### Standard library

- **`argparse`** — parses the `lock` / `unlock` subcommands and their flags. Used instead of hand-rolled `sys.argv` parsing because it gives you `--help` for free.
- **`base64`** — Fernet expects the key as URL-safe base64. PBKDF2 returns raw bytes, so we encode them before handing the key to Fernet. This is a serialization detail, not a security choice.
- **`getpass`** — reads the password from stdin without printing it back. Without this, your password ends up in your terminal scrollback and possibly in shell history if piped in.
- **`os`** — only used for `os.urandom(16)`, which is the canonical way to get cryptographically secure random bytes in Python. Do not be tempted to use the `random` module here; it is not designed for security and is predictable.
- **`sys`** — exit codes, and printing errors to stderr instead of stdout so that `python file_locker.py lock x.pdf 2>/dev/null` still works the way you would expect.
- **`pathlib.Path`** — used everywhere a filename appears. The reason for `Path` over plain strings is that `with_suffix`, `with_name`, and `is_file` are clearer and safer than string manipulation, especially across operating systems.

## Common issues

This is a small script, not a vault product. The points below are real limitations you should know about before relying on it.

### Whole file is loaded into memory

`open(...).read()` reads the entire file at once. Locking a 4 GB video on an 8 GB machine will swap heavily or fail. Fernet itself does not support streaming.

**Fix:** rewrite using a streaming AEAD such as `AES-GCM` from `cryptography.hazmat.primitives.ciphers.aead`, with a chunked frame format (one nonce per chunk, e.g. 64 KB). This is non-trivial because each chunk needs its own authenticated header. For typical documents (under a few hundred MB) the current code is fine.

### Original file is not removed after locking

`lock` writes a new `.locked` file but leaves the plaintext on disk. You have to delete it yourself. Worse, a normal delete on most filesystems just unlinks the entry — recovery tools can still find the data.

**Fix:** add a `--shred` flag. On HDDs, overwriting with random bytes a few times is enough. On SSDs it is not, due to wear-leveling and over-provisioning; the only reliable answer there is to encrypt the whole disk with BitLocker / LUKS / FileVault so that "deleted" data is also unreadable.

### PBKDF2 is not the strongest choice anymore

PBKDF2 is fine, but Argon2id is the current OWASP recommendation for password hashing because it is harder to attack on GPUs and ASICs. Our 200,000 iterations of PBKDF2-HMAC-SHA256 are also below OWASP's current 600,000 guidance.

**Fix:** swap `PBKDF2HMAC` for `argon2-cffi`, or at minimum bump `ITERATIONS` to 600,000. Higher numbers slow down legitimate unlocks too, so test the number you pick on the slowest machine you expect to use.

### No password strength check

The script accepts any non-empty password, including `1`. A short password is the weakest link in the whole system, regardless of how good the cipher is.

**Fix:** enforce a minimum length (12+ characters is reasonable), or pull in `zxcvbn-python` to score password strength and reject weak ones.

### Filename and file size leak

The output is `<original_name>.locked`, so anyone who sees the locked file knows what the original was called. The ciphertext is also only slightly larger than the plaintext, so file size is preserved within a few hundred bytes.

**Fix:** rename to a UUID before locking, and store the original filename inside the encrypted blob. For size, padding to fixed buckets (e.g. round up to the next megabyte) hides exact length at the cost of disk space.

### Forgetting the password means the file is gone

This is by design — if there were a recovery path, the encryption would be pointless — but people forget anyway. The script has no recovery hook, no hint storage, nothing.

**Fix:** save the password in a password manager (Bitwarden, 1Password, KeePassXC) at the same time you lock the file. Treat this as a habit, not an afterthought.

### One file at a time

There is no `--recursive` flag and no way to lock a folder in one call. You have to script the loop yourself.

**Fix:** in bash, `for f in *.pdf; do python file_locker.py lock "$f"; done`. In PowerShell, `Get-ChildItem *.pdf | ForEach-Object { python file_locker.py lock $_.Name }`. A proper recursive flag could be added in ~10 lines.

### Password lives in process memory

While the script runs, the password is a Python `str`. Python strings are immutable and the garbage collector does not zero them, so a memory dump of the process could recover the password until it is overwritten. There is no good fix in pure Python; if this matters to your threat model, you should be using full-disk encryption and a hardware security module rather than a 200-line script.

### Wrong password and corrupted file look the same

Both produce `InvalidToken` from Fernet, and the script reports "wrong password or file is corrupted" for both. There is no way to tell them apart from outside, because doing so would leak information to an attacker. This is the correct cryptographic behavior, just sometimes a confusing user experience.

## Quick test

```bash
echo "this is a secret" > note.txt
python file_locker.py lock note.txt        # enter a password twice
rm note.txt
python file_locker.py unlock note.txt.locked   # enter the same password
cat note.txt                               # "this is a secret"
```

If the unlock prints `[ERROR] Wrong password or file is corrupted.`, you typed it wrong; try again.
