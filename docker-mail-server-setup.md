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

# 3. SSL Certificates (Cloudflare DNS Challenge Automation)

Automatic renewal via Cloudflare API:

## Cloudflare API Token

* Token created with permission to edit DNS zone for `exeltan.com`.
* Saved at `/root/.secrets/certbot/cloudflare.ini` with permissions `600`.

## Certbot Command

```bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/certbot/cloudflare.ini \
  -d mail.exeltan.com
```

> Certificate automatically renewed and installed to `/etc/letsencrypt/live/mail.exeltan.com/`.

## Auto Renewal Setup

Added to `root` crontab (`sudo crontab -e`):

```bash
0 3 * * * certbot renew --dns-cloudflare --dns-cloudflare-credentials /root/.secrets/certbot/cloudflare.ini --post-hook "docker restart mailserver"
```

This renews nightly at 3AM, and restarts mailserver if cert is renewed.

---

# 4. Cloudflare DNS Settings

| Type | Name              | Content                                             | TTL  | Proxy    |
| :--- | :---------------- | :-------------------------------------------------- | :--- | :------- |
| A    | `mail`            | `<YOUR IP>`                                         | Auto | DNS Only |
| MX   | `@`               | `mail.exeltan.com`                                  | Auto | DNS Only |
| TXT  | `_dmarc`          | `v=DMARC1; p=none; rua=mailto:your-email@gmail.com` | Auto | DNS Only |
| TXT  | `@`               | `v=spf1 ip4:<YOUR IP> -all`                    | Auto | DNS Only |
| TXT  | `mail._domainkey` | (DKIM Public Key - see below)                       | Auto | DNS Only |


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

| Feature                | Status                         |
| :-------------------   | :----------------------------- |
| SMTP 587 Submission    | Working                        |
| SPF Policy             | Set and Validated              |
| DKIM Signing           | Working                        |
| TLS Encryption         | Working (Let's Encrypt Cert)   |
| Gmail Inbox Delivery   | Success                        |
| Automated Cert Renewal |  Enabled                       |

---

# Future Improvements

* [ ] Implement email monitoring with Grafana / Prometheus

---

