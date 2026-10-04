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
