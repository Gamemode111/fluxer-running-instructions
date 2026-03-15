# Fluxer — Installation Guide

> Step-by-step instructions for deploying Fluxer on a fresh Linux (Ubuntu/Debian) server with HTTPS support.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Create a System User](#1-create-a-system-user)
3. [Clone the Repository](#2-clone-the-repository)
4. [Install Nix & Devenv](#3-install-nix--devenv)
5. [Enter the Dev Shell](#4-enter-the-dev-shell)
6. [Build the Project](#5-build-the-project)
7. [Configure the Domain](#6-configure-the-domain)
8. [Generate SSL Certificates](#7-generate-ssl-certificates)
9. [Configure Caddy (HTTPS)](#8-configure-caddy-https)
10. [Start the Server](#9-start-the-server)
11. [Client-Side Setup](#10-client-side-setup)
12. [Troubleshooting](#troubleshooting)

---

## Prerequisites

- A Linux server (Ubuntu/Debian recommended)
- `sudo` privileges
- `git`, `curl` installed
- A public IP address (for remote access)

---

## 1. Create a System User

```bash
sudo adduser fluxer
sudo usermod -aG sudo fluxer
sudo -u fluxer -i
```

All subsequent commands should be run as the `fluxer` user.

---

## 2. Clone the Repository

```bash
git clone https://github.com/fluxerapp/fluxer.git
cd fluxer/
```

---

## 3. Install Nix & Devenv

Install the Nix package manager (daemon mode):

```bash
sh <(curl -L https://nixos.org/nix/install) --daemon
```

**Log out and log back in** for the Nix environment to take effect:

```bash
exit
sudo -u fluxer -i
```

Install Devenv:

```bash
nix-env --install --attr devenv -f https://github.com/NixOS/nixpkgs/tarball/nixpkgs-unstable
```

---

## 4. Enter the Dev Shell

```bash
cd ~/fluxer
devenv shell -- --show-trace
```

### Known Issue: `configPath` missing

If you see an error like:

```
error: attribute 'configPath' missing
```

at `/nix/store/…/src/modules/integrations/git-hooks.nix`, line 126 — apply this fix:

```bash
sudo nano /nix/store/<hash>-source/src/modules/integrations/git-hooks.nix
```

Change line 126 **from**:

```nix
configFile='${config.devenv.root}/${cfg.configPath}'
```

**to**:

```nix
configFile='${config.devenv.root}/${cfg.configPath or ""}'
```

Then re-enter the shell:

```bash
devenv shell
```

---

## 5. Build the Project

### 5.1 Install dependencies & generate schemas

```bash
pnpm install
pnpm run schema:generate
ls -la packages/config/src/MasterZodSchema.generated*
```

### 5.2 Build the WASM core

```bash
cd ~/fluxer/fluxer_app/crates/libfluxcore
cargo build --release --target wasm32-unknown-unknown
```

### 5.3 Generate assets

```bash
cd ~/fluxer/fluxer_app
pnpm run generate:colors
pnpm run generate:css-types
```

### 5.4 Extract & compile i18n translations

```bash
cd ~/fluxer/fluxer_app
pnpm run i18n:extract
pnpm run i18n:compile
ls -la src/locales/*/messages.mjs
```

### 5.5 Generate masks & WASM codegen

```bash
cd ~/fluxer/fluxer_app
pnpm run generate:masks

cd ~/fluxer/fluxer_app/crates/libfluxcore
cargo build --release --target wasm32-unknown-unknown

cd ~/fluxer/fluxer_app
pnpm run wasm:codegen
ls -la ~/fluxer/fluxer_app/pkgs/libfluxcore
```

### 5.6 Initial run to generate the config

Run the full environment once so that Fluxer compiles all remaining assets and auto-generates the configuration file `config/config.json`:

```bash
cd ~/fluxer
devenv up
```

> **⏳ Wait for the compilation to complete.** Build logs will scroll in the terminal — this may take several minutes depending on your hardware. Once the output stabilizes and all services have started, the file `config/config.json` will be created automatically.
>
> After you confirm the compilation has finished, **press `Ctrl + C`** to stop the process.

Verify the config was created:

```bash
ls -la config/config.json
```

You should see the file listed. Now proceed to editing it in the next step.

---

## 6. Configure the Domain

Edit `config/config.json` and update the `domain` section:

**Before:**

```json
"domain": {
  "base_domain": "localhost",
  "public_port": 48763
}
```

**After:**

```json
"domain": {
  "base_domain": "fluxer.dev",
  "public_port": 48763,
  "public_scheme": "https"
}
```

---

## 7. Generate SSL Certificates

### 7.1 Create the OpenSSL config

```bash
mkdir -p ~/fluxer/certs
```

Create the file `~/fluxer/certs/fluxer.conf` with the following content:

```ini
[req]
default_bits = 4096
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[dn]
CN = fluxer.dev

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = fluxer.dev
DNS.2 = localhost
IP.1 = 127.0.0.1
IP.2 = <YOUR_PUBLIC_IP>
```

> **Important:** Replace `<YOUR_PUBLIC_IP>` with your server's actual public IP address.

### 7.2 Generate the certificate

```bash
cd ~/fluxer/certs
openssl req -x509 -newkey rsa:4096 \
  -keyout fluxer.key \
  -out fluxer.crt \
  -days 365 -nodes \
  -config fluxer.conf

ls -la ~/fluxer/certs
```

You should see `fluxer.key`, `fluxer.crt`, and `fluxer.conf`.

---

## 8. Configure Caddy (HTTPS)

Edit `dev/Caddyfile.dev`.

**Before:**

```caddyfile
:48763 {
    handle /_caddy_health {
        respond "OK" 200
    }
```

**After:**

```caddyfile
fluxer.dev:48763 {
    tls /home/fluxer/fluxer/certs/fluxer.crt /home/fluxer/fluxer/certs/fluxer.key

    handle /_caddy_health {
        respond "OK" 200
    }
```

---

## 9. Start the Server

```bash
devenv up
```

---

## 10. Client-Side Setup

These steps are performed **on your local machine**, not on the server.

### 10.1 Update the hosts file

Add the following line to your hosts file:

| OS            | File path                                       |
|---------------|-------------------------------------------------|
| Windows       | `C:\Windows\System32\drivers\etc\hosts`         |
| Linux / macOS | `/etc/hosts`                                    |

```
<YOUR_SERVER_IP>   fluxer.dev
```

> Replace `<YOUR_SERVER_IP>` with the public IP of your server.

### 10.2 Launch Chrome (bypassing self-signed certificate warnings)

Since we use a self-signed certificate, Chrome needs special flags:

**Windows:**

```cmd
"C:\Program Files\Google\Chrome\Application\chrome.exe" ^
  --new-window ^
  --ignore-certificate-errors ^
  --disable-web-security ^
  --allow-insecure-localhost ^
  --user-data-dir="C:\chrome-hsts-bypass" ^
  about:blank
```

**Linux:**

```bash
google-chrome \
  --new-window \
  --ignore-certificate-errors \
  --disable-web-security \
  --allow-insecure-localhost \
  --user-data-dir="/tmp/chrome-hsts-bypass" \
  about:blank
```

### 10.3 Open Fluxer

Navigate to:

```
https://fluxer.dev:48763/login
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| **SSL certificate error** | Click "Advanced" → "Proceed" in the browser warning page. |
| **HSTS error in Chrome** | Make sure you launched Chrome with the flags from step 10.2. Use a fresh `--user-data-dir` to avoid cached HSTS policies. |
| **`configPath` missing** during `devenv shell` | Apply the Nix fix described in step 4. |
| **Port 48763 not reachable** | Check your firewall rules: `sudo ufw allow 48763/tcp` |
| **`devenv shell` hangs or fails** | Try `devenv shell -- --show-trace` for a detailed error trace. |
| **`config/config.json` not found** | Make sure you completed step 5.6 — run `devenv up`, wait for compilation to finish, then press `Ctrl + C`. |
