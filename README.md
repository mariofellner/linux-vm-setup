# linux-vm-setup

Initial-Setup-Script für eine frische **Linux-VM** in **VMware Workstation** hinter einem Corporate-Proxy – funktioniert auf Ubuntu, Rocky Linux und Fedora in Desktop- **und** Server-Varianten.

## Unterstützte Distributionen

| Distribution | Versionen | Desktop | Server |
|---|---|---|---|
| Ubuntu LTS | 22.04 / 24.04 / 26.04 | ✅ | ✅ |
| Rocky Linux | 9.x / 10.x | ✅ | ✅ |
| Fedora | 42 / 43 / 44 | ✅ | ✅ |

Andere Debian- oder RHEL-Derivate (Debian, AlmaLinux, Oracle Linux, CentOS Stream) funktionieren mit sehr hoher Wahrscheinlichkeit auch – sie werden über `ID_LIKE` erkannt. Das Script warnt bei nicht-getesteten Versionen, bricht aber nicht ab.

## Was macht das Script?

- Konfiguriert HTTP/HTTPS-Proxy für:
  - Login-Shells (`/etc/profile.d/99-proxy.sh`)
  - System-weit (`/etc/environment`)
  - **Paketmanager** – distro-abhängig:
    - Debian-Familie: `/etc/apt/apt.conf.d/95proxy`
    - RHEL-Familie: `/etc/dnf/dnf.conf` (`proxy=`, `proxy_username=`, `proxy_password=`)
  - wget (`/etc/wgetrc`)
  - curl (`/root/.curlrc`)
  - snap (falls installiert)
- Installiert **`open-vm-tools`** für saubere VMware-Integration.
  - Desktop-Variante zusätzlich **`open-vm-tools-desktop`** (Copy-Paste, Ordner-Shares, Mouse-Grab, Shutdown aus dem Host, dynamisches Display-Resizing).
  - Server-Variante ohne `-desktop`-Paket.
- Installiert und aktiviert **`openssh-server`** für Remote-Zugriff.
  - Ubuntu 24.04+: aktiviert `ssh.service` **und** `ssh.socket` (Socket-Activation).
  - Ubuntu 22.04 / Rocky / Fedora: aktiviert klassisch `ssh.service` bzw. `sshd.service`.
- Öffnet Port 22 in der jeweiligen Firewall:
  - Debian-Familie: **ufw** (nur wenn aktiv)
  - RHEL-Familie: **firewalld** (nur wenn aktiv) – `firewall-cmd --permanent --add-service=ssh`
- Führt am Ende automatische Self-Tests aus:
  - SSH-Unit aktiv
  - Port 22 lauscht
  - open-vm-tools läuft
  - Paketmanager-Update geht durch den Proxy

## Voraussetzungen

- Frische Installation einer der oben genannten Distros
- sudo-fähiger Benutzer
- Erreichbarer HTTP/HTTPS-Proxy

## Script holen und ausführbar machen

Bei **ZIP-Download von GitHub** gehen Unix-Dateirechte verloren. Typische Fehlermeldung: `sudo: ./setup-linux-vm.sh: command not found`.

### Variante A: Git clone (empfohlen)

```bash
# Ubuntu/Debian
sudo apt update && sudo apt install -y git
# Rocky/Fedora
sudo dnf install -y git

git clone https://github.com/mariofellner/linux-vm-setup.git
cd linux-vm-setup
sudo ./setup-linux-vm.sh
```

### Variante B: ZIP-Download – Execute-Bit nachsetzen

```bash
cd ~/Downloads/linux-vm-setup-main
chmod +x setup-linux-vm.sh
sudo ./setup-linux-vm.sh
```

### Variante C: Ohne Execute-Bit – direkt mit bash starten

```bash
sudo bash setup-linux-vm.sh
```

### Windows-Zeilenendings?

Falls beim Start `/usr/bin/env: 'bash\r': No such file or directory` erscheint:

