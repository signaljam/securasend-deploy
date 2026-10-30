# SecuraSend — Deployment Configuration

Deployment configuration for **SecuraSend**, a public instance of [Send](https://github.com/timvisee/send) running at [securasend.com](https://securasend.com).

This repository contains only the deployment layer — the Docker Compose stack, the reverse proxy configuration, and the environment template needed to stand up an equivalent instance. It does not contain application code, secrets, or user data.

---

## What this is

[Send](https://github.com/timvisee/send) is a fork of Mozilla's discontinued [Firefox Send](https://github.com/mozilla/send) — a file-sharing service where files are encrypted in the browser before upload, so the server never holds the decryption key. Files expire automatically and can be limited by download count.

**Lineage:**

```
Mozilla Send (archived 2020)
        └── timvisee/send            ← actively maintained fork
                └── this deployment  ← SecuraSend
```

This repo is built on [`timvisee/send-docker-compose`](https://github.com/timvisee/send-docker-compose), which provides the reference Compose setup including an nginx reverse proxy and automated Let's Encrypt certificates.

---

## Stack

Four containers, orchestrated by Docker Compose:

| Container | Role |
|---|---|
| `send` | The Send application (Node.js) |
| `redis` | Metadata store for upload records |
| `nginx-proxy` | Reverse proxy, TLS termination, HTTP → HTTPS redirect |
| `proxy-letsencrypt` | Certificate issuance and renewal via Let's Encrypt |

Uploaded files are stored on the host filesystem via a bind mount (`./uploads`), not inside the container. This means uploads survive container restarts and rebuilds.

---

## Requirements

- A Linux server with root or sudo access
- Docker Engine and the Docker Compose plugin
- A domain with an `A` record pointing at the server
- Ports 80 and 443 free on the host

Any reasonably sized VPS will run this comfortably. Send plus Redis typically use under 1 GB of RAM in normal operation. Disk is the real constraint — uploads accumulate until they expire, so size the volume according to how generous your file-size and expiry limits are.

---

## Deployment

### 1. Install Docker

Follow Docker's official instructions for your distribution. On Debian:

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/debian $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

Add your user to the `docker` group so you don't need `sudo` for every command:

```bash
sudo usermod -aG docker "$USER"
newgrp docker
```

### 2. Clone and configure

```bash
git clone https://github.com/signaljam/securasend-deploy.git
cd securasend-deploy
cp .env.example .env
```

Edit `.env` and set your own domain and a reachable email address:

```ini
DOCKER_SEND_IMAGE=registry.gitlab.com/timvisee/send:latest
HOST=your-domain.com
SEND_BASE_URL=https://your-domain.com
LETSENCRYPT_HOST=your-domain.com
LETSENCRYPT_EMAIL=you@example.com
```

Notes:

- `HOST` and `LETSENCRYPT_HOST` must match.
- `SEND_BASE_URL` must include the scheme and must not have a trailing slash.
- The email is used by Let's Encrypt for expiry and renewal-failure notices. It must be reachable, but it does not need to be an address on your domain.
- Watch for trailing whitespace on these lines — it can silently corrupt the value.

### 3. Fix upload directory permissions

**This step is required.** The Send container runs as an unprivileged user, but the `uploads` directory created by the Compose setup is owned by root. Without this, uploads fail partway through and crash the application:

```
uncaughtException: Error: ENOENT: no such file or directory, unlink '/uploads/...'
```

The browser connection drops mid-upload, and the container restarts. The fix is to give the host directory the same ownership as the user inside the container:

```bash
sudo chown -R 1000:1000 ./uploads
```

Verify it worked before starting the stack:

```bash
docker compose up -d
docker compose exec send sh -c "touch /uploads/.test && rm /uploads/.test && echo writable"
```

If that prints `writable`, uploads will work.

### 4. Start

```bash
docker compose up -d
```

The Let's Encrypt container will request a certificate once DNS resolves to your server. Certificates renew automatically thereafter.

Check status and logs:

```bash
docker compose ps
docker compose logs -f proxy-letsencrypt
```

---

## Configuration reference

The full set of environment variables is documented in [`timvisee/send`'s docker documentation](https://github.com/timvisee/send/blob/master/docs/docker.md). The ones that matter most for a public instance:

| Variable | Purpose |
|---|---|
| `MAX_FILE_SIZE` | Maximum upload size in bytes |
| `MAX_EXPIRE_SECONDS` | Longest permitted expiry |
| `EXPIRE_TIMES_SECONDS` | Expiry options offered in the UI |
| `MAX_DOWNLOADS` | Upper bound on download count |
| `DOWNLOAD_COUNTS` | Download-count options offered in the UI |
| `DEFAULT_DOWNLOADS` | Default download limit |

Several branding elements — accent colours, footer text, custom URLs — can be set through environment variables without rebuilding the application. Changing the logo requires building a custom image from a fork of `timvisee/send`.

---

## Operating notes

**Disk usage.** Uploads occupy host disk until they expire. Monitor with `df -h` and tune `MAX_FILE_SIZE` / expiry limits accordingly. Public instances fill up faster than intuition suggests.

**Abuse handling.** Because files are end-to-end encrypted, the operator cannot inspect content. Practical measures are conservative size and expiry limits, a published contact address for complaints, and a willingness to delete records on valid notice.

**Updates.**

```bash
docker compose pull
docker compose up -d
```

**Backups.** The only irreplaceable artefacts are `.env` and the Let's Encrypt certificate volume. Uploads are transient by design. Store a copy of `.env` somewhere safe and outside the repository.

---

## Repository hygiene

`.gitignore` excludes:

- `.env` — contains deployment-specific values
- `uploads/` — contains user data

Neither should ever be committed. If you fork this, verify both remain excluded before your first push.

---

## Licensing

The deployment configuration in this repository is released under the **Mozilla Public License 2.0**, matching the licence of the upstream [`timvisee/send`](https://github.com/timvisee/send) project it deploys.

Send itself is derived from Mozilla's Firefox Send, originally released under MPL-2.0. Modifications to application source, if any, are published in a separate repository.

**The licence covers the code. It does not cover the name, logo, or visual identity — see below.**

---

## Branding and trademarks

The **MPL-2.0 licence grants rights to the code only.** It does not grant any right to use the names, logos, or visual identity of SecuraSend or Signal Jam.

Specifically, the following are trademarks and brand assets of Signal Jam and are **not** licensed for reuse:

- The name **SecuraSend**
- The name **Signal Jam**
- The SecuraSend and Signal Jam logos, wordmarks, and icons
- The visual identity of the SecuraSend instance (colour scheme, typography, and interface styling as presented at securasend.com)

You are free to use this code to operate your own file-sharing instance. If you do, you **must**:

1. Remove all Signal Jam and SecuraSend branding from your deployment, and
2. Operate under a name and visual identity of your own that does not suggest affiliation with, endorsement by, or sponsorship from Signal Jam.

For permission to use any Signal Jam brand asset, [contact us.](https://signaljam.me/contact)

---

## About Signal Jam

Signal Jam is a podcast, blog, and community project about regaining and maintaining privacy in the Surveillance Age. From time to time, we publish tools which we hope provide value in educating, promoting, and sustaining digital privacy.

---

## Credits

- [Mozilla](https://github.com/mozilla/send) — original Firefox Send
- [timvisee](https://github.com/timvisee) — maintained fork and the Compose reference setup this repo is built on
