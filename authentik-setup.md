
## Authetik Setup:

### 1. create authentik folder for set up


### 2. Create Secret key and save into .env file  
````
openssl rand -base64 64
````

### 3. Create Docker Compose File : 

````
services:
  authentik-server:
    image: goauthentik/server:latest
    environment:
      AUTHENTIK_SECRET_KEY: ${AUTHENTIK_SECRET_KEY}
      AUTHENTIK_POSTGRESQL__HOST: authentik-db
      AUTHENTIK_POSTGRESQL__USER: authentik
      AUTHENTIK_POSTGRESQL__PASSWORD: ${AUTHENTIK_APP_DB_PASSWORD}
      AUTHENTIK_POSTGRESQL__DB: authentik
      AUTHENTIK_REDIS__HOST: authentik-redis
    volumes:
      - /var/lib/authentik-data:/data/
    depends_on:
      - authentik-db
      - authentik-redis
    networks:
      - shared_network

  authentik-db:
    image: postgres:13-alpine  # Correct image for PostgreSQL
    environment:
      POSTGRES_USER: authentik
      POSTGRES_PASSWORD: ${AUTHENTIK_DB_PASSWORD}
      POSTGRES_DB: authentik
    volumes:
      - /var/lib/authentik-db-data:/var/lib/postgresql/data/
    networks:
      - shared_network

  authentik-redis:
    image: redis:alpine      # Correct image for Redis
    networks:
      - shared_network

volumes:
  authentik-data:
  authentik-db-data:

networks:
  shared_network:
````


##  Blocker
````
exeltan@v2202503261552325218:~/authentik$ docker compose up
WARNING: Error parsing config file (/home/exeltan/.docker/config.json): invalid character '}' looking for beginning of object key string
WARNING: Error parsing config file (/home/exeltan/.docker/config.json): invalid character '}' looking for beginning of object key string
[+] Running 2/4
 ⠋ worker Pulling                                                                                                                                       1.1s
 ✘ redis Error      error getting credentials - err: exit status 1, out: `exit status 2: gpg: decryption failed: No secret key`                         0.1s
 ⠋ server Pulling                                                                                                                                       1.1s
 ✘ postgresql Error error getting credentials - err: exit status 1, out: `exit status 2: gpg: decryption failed: No secret key`     

````

Docker attempts to use the stored credentials (in this case, from pass) for all registry interactions.
pulling an image from Docker Hub, it tries to use the GHCR credentials from docker helper credentials stored in PASS, leading to an authentication failure and the GPG decryption error.


**Hours spent debuuging:** 7 hours (07:00pm - 02:00 am)
## **Solution:**

To resolve this issue, you need to add Docker Hub credentials to your `pass` password store.

**Steps:**

1.  **Create a Docker Hub Account (if you don't have one):**
    * Go to [https://hub.docker.com/](https://hub.docker.com/) and sign up.

2.  **Log in to Docker Hub via the Command Line:**
    * Open your VPS and run: 

        ```bash
        docker login
        ```

    * Enter your Docker Hub username and password when prompted.

3.  **Check that Docker Hub is added to config.json**
    * Open your terminal and run:

        ```bash
         cat .docker/config.json
        ```

    * Enter your Docker Hub username and password when prompted and then check if you credentials are saved in you pass
      ```

        "auths": {
                "ghcr.io": {},
                "https://index.docker.io/v1/": {}
        },
        "credsStore": "pass"
        ```

3.  **Verify Credentials in `pass`:**
    * Docker will automatically store the Docker Hub credentials in `pass`.
    * To verify, run (Ids are Base 64 encoded):

        ```bash
        pass show
        
        Password Store
        └── docker-credential-helpers
            ├── aHR0cHM6Ly9pbmRleC5kb2NrZXIuaW8vdjEv (DHCR)
            │   └── mediocreasian
            └── Z2hjci5pbw== (GHCR)
                └── mediocreasian

        ```

    * This command should display your Docker Hub credentials.

4.  **Run `docker compose up` Again:**
    * Now, retry running your `docker compose up` command. It should successfully pull images from both GHCR and Docker Hub for the Authentik docker compose file 

**Explanation:**

By logging in to Docker Hub, you ensure that Docker stores the necessary credentials in `pass` under the correct path, allowing it to authenticate with Docker Hub OR GHCR  when pulling images.

**Important Considerations:**
* **Security:** Ensure your Docker Hub password and GPG key are secure.
* **`pass` Configuration:** Verify that `pass` is correctly initialized and configured.
* **Base64 Encoding:** Remember that Docker stores registry URLs as base64 encoded strings in `pass`.


5. Blocker Number  2: 
##  This is because of our CloudFlare set up where 
````
time="2025-04-05T17:54:39Z" level=error msg="Unable to obtain ACME certificate for domains \"authentik.exeltan.com\": unable to generate a certificate for the domains [authentik.exeltan.com]: 
error: one or more domains had a problem:\n[authentik.exeltan.com] acme: error: 403 :: urn:ietf:params:acme:error:unauthorized :: 
Cannot negotiate ALPN protocol \"acme-tls/1\" for tls-alpn-01 challenge\n" rule="Host(authentik.exeltan.com)" providerName=myresolver.acme ACME CA="https://acme-v02.api.letsencrypt.org/directory"
````


## 💡 Root Cause

The error occurs because **Cloudflare was configured with SSL mode set to _Full (Strict)_**, which requires a valid certificate to be already present and trusted on the origin server (your VPS).

At the same time, **Traefik was trying to solve a TLS-ALPN-01 challenge**, which failed because:

- Cloudflare proxy (orange cloud ☁️) intercepted the TLS handshake
- Let’s Encrypt could not directly connect and negotiate the ALPN protocol `acme-tls/1` with Traefik
- Resulted in a 403 Unauthorized error from Let's Encrypt

---

## Understanding Cloudflare SSL Modes

- **Flexible:** Cloudflare serves HTTPS to users, but communicates with your server via HTTP
- **Full:** Cloudflare uses HTTPS to connect to your server, but doesn't validate the certificate
- **Full (Strict):** Cloudflare uses HTTPS **and** requires a **valid certificate** from a trusted CA (e.g. Let’s Encrypt)

Using **Full (Strict)** before Let’s Encrypt is properly configured on the origin **breaks the initial ACME challenge**.

---

### **Steps:**

####  Resolution

1.  **Temporarily disable the Cloudflare proxy**:
    - Go to your **DNS settings** in Cloudflare
    - Set the record for `authentik.exeltan.com` to **"DNS Only"** (gray cloud ☁️)
  
2.  **Update Traefik to use the HTTP challenge**:
    ```yaml
    - "--certificatesresolvers.myresolver.acme.httpchallenge.entrypoint=web"
    ```

3.  **Ensure HTTP (port 80) and HTTPS (port 443) are open** on your VPS

4.  **Restart Traefik and wait for it to obtain the certificate**

5.  **Once the cert is issued**, go back to Cloudflare:
    - Enable proxy (orange cloud ☁️)
    - Set SSL mode back to **Full (Strict)**

---

####  Verified Working Setup

- DNS record for `authentik.exeltan.com` pointing to VPS IP
- Cloudflare proxy **disabled during certificate issuance**
- Traefik using **httpChallenge**
- ACME cert successfully generated and auto-renewing

---


### Check if SSL Certificate has been issued: 
  ````
  openssl s_client -connect <sub_domain>.exeltan.com:443
  ````