```bash
sed -i 's/\r$//' setup-linux-vm.sh
```

## Verwendung

### Interaktiv (fragt nach Host und Port)

```bash
sudo ./setup-linux-vm.sh
```

### Mit Parametern

```bash
sudo ./setup-linux-vm.sh --host proxy.example.com --port 8080
```

### Mit Proxy-Auth

```bash
sudo ./setup-linux-vm.sh \
  --host proxy.example.com --port 8080 \
  --user mario --pass 'GeheimesPasswort!' \
  --yes
```

### Via Umgebungsvariablen

```bash
sudo PROXY_HOST=proxy.example.com PROXY_PORT=8080 ASSUME_YES=1 \
  ./setup-linux-vm.sh
```

### Ohne Proxy (Direktzugriff)

```bash
sudo ./setup-linux-vm.sh --skip-proxy
```

### Distro-/Variant-Override

Autodetect funktioniert normalerweise. Falls nötig, manuell:

```bash
sudo ./setup-linux-vm.sh --server
sudo ./setup-linux-vm.sh --desktop
sudo DISTRO_FAMILY=rhel ./setup-linux-vm.sh    # erzwinge RHEL-Pfad
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

## Distro-spezifische Besonderheiten

### Rocky Linux / Fedora (RHEL-Familie)

- **SSH-Service** heißt `sshd` (nicht `ssh`). Keine Socket-Activation.
- **Firewall** ist `firewalld`. SSH-Service ist standardmäßig schon offen – das Script stellt sicher, dass es bleibt:
  ```
  firewall-cmd --permanent --add-service=ssh
  firewall-cmd --reload
  ```
- **SELinux** ist enforcing. Für die Standard-Operationen (SSH, vmtoolsd) gibt es passende Policies – es sind keine Anpassungen nötig.
- **Rocky 10** hat den Root-Account standardmäßig deaktiviert – das Script nutzt sowieso einen sudo-fähigen User.

### Ubuntu 24.04+ (Debian-Familie)

- **SSH Socket-Activation**: `ssh.socket` lauscht auf Port 22 und startet `ssh.service` bei Bedarf. Das Script enabled beides.
- **UFW** ist häufig installiert aber inaktiv – das Script öffnet Port 22 nur, falls UFW tatsächlich aktiv ist.

## Logging

Alle Aktionen werden in `/var/log/setup-linux-vm.log` protokolliert.

## Idempotenz

Das Script kann mehrfach ausgeführt werden. Originale von `/etc/environment`, `/etc/wgetrc` und `/etc/dnf/dnf.conf` werden beim ersten Lauf als `.orig` gesichert. Proxy-Einträge werden vor dem Schreiben entfernt, sodass keine Duplikate entstehen.

## Nach dem Lauf

```bash
# IP der VM ermitteln
hostname -I

# Per SSH vom Host verbinden
ssh <benutzer>@<vm-ip>

# Status der Dienste (Ubuntu)
systemctl status ssh ssh.socket open-vm-tools

# Status der Dienste (Rocky/Fedora)
systemctl status sshd vmtoolsd
```

### SSH-Härtung

Nach dem ersten Login Umstieg auf SSH-Keys empfohlen:

```bash
# vom Host:
ssh-copy-id <benutzer>@<vm-ip>

# auf der VM:
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
# Ubuntu (24.04+): ssh-Unit reicht, Socket-Activation startet beim nächsten Connect neu
sudo systemctl restart ssh   # Ubuntu
sudo systemctl restart sshd  # Rocky/Fedora
```

## Getestet mit

- Ubuntu 22.04 / 24.04 / 26.04 LTS (Desktop und Server)
- Rocky Linux 9.7 / 10.1 (Workstation und Server)
- Fedora 42 / 43 / 44 (Workstation und Server)
- VMware Workstation 17
- Squid als HTTP-Proxy (automatisierte Tests)

## Lizenz

MIT – siehe [LICENSE](LICENSE).
