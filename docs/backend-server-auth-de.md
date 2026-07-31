# 🚀 Anleitung: Proton `lumo-tamer` Setup & Auth-Transfer (Windows ➔ Linux Server)

Diese Anleitung beschreibt, wie man `lumo-tamer` baut, als OpenAI-kompatiblen Server einrichtet und Proton-Sperren (422 Abuse Detection) umgeht, indem man die Browser-Session unter Windows extrahiert und nahtlos auf einen headless Linux-Server überträgt.

---

## 🎯 Das Problem & Die Strategie

* **Das Problem:** Protons Sicherheitssystem blockiert CLI-Logins (`proton-auth` / Go-Binary) sehr schnell mit Fehlermeldungen (z. B. `422 POST [https://mail.proton.me/api/auth/v4](https://mail.proton.me/api/auth/v4)...` / Code 2028).
* **Die Strategie:** Wir bauen das Projekt lokal, führen den Login **einmalig** über einen echten Google Chrome Browser unter Windows durch und übertragen die erzeugte Vault-Datei auf den Linux-Server. Der Server hält die Session danach über `autoRefresh` dauerhaft selbstständig aktiv.

---

## 🛠️ Schritt 1: Build & Vorbereitung unter Windows

### 1.1 Projekt bauen und CLI verknüpfen
Öffne die Eingabeaufforderung (CMD) oder PowerShell im Projektordner und führe den Build-Prozess durch:

```cmd
cd D:\github.com\velvet-lab\lumo-tamer

# Abhängigkeiten installieren
npm install

# Vollständigen Build ausführen (baut das Projekt und die Go-Binaries)
npm run build:all

# Wichtig: Default-Konfiguration in den Dist-Ordner kopieren
copy config.defaults.yaml dist\config.defaults.yaml

# CLI global im System verknüpfen (stellt den 'tamer' Befehl bereit)
npm link
```

### 1.2 Pfad-Workaround für Windows-Binaries
Der Build erzeugt im `dist`-Ordner die Datei `proton-auth` (ohne `.exe`-Endung). Falls unter Windows Pfad-Fehler beim Ausführen von `proton-auth` auftreten, spiegelst du die Datei- und Ordnerstruktur manuell:

```cmd
mkdir dist\dist
copy dist\proton-auth dist\dist\proton-auth
copy dist\proton-auth dist\proton-auth.exe
copy dist\proton-auth dist\dist\proton-auth.exe
```

### 1.3 Master-Key im Windows Tresor hinterlegen (Der Master-Trick)
Damit die unter Windows verschlüsselte `vault.enc` später auf dem Linux-Server ohne Entschlüsselungsfehler gelesen werden kann, hinterlegen wir den Linux-Vault-Key direkt im **Windows Credential Manager** unter den **Generischen Anmeldeinformationen**:

1. Öffne im Windows-Startmenü die **Anmeldeinformationsverwaltung** (*Windows Credential Manager*).
2. Wähle **Windows-Anmeldeinformationen** aus.
3. Klicke im Bereich **Generische Anmeldeinformationen** auf **Generische Anmeldeinformation hinzufügen**.
4. Trage die Felder exakt wie folgt ein:

   | Feld | Wert / Format | Beschreibung |
   | :--- | :--- | :--- |
   | **Internet- oder Netzwerkadresse** | `lumo-tamer/vault-key` | Setzt sich zusammen aus `<service>/<account>` |
   | **Benutzername** | `vault-key` | Muss exakt dem `keychain.account` aus der `config.yaml` entsprechen |
   | **Kennwort** | `dein-geheimer-server-key` | Der Schlüssel/Textstring, den dein Linux-Server als Vault-Key liest |

#### 📌 Referenz im Windows-Tresor:
```text
lumo-tamer/vault-key
   Internet- oder Netzwerkadresse: lumo-tamer/vault-key
   Benutzername: vault-key
   Kennwort: ********
```

---

## 🌐 Schritt 2: Chrome Debugging & Session-Extraction unter Windows

### 2.1 Chrome im CDP-Remote-Debugging-Modus starten
Schließe alle laufenden Chrome-Instanzen (`taskkill /F /IM chrome.exe`) und starte Chrome mit explizitem Datenverzeichnis:

```cmd
"C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-address=0.0.0.0 --remote-debugging-port=9222 --remote-debugging-allowed-origins=* --disable-dev-shm-usage --user-data-dir="C:\chrome-dev-profile"
```

> ⚠️ **Wichtig:** Der Parameter `--user-data-dir` ist zwingend erforderlich, da Chrome das Remote-Debugging auf Port 9222 ohne eigenes Profil laut Sicherheitsrichtlinie sonst blockiert.

