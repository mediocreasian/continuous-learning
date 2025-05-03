# Docker Mailserver + Cloudflare DNS Full Setup Documentation

---

# 1. VPS Host Environment

| Item              | Details                   |
| :---------------- | :------------------------ |
| VPS IP            | `<YOUR_IP>`               |
| Domain            | `exeltan.com`             |
| Reverse Proxy     | Traefik (already running) |
| Docker Mailserver | Deployed manually         |

> VPS running Debian 12 (Bookworm) with Docker and Docker Compose installed.

---

# 2. Docker-Mailserver Setup

## Directory Structure

```bash
~/docker-mail-server
|-- docker-compose.yml
|-- mailserver.env
|-- config/    # Mounted to /tmp/docker-mailserver
     |-- postfix-accounts.cf
     |-- opendkim/keys/exeltan.com/mail.private
     |-- opendkim/keys/exeltan.com/mail.txt
```

## docker-compose.yml

```yaml
services:
  mailserver:
    image: mailserver/docker-mailserver:latest
    container_name: mailserver
    hostname: mail
    domainname: exeltan.com
    env_file: mailserver.env
    ports:
      - "587:587"
    volumes:
      - ./config:/tmp/docker-mailserver/
      - /etc/letsencrypt:/etc/letsencrypt:ro
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
```

>  Note: Only `config/` is mounted now (no separate `data/`).

## mailserver.env

```properties
DOMAINNAME=exeltan.com
ENABLE_CLAMAV=0
ENABLE_FAIL2BAN=1
ENABLE_POSTGREY=0
ONE_DIR=1
PERMIT_DOCKER=network
ACCOUNTS=noreply@exeltan.com|<YOUR_PASSWORD>
ENABLE_TLS=1
TLS_FLAVOR=manual
SSL_TYPE=manual
SSL_CERT_PATH=/etc/letsencrypt/live/mail.exeltan.com/fullchain.pem
SSL_KEY_PATH=/etc/letsencrypt/live/mail.exeltan.com/privkey.pem
```

---

# 3. SSL Certificates (Manual DNS Challenge)

Manual DNS-01 validation used:

```bash
sudo certbot certonly --manual --preferred-challenges dns -d mail.exeltan.com
```

During certificate request, Certbot prompted to create a DNS TXT Record:

| Type | Name                   | Content                        | TTL  | Proxy    |
| :--- | :--------------------- | :----------------------------- | :--- | :------- |
| TXT  | `_acme-challenge.mail` | `<random_string_from_certbot>` | Auto | DNS Only |

Once validated, certbot issued:

* `/etc/letsencrypt/live/mail.exeltan.com/fullchain.pem`
* `/etc/letsencrypt/live/mail.exeltan.com/privkey.pem`

Mounted into container as read-only.

> Note: Certificate must be manually renewed every 90 days unless automated.

---

# 4. Cloudflare DNS Settings

| Type | Name              | Content                          | TTL             | Proxy    |
| :--- | :---------------- | :------------------------------- | :-------------- | :------- |
| A    | `mail`            | `<YOUR_IP>`                      | Auto            | DNS Only |
| MX   | `@`               | `mail.exeltan.com`               | Auto            | DNS Only |
| TXT  | `@`               | `v=spf1 ip4:<YOUR_IP> -all`      | Auto            | DNS Only |
| TXT  | `mail._domainkey` | (DKIM Public Key - see below)    | Auto            | DNS Only |
| TXT  | `_dmarc`          | *(Optional — Not Yet Added)*     | *(Recommended)* | DNS Only |

---

# 5. DKIM Setup

## Generated DKIM Key

Located at:

```bash
config/opendkim/keys/exeltan.com/mail.txt
```

Contents:

```text
mail._domainkey IN TXT (
  "v=DKIM1; h=sha256; k=rsa; p=<SOMETHING SOMETHING SOMETHING>"
  "<SOMETHING SOMETHING SOMETHING>"
)
```

Formatted into one line when inserting into Cloudflare.

---

# 6. Mail User Setup

Created inside the container manually:

```bash
docker exec -it mailserver addmailuser noreply@exeltan.com
```

Password: same as in `ACCOUNTS` variable.

---

# 7. Manual Email Sending Test

Using `swaks`:

```bash
swaks --to <YOUR_EMAIL> \
  --from noreply@YOUR_DNS.com \
  --server mail.YOUR_DNS.com \
  --port 587 \
  --auth LOGIN \
  --auth-user noreply@YOUR_DNS.com \
  --auth-password '<YOUR_PASSWORD>=' \
  --tls \
  --header "Subject:  Hello from YOUR_DNS Mail Server" \
  --body "This is a test email sent with real TLS and full domain authentication!"
```

Things done After this:
- Gmail Accepted
- SPF Passed
- DKIM Passed
- No Spam

---

# 8. Summary of Success

| Feature              | Status                         |
| :------------------- | :----------------------------- |
| SMTP 587 Submission  | Working                        |
| SPF Policy           | Set and Validated              |
| DKIM Signing         | Working                        |
| TLS Encryption       | Working (Let's Encrypt Cert)   |
| Gmail Inbox Delivery | Success                        |

---


# Future Improvements

* Automate Let's Encrypt renewal (IMPORTANT)
* Create a DMARC record: `v=DMARC1; p=none; rua=mailto:your-email@gmail.com`
* Tighten DMARC policy to "quarantine" or "reject" for stricter enforcement
* Monitor mail logs with Grafana / Prometheus (optional)

---
