# 🚀 Guide: Proton `lumo-tamer` Setup & Auth Transfer (Windows ➔ Linux Server)

This guide explains how to build `lumo-tamer`, set it up as an OpenAI-compatible server, and bypass Proton's security restrictions (422 Abuse Detection) by extracting the browser session under Windows and transferring it to a headless Linux server.

---

## 🎯 Problem & Strategy

* **The Problem:** Proton's security system quickly blocks CLI logins (`proton-auth` / Go binary) with error responses (e.g., `422 POST [https://mail.proton.me/api/auth/v4](https://mail.proton.me/api/auth/v4)...` / Code 2028).
* **The Strategy:** We perform the login **once** via a real Google Chrome browser under Windows and transfer the generated encrypted vault file to the Linux server. The server then maintains the session automatically in the background using `autoRefresh`.

---

## 🛠️ Step 1: Build & Preparation on Windows

### 1.1 Clone, Build, and Link the CLI
Open Command Prompt (CMD) or PowerShell and clone the repository:

```cmd
# Clone the repository & enter the directory
git clone https://github.com/ZeroTricks/lumo-tamer.git
cd lumo-tamer

# Install dependencies
npm install

# Run full build (compiles the project and Go binaries)
npm run build:all

# Important: Copy default configuration into the dist folder
copy config.defaults.yaml dist\config.defaults.yaml

# Globally link CLI (provides the 'tamer' command system-wide)
npm link
```

### 1.2 Path Workaround for Windows Binaries
The build process generates the file `proton-auth` (without the `.exe` extension) in the `dist` directory. To prevent path errors under Windows, mirror the binary file structure manually:

```cmd
mkdir dist\dist
copy dist\proton-auth dist\dist\proton-auth
copy dist\proton-auth dist\proton-auth.exe
copy dist\proton-auth dist\dist\proton-auth.exe
```

### 1.3 Store Master Key in Windows Credential Manager (The Master Trick)
To ensure the `vault.enc` file encrypted on Windows can be decrypted on the Linux server, we store the Linux vault key directly in the **Windows Credential Manager** under **Generic Credentials**:

1. Open **Credential Manager** from the Windows Start menu.
2. Select **Windows Credentials**.
3. Under **Generic Credentials**, click **Add a generic credential**.
4. Fill out the fields exactly as follows:

   | Field | Value / Format | Description |
   | :--- | :--- | :--- |
   | **Internet or network address** | `lumo-tamer/vault-key` | Formatted as `<service>/<account>` |
   | **User name** | `vault-key` | Must match `keychain.account` in `config.yaml` |
   | **Password** | `your-secret-server-key` | The secret key/string your Linux server uses to decrypt the vault |

#### 📌 Reference in Windows Credential Manager:
```text
lumo-tamer/vault-key
   Internet or network address: lumo-tamer/vault-key
   User name: vault-key
   Password: ********
```

---

## 🌐 Step 2: Chrome Debugging & Session Extraction on Windows

### 2.1 Launch Chrome in CDP Remote Debugging Mode
Close all running Chrome instances (`taskkill /F /IM chrome.exe`) and launch Chrome with a dedicated user data directory:

```cmd
"C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-address=0.0.0.0 --remote-debugging-port=9222 --remote-debugging-allowed-origins=* --disable-dev-shm-usage --user-data-dir="C:\chrome-dev-profile"
```

> ⚠️ **Important:** The `--user-data-dir` flag is required. Chrome security policies block remote debugging on port 9222 without an explicit user data profile.

### 2.2 Log In & Configure `config.yaml` for Windows
1. In the newly launched Chrome window, navigate to **[lumo.proton.me](https://lumo.proton.me)** and log in normally.
2. Edit `config.yaml` in your Windows project directory:

```yaml
auth:
  method: "browser"

  vault:
    path: "sessions/vault.enc"
    keychain:
      service: "lumo-tamer"
      account: "vault-key"

  browser:
    # Use 127.0.0.1 instead of localhost to avoid IPv6 (::1 ECONNREFUSED) resolution issues on Windows
    cdpEndpoint: "http://127.0.0.1:9222"
```

### 2.3 Extract Session via CLI (`tamer auth browser`)
Run the dedicated authentication command in your CMD/PowerShell:

```cmd
tamer auth browser
```

`lumo-tamer` connects to Chrome via CDP, extracts the authenticated tokens, and encrypts them into `sessions/vault.enc` using the key from your Windows Credential Manager. Once the command completes, all required session files will be generated in the `sessions/` directory.

---

## 📦 Step 3: Transfer to Linux Server

Copy the generated **`sessions/`** folder from Windows to your Linux server (using `scp`, Rsync, or WinSCP) into the `lumo-tamer` project directory:

```bash
scp -r sessions/ user@your-server-ip:/path/to/lumo-tamer/
```

---

## ⚙️ Step 4: Server Configuration & Start (Linux)

### 4.1 Edit `config.yaml` on Linux Server
Configure `config.yaml` on your Linux server:

```yaml
auth:
  # Use "login" or "rclone" as server fallback (no CDP browser required)
  method: "login"

  vault:
    path: "sessions/vault.enc"
    keyFilePath: "sessions/vault.key"

  autoRefresh:
    enabled: true
    intervalHours: 20
    onError: true

server:
  port: 3003
  apiKey: "your-secure-api-key"
  apiModelName: "lumo"
```

> **Note:** Ensure the contents of `sessions/vault.key` on the server match **exactly** the password entered in Step 1.3 in the Windows Credential Manager.

### 4.2 Start Docker Container
Since the pre-built Docker image is available on GitHub Container Registry, start the container using Docker Compose:

```bash
# Pull the latest image and launch container in background
docker compose pull
docker compose up -d
```

---

## 🧹 Step 5: Windows Cleanup

Once the server is up and running successfully, clean up the temporary Chrome profile and credentials from your Windows machine:

1. **Terminate Chrome debug process:**
   ```cmd
   taskkill /F /IM chrome.exe
   ```

2. **Delete temporary Chrome profile directory:**
   ```cmd
   rmdir /S /Q "C:\chrome-dev-profile"
   ```

3. **Remove Windows Credential Manager entry (optional):**
   * Open **Credential Manager**.
   * Go to **Windows Credentials** $\rightarrow$ **Generic Credentials**.
   * Expand `lumo-tamer/vault-key` and click **Remove**.

4. **Unlink global NPM package (optional):**
   ```cmd
   npm unlink -g lumo-tamer
   ```

---

## 🧪 Step 6: Testing & Verification

Test the connection from your Windows machine (or any client in the network). Replace `<YOUR-SERVER-IP>` with your Linux server's IP address (e.g., `192.168.1.100`):

```bash
curl http://<YOUR-SERVER-IP>:3003/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-secure-api-key" \
  -d '{
    "model": "lumo-max",
    "messages": [
      {"role": "user", "content": "Who are you?"}
    ]
  }'
```

* **Success Criteria:** The model responds correctly identifying itself as Proton Lumo.
* **Outcome:** The session token will be automatically refreshed in the background by `lumo-tamer` on your Linux server. **No re-authentication via Windows browser is required.**