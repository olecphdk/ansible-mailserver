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