### 2.2 Anmelden & `config.yaml` für Windows anpassen
1. Öffne im neu gestarteten Chrome-Fenster **[lumo.proton.me](https://lumo.proton.me)** und logge dich regulär ein.
2. Passe die `config.yaml` im Windows-Projektverzeichnis an:

```yaml
auth:
  method: "browser"

  vault:
    path: "sessions/vault.enc"
    keychain:
      service: "lumo-tamer"
      account: "vault-key"

  browser:
    # 127.0.0.1 statt localhost nutzen, um IPv6-Probleme (::1 ECONNREFUSED) unter Windows zu vermeiden
    cdpEndpoint: "http://127.0.0.1:9222"
```

### 2.3 Session via CLI auslesen (`tamer auth browser`)
Führe in der CMD/PowerShell den dezidierten Auth-Befehl aus:

```cmd
tamer auth browser
```

`lumo-tamer` verbindet sich via CDP mit Chrome, zieht die authentifizierten Tokens aus dem Browser und schreibt sie verschlüsselt in `sessions/vault.enc` (unter Verwendung des Keys aus dem Windows Tresor). Sobald der Befehl erfolgreich durchgelaufen ist, sind alle benötigten Dateien im Ordner `sessions/` bereit.

---

## 📦 Schritt 3: Transfer auf den Linux-Server

Kopiere den erzeugten Ordner **`sessions/`** von Windows auf deinen Linux-Server (per `scp`, Rsync oder WinSCP) in das `lumo-tamer`-Projektverzeichnis:

```bash
scp -r sessions/ user@dein-server:/pfad/zu/lumo-tamer/
```

---

## ⚙️ Schritt 4: Server-Konfiguration & Start (Linux)

### 4.1 Server `config.yaml` anpassen
Passe die `config.yaml` auf dem Linux-Server an:

```yaml
auth:
  # Auf dem Server nutzen wir "login" oder "rclone" als Fallback (kein CDP-Browser nötig)
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
  apiKey: "dein-sicherer-api-key"
  apiModelName: "lumo"
```

> **Hinweis:** Stelle sicher, dass der Inhalt von `sessions/vault.key` auf dem Server exakt mit dem Passwort übereinstimmt, das du in Schritt 1.3 im Windows-Tresor eingegeben hast.

### 4.2 Docker Container starten
Da das Docker-Image bereits über GitHub Container Registry (GHCR) bereitsteht, wird der Container direkt via Docker Compose gestartet:

```bash
# Docker Image pullen und Container im Hintergrund starten
docker compose pull
docker compose up -d
```

---

## 🧹 Schritt 5: Aufräumen auf dem Windows-Rechner

Nachdem der Transfer erfolgreich war und der Server läuft, kannst du das temporäre Debug-Profil und die Anmeldedaten auf Windows sicher entfernen:

1. **Chrome-Prozesse beenden:**
   ```cmd
   taskkill /F /IM chrome.exe
   ```

2. **Temporäres Chrome-Entwicklerprofil löschen:**
   ```cmd
   rmdir /S /Q "C:\chrome-dev-profile"
   ```

3. **Eintrag aus dem Windows-Tresor entfernen (optional):**
   * Öffne die **Anmeldeinformationsverwaltung** (*Windows Credential Manager*).
   * Gehe zu **Windows-Anmeldeinformationen** $\rightarrow$ **Generische Anmeldeinformationen**.
   * Klappe den Eintrag `lumo-tamer/vault-key` auf und klicke auf **Entfernen**.

4. **NPM-Verknüpfung entfernen (optional):**
   ```cmd
   npm unlink -g lumo-tamer
   ```

---

## 🧪 Schritt 6: Funktionsprüfung

Teste die Verbindung vom Windows-Rechner (oder einem beliebigen Client im Netzwerk) aus. Ersetze `<DEINE-SERVER-IP>` durch die IP-Adresse deines Linux-Servers (z. B. `192.168.1.100`):

```bash
curl http://<DEINE-SERVER-IP>:3003/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer dein-sicherer-api-key" \
  -d '{
    "model": "lumo-max",
    "messages": [
      {"role": "user", "content": "Wer bist du?"}
    ]
  }'
```

* **Erfolgskriterium:** Das Modell antwortet sauber als Proton Lumo.
* **Ergebnis:** Die Session wird von `lumo-tamer` auf dem Linux-Server automatisch im Hintergrund verlängert. Der Browser-Login unter Windows muss **nicht** wiederholt werden.