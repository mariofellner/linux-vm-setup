# ubuntu-vm-setup

Initial-Setup-Script für eine frische **Ubuntu LTS**-VM in **VMware Workstation** hinter einem Corporate-Proxy.

**Unterstützte Varianten:** Desktop und Server (werden automatisch erkannt).
**Getestet mit:** Ubuntu 22.04, 24.04 und 26.04 LTS.

## Was macht das Script?

- Konfiguriert HTTP/HTTPS-Proxy für:
  - Login-Shells (`/etc/profile.d/99-proxy.sh`)
  - System-weit (`/etc/environment`)
  - APT (`/etc/apt/apt.conf.d/95proxy`)
  - wget (`/etc/wgetrc`)
  - curl (`/root/.curlrc`)
  - snap (falls installiert)
- Installiert **`open-vm-tools`** für saubere VMware-Integration.
  - **Desktop-Variante** zusätzlich **`open-vm-tools-desktop`** (Copy-Paste, Ordner-Shares, Mouse-Grab, Shutdown aus dem Host, dynamisches Display-Resizing).
  - **Server-Variante** ohne `-desktop`-Paket (keine GUI-Abhängigkeiten).
- Installiert und aktiviert **`openssh-server`** für Remote-Zugriff.
  - Aktiviert **`ssh.socket` UND `ssh.service`** (Ubuntu 24.04 nutzt Socket-Activation).
- Öffnet Port 22 in `ufw`, falls UFW aktiv ist.
- Führt am Ende automatische Self-Tests aus:
  - ssh.service **oder** ssh.socket aktiv
  - Port 22 lauscht
  - echter SSH-Banner-Handshake gegen `127.0.0.1:22`
  - `PasswordAuthentication`-Status (nur Info)
  - open-vm-tools läuft
  - `apt update` geht über den Proxy (mit `Error-Mode=any`)

## Voraussetzungen

- Ubuntu 22.04, 24.04 oder 26.04 LTS (Desktop **oder** Server; frisch installiert)
- sudo-fähiger Benutzer
- Erreichbarer HTTP/HTTPS-Proxy

## Desktop oder Server? (Autodetect)

Das Script erkennt die Variante automatisch:

1. `systemctl get-default` → `graphical.target` = Desktop, sonst Server
2. Alternativ: ist `ubuntu-desktop` oder `ubuntu-desktop-minimal` installiert? → Desktop

Falls die Erkennung falsch liegt oder du sie überschreiben willst:

```bash
sudo ./setup-ubuntu-vm.sh --server      # erzwinge Server-Variante
sudo ./setup-ubuntu-vm.sh --desktop     # erzwinge Desktop-Variante
```

Der einzige Unterschied: auf **Desktop** wird zusätzlich `open-vm-tools-desktop`
installiert (GUI-Integration), auf **Server** nicht (keine X11/GUI-Abhängigkeiten).

## Script holen und ausführbar machen

Je nachdem, wie du das Script auf die VM bekommst, fehlt ggf. das Execute-Bit.
Bei **ZIP-Download von GitHub** (grüner "Code"-Button → "Download ZIP")
gehen Unix-Dateirechte immer verloren – das Script wird nach dem Entpacken
nicht direkt startbar sein. Typische Fehlermeldung:

```
sudo: ./setup-ubuntu-vm.sh: command not found
```

### Variante A: Git clone (empfohlen)

Beim Klonen via `git` bleiben die Dateirechte inkl. Execute-Bit erhalten:

```bash
# Einmalig apt + git über den Proxy bereitstellen (falls git fehlt):
sudo apt update && sudo apt install -y git

git clone https://github.com/mariofellner/ubuntu-vm-setup.git
cd ubuntu-vm-setup
sudo ./setup-ubuntu-vm.sh
```

### Variante B: ZIP-Download – Execute-Bit nachsetzen

```bash
cd ~/Downloads/ubuntu-vm-setup-main     # oder wo du entpackt hast
chmod +x setup-ubuntu-vm.sh
sudo ./setup-ubuntu-vm.sh
```

### Variante C: Ohne Execute-Bit – direkt mit bash starten

Funktioniert immer, auch ohne `chmod`:

```bash
sudo bash setup-ubuntu-vm.sh
```

### Windows-Zeilenendings?

Falls das Script über Windows/OneDrive/Teams auf die VM kam und beim Start
folgender Fehler erscheint:

```
/usr/bin/env: 'bash\r': No such file or directory
```

dann sind CRLF-Line-Endings schuld. Einmalig beheben mit:

```bash
sed -i 's/\r$//' setup-ubuntu-vm.sh
```

## Verwendung

### Interaktiv (fragt nach Host und Port)

```bash
sudo ./setup-ubuntu-vm.sh
```

### Mit Parametern

```bash
sudo ./setup-ubuntu-vm.sh --host proxy.example.com --port 8080
```

### Mit Proxy-Auth

```bash
sudo ./setup-ubuntu-vm.sh \
  --host proxy.example.com --port 8080 \
  --user mario --pass 'GeheimesPasswort!' \
  --yes
```

### Via Umgebungsvariablen

```bash
sudo PROXY_HOST=proxy.example.com PROXY_PORT=8080 ASSUME_YES=1 \
  ./setup-ubuntu-vm.sh
```

### Ohne Proxy (Direktzugriff)

```bash
sudo ./setup-ubuntu-vm.sh --skip-proxy
```

## Optionen

| Option | Beschreibung |
|---|---|
| `--host <host>` | Proxy-Hostname oder IP |
| `--port <port>` | Proxy-Port (1–65535) |
| `--user <user>` | Proxy-Benutzer (optional) |
| `--pass <pw>` | Proxy-Passwort (optional, wird URL-encoded) |
| `--no-proxy <liste>` | Komma-Liste für `NO_PROXY` |
| `--skip-proxy` | Proxy-Konfiguration überspringen |
| `--server` | Als Server behandeln (kein `open-vm-tools-desktop`) |
| `--desktop` | Als Desktop behandeln (mit `open-vm-tools-desktop`) |
| `-y`, `--yes` | Alle Rückfragen mit Ja beantworten |
| `-h`, `--help` | Hilfe anzeigen |

## Logging

Alle Aktionen werden in `/var/log/setup-ubuntu-vm.log` protokolliert.

## Idempotenz

Das Script kann mehrfach ausgeführt werden. Originale von `/etc/environment` und `/etc/wgetrc` werden beim ersten Lauf als `.orig` gesichert; Proxy-Einträge werden vor dem Schreiben entfernt.

## Nach dem Lauf

```bash
# IP der VM ermitteln
hostname -I

# Per SSH vom Host verbinden
ssh <benutzer>@<vm-ip>

# Status der Dienste
systemctl status ssh ssh.socket open-vm-tools
```

### SSH-Hinweise

Ubuntu 24.04 aktiviert SSH per **Socket-Activation**. `ssh.socket` lauscht auf Port 22 und startet `ssh.service` beim ersten Connect. Der Self-Test akzeptiert deshalb beide Zustände als "aktiv".

Standardmäßig ist `PasswordAuthentication yes` aktiv – der Self-Test meldet den Status. Für höhere Sicherheit empfiehlt sich nach dem ersten Login der Umstieg auf SSH-Keys:

```bash
# auf dem Host:
ssh-copy-id <benutzer>@<vm-ip>

# auf der VM:
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

## Getestet mit

| Ubuntu | Desktop | Server |
|---|---|---|
| 22.04 LTS (Jammy Jellyfish) | ✅ | ✅ |
| 24.04 LTS (Noble Numbat) | ✅ | ✅ |
| 26.04 LTS (Resolute Raccoon) | ✅ | ✅ |

- VMware Workstation 17
- Squid als HTTP-Proxy (für automatisierte Tests)

### Hinweise zu 26.04 LTS

Ubuntu 26.04 LTS wurde am 23. April 2026 released und bringt unter anderem
systemd 259 und Linux-Kernel 7.0 mit. Für dieses Script sind keine
Anpassungen nötig – alle verwendeten Pfade (`/etc/environment`,
`/etc/apt/apt.conf.d/`, `/etc/profile.d/`, `ssh.socket`-Activation) sind
kompatibel.

## Lizenz

MIT – siehe [LICENSE](LICENSE).
