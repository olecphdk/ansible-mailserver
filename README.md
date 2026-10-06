# ansible-mailserver

Infrastructure-as-Code for the personal mail server on AlmaLinux using Postfix, Dovecot, OpenDKIM, Fail2ban and Let's Encrypt.

## Architecture

- Server hostname, PTR and TLS endpoint: `slushice.femto.dk`
- Public IPv4: `135.181.106.207`
- Primary mail domain: `femto.dk`
- DKIM selector: `mail2025`
- Mail storage: Dovecot Maildir
- SMTP: Postfix
- IMAP: Dovecot
- TLS: Let's Encrypt

`slushice.femto.dk` is the infrastructure hostname. User-facing addresses use `@femto.dk`.

## System updates

Normal configuration runs do not upgrade all AlmaLinux packages. The full package update task is tagged with both `never` and `updates`, so it only runs when explicitly requested.

Run system updates with:

```bash
ansible-playbook playbooks/setup-mailserver.yml --tags updates
```

A normal configuration run remains:

```bash
ansible-playbook playbooks/setup-mailserver.yml
```

This keeps OS patching separate from ordinary mail configuration changes, making failures and regressions easier to attribute.

## Fail2ban

Fail2ban uses the systemd journal backend on AlmaLinux. The jail configuration therefore does not set file-based `logpath` values.

The common policy is controlled by:

```yaml
fail2ban_bantime: "1h"
fail2ban_findtime: "10m"
fail2ban_maxretry: 6
```

These values are applied through the jail `[DEFAULT]` section to SSH, Postfix, Postfix SASL and Dovecot.

Verify after deployment with:

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
sudo fail2ban-client status postfix-sasl
```

## Dovecot listeners and authentication

Dovecot uses a separate mail password instead of the Linux/PAM password. Authentication is split deliberately:

```text
passdb -> /etc/dovecot/users
userdb -> system passwd database
```

The passwd-file contains only the mail authentication credential. Dovecot still obtains UID, GID and home directory from the Unix account, so mail remains in `/home/<dovecot_user>/Maildir`.

The account is configured once through `dovecot_user` in `group_vars/all.yml`. By default it follows `ansible_user`, so the role contains no hard-coded personal username:

```yaml
dovecot_user: "{{ ansible_user }}"
```

The same value is currently used as both Dovecot login name and Unix account. `auth_username_format = %n` means a client-supplied domain is stripped before lookup; tightening this to a full mail address is a separate change.

The password hash is not stored in Git. On the initial migration, generate a new mail-only password hash on slushice:

```bash
sudo doveadm pw
```

Copy the complete result, including its `{SCHEME}` prefix, into an encrypted vars file on the Ansible controller:

```bash
mkdir -p ~/.config/ansible-mailserver
ansible-vault create ~/.config/ansible-mailserver/mail-secrets.yml
```

The encrypted file should contain:

```yaml
dovecot_mail_password_hash: '{SCHEME}...'
```

Run the first migration with:

```bash
ansible-playbook playbooks/setup-mailserver.yml \
  --ask-vault-pass \
  --extra-vars "@$HOME/.config/ansible-mailserver/mail-secrets.yml"
```

After `/etc/dovecot/users` exists, ordinary playbook runs do not require the secret file and leave the existing password hash unchanged. To rotate the mail password, generate a new hash and rerun with the external vars file.

Verify authentication on the server:

```bash
sudo doveadm auth test <dovecot_user>
sudo doveconf -n | grep -A8 -E '^(passdb|userdb|auth_username_format|protocols)'
```

The effective authentication configuration should contain one `passwd-file` passdb and one system `passwd` userdb, with no PAM passdb.

Dovecot is restricted to IMAP plus LMTP:

```text
protocols = imap lmtp
```

POP3 is disabled, and the plaintext IMAP listener on port 143 is disabled. Client IMAP is exposed only as implicit TLS on port 993. LMTP remains available through the Unix socket used by Postfix.

Verify listeners:

```bash
sudo ss -ltnp | grep -E ':(110|143|993|995)\\b'
```

Expected TCP listener:

```text
993
```

After changing the Dovecot password, update both incoming IMAP and outgoing SMTP authentication in K-9 and test ports 993 and 587. The Unix password must not be locked until both tests pass.

Once both mail tests succeed, lock the Unix password for the Dovecot account:

```bash
sudo passwd -l <dovecot_user>
sudo passwd -S <dovecot_user>
```

The status should show `L` for the account.

Keep the current SSH session open and verify a fresh SSH login from the Ansible controller using the configured SSH key. For example:

```bash
ssh -i ~/.ssh/id_ed25519 <dovecot_user>@<mail-server>
```

Finally, repeat one IMAP sync and one SMTP submission from the mail client. This confirms that mail authentication is independent of the now-locked Unix password.

## TLS certificates

TLS certificate identity is kept separate from the server/mail-domain variables:

```yaml
mail_domain: "slushice.femto.dk"
tls_cert_name: "slushice.femto.dk"
letsencrypt_domains:
  - "slushice.femto.dk"
