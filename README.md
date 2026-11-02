# Neomutt + Gmail Setup (Direct IMAP, No Local Mail Storage)

**Purpose**: Set up neomutt for reading/sending kernel-dev mail via Gmail,
without storing a local copy of mail on disk. Uses msmtp for sending
and gpg for securely storing the Gmail App Password.

> NOTE: This guide is for `Ubuntu/Debian` users. But this works on `Arch Linux` with a few adjustments (you need to do it yourself btw~).

## Step 1 - Install Packages

```bash
   $ sudo apt update
   $ sudo apt install neomutt isync msmtp msmtp-mta pass gnupg
```
> NOTE: If prompted "Enable AppArmor support?" during install, choose YES.
> It only adds a security confinement profile for the package; no downside.

## Step 2 - Generate a Gmail App Password 

1. Go to `Google Account` -> `Google Account`
2. Confirm `2-Step Verification` is ON (required)
3. Go to `Security` -> `App Passwords` (or [myaccount.google.com/apppasswords](myaccount.google.com/apppasswords))
4. Create a new app password, name it e.g. "`neomutt`"
5. Copy the 16-character password shown (no spaces when using it)
   Keep this handy for `Step 3`.

## Step 3 - Encrypt the App Password with GPG

3a. Generate a GPG key (skip if one already exists):
    gpg --full-generate-key
    - Key type: `RSA and RSA` (default)
    - Key size: `4096`
    - Expiry: `0` (does not expire) or your preference
    - Real name: Enter your name
    - Email: Your Gmail address
    - Set a passphrase (you will need this later to decrypt)

3b. Save the app password to a temp file:
```bash
   $ echo "<your_16_char_app_password>" > mbsync.pw
```

3c. Encrypt it:
```bash
   $ gpg --encrypt --recipient "your.email@gmail.com" mbsync.pw
```

3d. Move into place and delete the plaintext:
```bash
   $ mv mbsync.pw.gpg ~/.mbsync-pw.gpg
   $ rm mbsync.pw
```

3e. Verify decryption works:
```bash
   $ gpg --quiet --for-your-eyes-only --no-tty --decrypt ~/.mbsync-pw.gpg
```
> NOTE: Should prompt for GPG passphrase, then print the app password

## Step 4 - Configure Neomutt (Direct IMAP, No Local Storage)

4a. Create config directory:
```bash
   $ mkdir -p ~/.config/neomutt
```

4b. Create `~/.config/neomutt/neomuttrc` with the following content:
```
    # --- Identity ---
    set realname = "Your Name"
    set from = "your.email@gmail.com"
    set envelope_from = yes
    set use_from = yes

    # --- IMAP connection (direct, nothing stored locally) ---
    set folder = "imaps://your.email@gmail.com@imap.gmail.com/"
    set spoolfile = "+INBOX"
    set imap_user = "your.email@gmail.com"
    set imap_pass = "`gpg --quiet --for-your-eyes-only --no-tty --decrypt ~/.mbsync-pw.gpg`"
    set ssl_starttls = yes
    set ssl_force_tls = yes

    # --- No local caching of headers or messages ---
    unset header_cache
    unset message_cachedir

    # --- General behavior ---
    set mail_check = 60
    set timeout = 15
    set editor = "nvim"
    set edit_headers = yes
    set sig_dashes = no
    set sort = threads
    set sort_aux = last-date-received

    # --- Sending via msmtp (configured in Step 5) ---
    set sendmail = "/usr/bin/msmtp"
```

4c. Launch neomutt:
```bash
   $ neomutt
```
> - Prompts for GPG passphrase to decrypt app password
> - Connects live to imap.gmail.com and shows INBOX

4d. Check other folders (optional):
    Press 'l' inside neomutt, then type e.g.:
```
    imaps://your.email@gmail.com@imap.gmail.com/[Gmail]/Sent Mail
```

## Step 5 - Configure MSMTP (for Sending Mail)

5a. Create `~/.msmtprc` with the following content:
```
    defaults
    tls on
    tls_trust_file /etc/ssl/certs/ca-certificates.crt
    logfile ~/.msmtp.log

    account gmail
    host smtp.gmail.com
    port 587
    auth on
    tls_starttls on
    from your.email@gmail.com
    user your.email@gmail.com
    passwordeval "gpg --quiet --for-your-eyes-only --no-tty --decrypt ~/.mbsync-pw.gpg"

    account default : gmail
```

5b. Lock down file permissions:
```bash
   $ chmod 600 ~/.msmtprc
```

5c. Test sending:
```bash
   $ echo -e "Subject: test\n\nhello from msmtp" | msmtp your.email@gmail.com
```
> NOTE:
> - Prompts for GPG passphrase
> - No output = success
> - Check inbox for the test email

5d. Check logs if it fails:
```bash
   $ cat ~/.msmtp.log
```

---

**Result / What this setup gives you?**
- neomutt reads mail directly from Gmail via IMAP, no local mail files saved
- msmtp sends mail through Gmail's SMTP using the same encrypted app password
- Gmail App Password stored once, encrypted with GPG, reused by both tools
- Ready to link with git send-email (separate step) using the same msmtp account

**Notes**
- Every folder switch/search in neomutt round-trips to Gmail over network
  (slower than a local Maildir cache, but zero mail stored on disk).
- If performance becomes an issue with high-volume lists (e.g. LKML),
  mbsync + local Maildir can be added later as a caching layer.