```

Certbot uses `tls_cert_name` as the certificate lineage name and requests every name listed in `letsencrypt_domains`. Postfix and Dovecot both read the certificate from:

```text
/etc/letsencrypt/live/<tls_cert_name>/
```

This allows certificate names/SANs to change independently of the Postfix hostname configuration.

## SMTP services

Public SMTP on port 25 is for server-to-server mail delivery and does not offer SMTP AUTH.

Client submission uses port 587 with mandatory TLS and Dovecot SASL authentication:

```text
25  -> SMTP delivery, no AUTH
587 -> STARTTLS + AUTH for mail clients
993 -> TLS + Dovecot authentication for IMAP
```

The submission service explicitly enables `smtpd_sasl_auth_enable=yes` in `master.cf`, while the global Postfix setting keeps AUTH disabled by default.

After deployment, verify that port 25 does not advertise AUTH:

```bash
printf 'EHLO test\r\nQUIT\r\n' | nc localhost 25
```

The response should not contain an `AUTH` capability.

## Migration status

The server is being tested in parallel with Proton Mail. Proton remains the production MX for `femto.dk`; changing MX is a separate cutover step and is intentionally not automated here.

Current SPF desired state while both systems may send mail:

```text
v=spf1 ip4:135.181.106.207 include:_spf.protonmail.ch mx ~all
```

Current DMARC policy:

```text
v=DMARC1; p=none
```

Proton's DKIM CNAME records remain in DNS alongside the independent slushice selector `mail2025._domainkey.femto.dk`.

## DKIM

OpenDKIM's `-h` / `--hash-algorithms` option expects a hash algorithm such as `sha256`, not the DKIM signing algorithm name `rsa-sha256`.

Generate keys with:

```text
opendkim-genkey -b 2048 -h sha256 -r -s mail2025 -d femto.dk
```

The resulting DNS TXT record should contain `h=sha256`. The mail signature itself will use `a=rsa-sha256`.

Verify published DKIM:

```bash
dig +short TXT mail2025._domainkey.femto.dk
opendkim-testkey -d femto.dk -s mail2025 -vvv
```

Expected result from `opendkim-testkey` is `key OK`. A `key not secure` warning refers to DNSSEC validation and does not mean the DKIM key failed.

## DNS verification

Check authoritative DNS directly:

```bash
dig @ns01.one.com femto.dk TXT +noall +answer
dig @ns02.one.com femto.dk TXT +noall +answer
dig @ns01.one.com mail2025._domainkey.femto.dk TXT +noall +answer
dig @ns02.one.com mail2025._domainkey.femto.dk TXT +noall +answer
```

DNS TTL is currently 3600 seconds, so recipient systems may continue to use an older cached SPF or DKIM record for up to about an hour after a change.

## Outbound smoke test

Before changing MX, verify outbound delivery directly from slushice:

```bash
nc localhost 25 <<EOF
EHLO slushice.femto.dk
MAIL FROM:<user@femto.dk>
RCPT TO:<external-test-address>
DATA
From: Test User <user@femto.dk>
To: <external-test-address>
Subject: slushice outbound test

Test from slushice.
.
QUIT
EOF
```

Inspect the received message's `Authentication-Results`. Before production cutover the target is:

```text
spf=pass
dkim=pass
dmarc=pass
```

Do not change the `femto.dk` MX from Proton to slushice until outbound authentication and inbound delivery have both been verified.
