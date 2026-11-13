# CookieSuite – Betriebshandbuch

| Angabe | Wert |
| --- | --- |
| Dienst | CookieSuite der CookieCraft AG |
| Stand | 13. November 2026 |
| Zielgruppe | Betrieb, Entwicklung und Personen, die den Dienst übernehmen |
| Produktiv-URL | [CookieSuite](https://cookiecraft.bizfactory.tech) |
| Quellcode | [GitHub-Repository](https://github.com/vanthunder/CookieSuite) |

---

## Inhaltsverzeichnis

1. [Schnellstart: Die wichtigsten Befehle](#1-schnellstart-die-wichtigsten-befehle)
2. [Aktueller Betriebsstand](#2-aktueller-betriebsstand)
3. [Architektur und Datenfluss](#3-architektur-und-datenfluss)
4. [Dateien, Verzeichnisse und Zuständigkeiten](#4-dateien-verzeichnisse-und-zuständigkeiten)
5. [Auf den Server zugreifen](#5-auf-den-server-zugreifen)
6. [Erstinstallation und Konfiguration](#6-erstinstallation-und-konfiguration)
7. [Docker Compose im täglichen Betrieb](#7-docker-compose-im-täglichen-betrieb)
8. [Anwendung sicher aktualisieren](#8-anwendung-sicher-aktualisieren)
9. [PostgreSQL-Backups und Wiederherstellung](#9-postgresql-backups-und-wiederherstellung)
10. [Mailserver: Postfix und lokale E-Mails](#10-mailserver-postfix-und-lokale-e-mails)
11. [Nginx, HTTPS und Zertifikate](#11-nginx-https-und-zertifikate)
12. [Continuous Integration (CI)](#12-continuous-integration-ci)
13. [Automatisierte Releases](#13-automatisierte-releases)
14. [Sicherheit und Benutzerverwaltung](#14-sicherheit-und-benutzerverwaltung)
15. [Logs, Wartung und Fehlerbehebung](#15-logs-wartung-und-fehlerbehebung)
16. [Notfall- und Übergabecheckliste](#16-notfall--und-übergabecheckliste)
17. [Glossar und weiterführende Links](#17-glossar-und-weiterführende-links)

---

## 1. Schnellstart: Die wichtigsten Befehle

Der Produktivserver wird über das Administratorkonto `cookieops` gewartet. Die Anmeldung erfolgt mit einem SSH-Schlüssel. Für Wartungsbefehle mit erhöhten Rechten verwendet das Konto `sudo`. Wer einen Zugang für eine weitere Person oder einen weiteren Rechner einrichtet, beginnt mit [Kapitel 5](#5-auf-den-server-zugreifen).

| Ziel | Befehl bzw. Ort |
| --- | --- |
| Website öffnen | https://cookiecraft.bizfactory.tech |
| Anmeldung über einen *lokal eingerichteten* SSH-Alias | `ssh cookiecraft` |
| Anmeldung ohne Alias mit dem eigenen Schlüssel | [Anmeldebefehl in Abschnitt 5.4](#54-zum-ersten-mal-anmelden-und-rechte-prüfen) |
| In das Anwendungsverzeichnis wechseln | `cd /opt/cookiesuite` |
| Containerstatus | `sudo docker compose ps` |
| App-Logs | `sudo docker compose logs --tail=100 app` |
| Datenbank-Logs | `sudo docker compose logs --tail=100 db` |
| Webserver-Status | `sudo systemctl status nginx` |
| Mailserver-Status | `sudo systemctl status postfix` |
| Firewall-Regeln | `sudo ufw status verbose` |
| HTTPS-Healthcheck | `curl -fsSI https://cookiecraft.bizfactory.tech/up` |
| Backup sofort starten | `sudo /usr/local/sbin/cookiesuite-backup.sh` |
| Backup-Dateien ansehen | `sudo ls -lh /var/backups/cookiesuite/` |
| SMTP-Warteschlange | `sudo postqueue -p` |
| Lokale Admin-Mails lesen | `sudo less /var/mail/cookieadmin` |
| Automatische CI-Tests | https://github.com/vanthunder/CookieSuite/actions |

Vor einem Release den laufenden Stand erfassen, ein Datenbank-Backup erstellen und den Rückweg festlegen. Befehle mit Auswirkungen auf persistente Daten stehen in [Abschnitt 15.3](#153-besonders-vorsichtig-bei-diesen-befehlen).

### Was wird wo ausgeführt?

- **Eigener Rechner (Windows, macOS oder Linux):** SSH-Verbindung und Arbeit am lokalen Git-Checkout.
- **Ubuntu-Host:** Nginx, Certbot, Postfix, UFW, SSH, Backup-Skript und Docker-Engine.
- **App-Container:** PHP, Laravel, Apache und Artisan-Befehle.
- **Datenbank-Container:** PostgreSQL, `pg_dump`, `pg_restore` und Datenbankoperationen.
- **GitHub:** Quellcode, Versionshistorie und automatisierte Tests.

---

## 2. Aktueller Betriebsstand

Diese Übersicht beschreibt den Produktivserver und die verfügbaren Betriebsfunktionen zum oben genannten Stand.

| Bestandteil | Beobachteter Stand | Bedeutung |
| --- | --- | --- |
| Betriebssystem | Ubuntu 24.04 LTS | Produktiv-VM |
| Serverressourcen | 4 vCPU, 8 GB RAM, ca. 160 GB nomineller Plattenspeicher | Einzelner Ubuntu-Host |
| Docker / Compose | Installiert und funktionierend | Ausführung der Container |
| Laravel / Apache | PHP 8.3 im App-Container | Anwendung |
| PostgreSQL | PostgreSQL 16; Healthcheck erfolgreich | Datenbank |
| Datenbankmigrationen | Vier Migrationen als `Ran` bestätigt | Schema vorhanden |
| Reverse Proxy | Host-Nginx auf HTTP/HTTPS | Leitet Anfragen an Docker weiter |
| HTTPS | Let's-Encrypt-Zertifikat; HTTP 301 → HTTPS 200 bestätigt | Verschlüsselte Website |
| SMTP | Postfix auf dem Ubuntu-Host | Annahme von Laravel-Mails |
| Lokale Mailzustellung | Zustellung an `cookieadmin` beim Anlegen eines Kekses geprüft | Benachrichtigung im lokalen Systempostfach |
| Externe Mailzustellung | Empfängerserver haben den direkten Versand abgelehnt | Externer Versand bleibt offen |
| Backups | Manueller Datenbankdump erfolgreich; täglicher Cronjob eingerichtet | Lokale Datensicherung |
| SSH | Schlüsselanmeldung als `cookieops` getestet, SSH-Passwortlogin deaktiviert | Wartungszugang auf Port 22 |
| UFW | Aktiv; öffentlich vorgesehen: 22, 80, 443/TCP | Netzwerkzugriff einschränken |
| GitHub Actions | Workflow auf `main`; grüner Lauf bestätigt | CI |
| Produktiv-Releases | Manueller Rollout nach CI-Prüfung | Ablauf in Kapitel 8 |
| Administratorkonto | `cookieops` mit `sudo` eingerichtet | Erhöhte Rechte bei Bedarf |
| Root-Anmeldung per SSH | `PermitRootLogin no` in der Hauptkonfiguration vor den eingebundenen Dateien. Der bisherige Root-Schlüssel wurde beim Login abgewiesen. | Direkter Root-SSH-Zugang gesperrt |
| Offsite-Backup und Restore-Probe | Ausstehend | Wiederherstellung nach Hostverlust noch nicht erprobt |

---

## 3. Architektur und Datenfluss

### 3.1 Gesamtbild

~~~mermaid
flowchart LR
    browser["Webbrowser"]
    workstation["Admin-Rechner"]

    subgraph host["Ubuntu-Produktivserver"]
        proxy["Nginx · HTTPS mit Let's Encrypt"]
        sshd["OpenSSH · cookieops"]
        postfix["Postfix"]
        mailbox["Lokales Postfach · cookieadmin"]
        cron["Täglicher Backup-Cronjob"]
        backup["Dumps · /var/backups/cookiesuite/"]

        subgraph compose["Docker Compose"]
            app["CookieSuite · Laravel und Apache"]
            db[("PostgreSQL 16 · Volume pgdata")]
        end

        proxy -->|"127.0.0.1:8080"| app
        app -->|"5432 intern"| db
        app -->|"SMTP 25 intern"| postfix --> mailbox
        cron -->|"pg_dump"| db
        cron -->|"schreibt Dump"| backup
    end

    browser -->|"80 leitet auf 443 um"| proxy
    workstation -->|"22 mit SSH-Schlüssel"| sshd
~~~

Im Browser läuft der Zugriff über Nginx zur App. Für die Wartung gibt es einen eigenen SSH-Zugang zum Ubuntu-Host. Quellcode und automatische Tests liegen bei GitHub. Der Ablauf für Produktivupdates steht in [Kapitel 8](#8-anwendung-sicher-aktualisieren).

### 3.2 Zuständigkeiten der Dienste

- **Nginx** nimmt auf dem Ubuntu-Host öffentliche Anfragen an, terminiert HTTPS und leitet sie an den App-Container weiter. Certbot verwaltet die TLS-Zertifikate.
- **Apache/PHP/Laravel** verarbeiten Webanfragen im Anwendungscontainer. Apache verwendet als öffentliches Verzeichnis `/var/www/html/public`.
- **PostgreSQL** ist ein eigener Container. Die Datenbank bekommt keine öffentliche Portfreigabe.
- **Docker Compose** definiert diese Container, das interne Netzwerk, ihre Abhängigkeiten und Volumes.
- **Postfix** läuft auf Ubuntu. Laravel erreicht ihn per SMTP über die private Docker-Bridge.
- **Docker-Volumes** bewahren Daten unabhängig vom Lebenszyklus einzelner Container auf.

### 3.3 Die Ports verstehen

| Port | Wer hört darauf? | Öffentlich? |
| --- | --- | --- |
| `22/tcp` | SSH auf Ubuntu | Ja, per UFW erlaubt; Key-Anmeldung |
| `80/tcp` | Nginx | Ja; HTTP-Weiterleitung und Zertifikatsprüfung |
| `443/tcp` | Nginx | Ja; HTTPS |
| `127.0.0.1:8080` | Weiterleitung an den App-Container | Nur Host-Loopback |
| `5432/tcp` | PostgreSQL | Nur im Docker-Netzwerk |
| `127.0.0.1:25` und `172.18.0.1:25` | Postfix | Nur Host / internes Docker-Netzwerk |

`172.18.0.1` ist die dokumentierte Docker-Gateway-IP. Nach Änderungen am Compose-Netzwerk Gateway, Bridge-Interface, Postfix-Listener, `MAIL_HOST` und UFW-Regel gemeinsam prüfen.

Bei Docker und UFW gelten besondere Regeln: Docker konfiguriert eigene Paketfilter-/NAT-Regeln. Deshalb **nicht allein** aus `ufw status` auf die Erreichbarkeit aller Docker-Ports schließen. Weitere Informationen: https://docs.docker.com/engine/network/packet-filtering-firewalls/

---

## 4. Dateien, Verzeichnisse und Zuständigkeiten

### 4.1 GitHub-Repository

~~~text
CookieSuite/
├── .github/
│   └── workflows/
│       └── ci.yml              # GitHub Actions: automatisierte Tests
├── app/                        # Laravel: Controller, Models, Mail-Klassen
├── bootstrap/                  # Laravel-Bootstrap und Proxy-Vertrauen
├── config/                     # Laravel-Konfigurationsdateien
├── database/
│   └── migrations/             # Datenbankschema und Änderungen
├── docker/
│   └── apache.conf             # Apache VirtualHost im Container
├── public/                     # Öffentlicher Einstiegspunkt, CSS
├── resources/                  # Blade-Templates und Frontend-Ressourcen
├── routes/
│   └── web.php                 # URL-Routen der Anwendung
├── tests/                      # Automatisierte Laravel-Tests
├── .dockerignore               # Nicht ins Image zu kopierende Dateien
├── .env.example                # Vorlage; NICHT die Produktiv-Konfiguration
├── Dockerfile                  # Bauanleitung des PHP/Apache-Images
├── compose.yaml                # App, PostgreSQL, Volumes und Netzwerk
├── composer.json               # PHP-Abhängigkeiten
├── composer.lock               # Festgeschriebene Paketversionen
└── OPERATIONS.md               # Dieses Betriebshandbuch
~~~

### 4.2 Dateien auf Ubuntu, die nicht im Git-Repository liegen

| Pfad | Bedeutung |
| --- | --- |
| `/opt/cookiesuite/` | Ausgechecktes Git-Repository / Produktionsanwendung |
| `/opt/cookiesuite/.env` | Produktive Konfiguration inklusive Geheimnissen |
| `/etc/nginx/sites-available/cookiesuite` | Nginx-Konfiguration |
| `/etc/nginx/sites-enabled/cookiesuite` | Aktivierung der Nginx-Site |
| `/etc/letsencrypt/live/cookiecraft.bizfactory.tech/` | Zertifikate und private TLS-Schlüssel |
| `/etc/postfix/main.cf` | Postfix-Konfiguration |
| `/var/mail/cookieadmin` | Lokales Administrator-Mailpostfach |
| `/usr/local/sbin/cookiesuite-backup.sh` | Manuelles/automatisches Backup-Skript |
| `/etc/cron.d/cookiesuite-backup` | Zeitplan des Backup-Jobs |
| `/var/backups/cookiesuite/` | Lokale PostgreSQL-Sicherungen |
| `/var/log/cookiesuite-backup.log` | Ausgaben des Backup-Cronjobs |
| `/etc/ssh/sshd_config` und `/etc/ssh/sshd_config.d/00-cookiecraft.conf` | SSH-Regeln. Bei Änderungen die wirksamen Werte mit `sudo sshd -T` prüfen. |
| `/home/cookieops/.ssh/authorized_keys` | Freigegebene öffentliche Schlüssel für `cookieops` |

**Wichtig:** Git alleine sichert **nicht** diese Host-Dateien oder die produktiven Docker-Volumes. Bei einem Neuaufbau müssen sie separat wiederhergestellt beziehungsweise neu erzeugt werden.

---

## 5. Auf den Server zugreifen

### 5.1 Zugang für einen weiteren Rechner oder eine neue Person

SSH öffnet ein verschlüsseltes Terminal auf dem Ubuntu-Produktivserver. Der Server ist über `cookiecraft.bizfactory.tech` auf Port `22` erreichbar. Für den vorhandenen Betriebszugang ist `cookieops` das SSH-Konto. `cookieadmin` empfängt lokale Benachrichtigungsmails.

| Situation | Konto auf dem Server | Schlüssel |
| --- | --- | --- |
| Dieselbe Person nutzt einen weiteren Rechner | Das vorhandene Konto, zum Beispiel `cookieops` | Auf dem weiteren Rechner einen eigenen Schlüssel erzeugen |
| Eine weitere Person übernimmt Betriebsaufgaben | Ein persönliches Linux-Konto für diese Person | Auf ihrem Rechner einen eigenen Schlüssel erzeugen. `sudo` nur für Aufgaben mit erhöhten Rechten vergeben. |

~~~mermaid
sequenceDiagram
    participant PC as Neuer Rechner
    participant Admin as Bestehender Admin
    participant Host as Ubuntu-Server
    PC->>PC: SSH-Schlüsselpaar erzeugen
    PC->>Admin: Öffentlichen Schlüssel (.pub) übergeben
    Admin->>Host: Schlüssel dem richtigen Konto zuordnen
    Admin->>PC: Server-Fingerabdruck separat mitteilen
    PC->>Host: Fingerabdruck prüfen und per SSH anmelden
    Host-->>PC: Sitzung unter dem eigenen Linux-Konto
~~~

### 5.2 Auf dem eigenen Rechner einen Schlüssel erzeugen

Öffne **PowerShell** unter Windows oder das **Terminal** unter macOS oder Linux. `ssh -V` zeigt, ob der OpenSSH-Client verfügbar ist. Fehlt er unter Windows, installiere die optionale Funktion **OpenSSH-Client**. [Microsoft-Anleitung zu OpenSSH](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse)

**Windows (PowerShell)**

~~~powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.ssh" | Out-Null
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\cookiecraft_ed25519"
Get-Content "$env:USERPROFILE\.ssh\cookiecraft_ed25519.pub"
~~~

**macOS und Linux (Terminal)**

~~~bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t ed25519 -f ~/.ssh/cookiecraft_ed25519
cat ~/.ssh/cookiecraft_ed25519.pub
~~~

Wähle beim Erstellen eine Passphrase. Falls `ssh-keygen` fragt, ob es eine vorhandene Datei überschreiben soll, brich ab und verwende einen freien Dateinamen. Passe den Namen anschließend auch im Anmeldebefehl und im Alias an.

Gib die `.pub`-Datei oder deren vollständige `ssh-ed25519`-Zeile über den vereinbarten internen Kanal an den zuständigen Admin weiter. Gib dabei das gewünschte Konto und den Rechner an. Die Datei **ohne `.pub`** ist dein privater Schlüssel und bleibt auf dem Gerät.

### 5.3 Auf dem Server den öffentlichen Schlüssel freigeben

Diesen Schritt führt ein Admin in einer bestehenden Ubuntu-Sitzung aus. Für eine weitere Person zuerst ein persönliches Konto anlegen. `anna` steht hier für den vereinbarten Linux-Benutzernamen.

~~~bash
sudo adduser anna
~~~

`adduser` fragt nach einem Passwort für das Linux-Konto. Es kann später für `sudo` gebraucht werden. Die SSH-Anmeldung erfolgt weiterhin mit dem Geräteschlüssel.

Wenn diese Person administrative Aufgaben übernimmt, gib ihr die dafür nötigen Rechte. Auf diesem Ubuntu-Host gewährt die Gruppe `sudo` weitreichende Administrationsrechte.

~~~bash
sudo usermod -aG sudo anna
~~~

Bei einem weiteren Rechner für dasselbe Konto direkt hier beginnen. Setze `account` auf das Zielkonto, etwa `anna` für die neue Person oder `cookieops` für den bestehenden Zugang. Der Befehl liest das Home-Verzeichnis des Kontos aus Ubuntu aus.

~~~bash
account=cookieops
account_home=$(getent passwd "$account" | cut -d: -f6)

if [ -n "$account_home" ] && [ -d "$account_home" ]; then
    key_file="$account_home/.ssh/authorized_keys"
    sudo install -d -m 700 -o "$account" -g "$(id -gn "$account")" "$account_home/.ssh" &&
        sudo -u "$account" touch "$key_file" &&
        sudo -u "$account" chmod 600 "$key_file" &&
        sudo -u "$account" nano "$key_file"
else
    printf 'Home-Verzeichnis für %s nicht gefunden. Benutzerkonto prüfen.\n' "$account"
fi
~~~

Hänge den neuen öffentlichen Schlüssel als eigene Zeile an und erhalte alle vorhandenen Einträge. In `nano` speicherst du mit `Strg+O`, bestätigst mit `Enter` und beendest mit `Strg+X`. Mit `sudo ssh-keygen -lf "$key_file"` lassen sich die eingetragenen Fingerabdrücke prüfen. Die Person am neuen Rechner kann den Fingerabdruck ihrer `.pub`-Datei ebenfalls mit `ssh-keygen -lf` prüfen. Halte die bestehende Sitzung offen, bis die Anmeldung vom neuen Rechner funktioniert.

### 5.4 Zum ersten Mal anmelden und Rechte prüfen

Vor der ersten Verbindung holst du den **ED25519-Fingerabdruck des Servers** beim zuständigen Admin ein. Dieser liest ihn über eine bereits verifizierte SSH-Sitzung oder die Anbieter-Konsole aus und teilt ihn über den vereinbarten internen Kanal mit.

~~~bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
~~~

Führe dann auf dem eigenen Rechner den Anmeldebefehl aus. Ersetze `cookieops` durch deinen Benutzernamen, falls dir ein persönliches Konto zugeteilt wurde. OpenSSH löst `~` in diesem Befehl auf allen drei Betriebssystemen zum eigenen Benutzerordner auf.

~~~shell
ssh -i ~/.ssh/cookiecraft_ed25519 -o IdentitiesOnly=yes -o HostKeyAlgorithms=ssh-ed25519 cookieops@cookiecraft.bizfactory.tech
~~~

Vergleiche beim ersten Verbindungsaufbau den angezeigten `SHA256:…`-Wert mit dem mitgeteilten Fingerabdruck. Bestätige mit `yes`, wenn beide übereinstimmen. Bei einem abweichenden oder plötzlich geänderten Fingerabdruck brich ab und lass den zuständigen Admin den Host überprüfen.

Führe in der neuen Sitzung `whoami` aus. Bei einem Administratorkonto prüfst du außerdem `sudo whoami`. Die erwarteten Ausgaben sind der eigene Linux-Benutzername und `root` für den zweiten Befehl. Die **Schlüssel-Passphrase** entsperrt den privaten Schlüssel auf deinem Rechner. `sudo` kann separat nach dem **Passwort deines Linux-Kontos** fragen. `exit` beendet die Sitzung.

### 5.5 Einen Kurzbefehl für weitere Anmeldungen einrichten

Lege auf dem eigenen Rechner eine Datei `config` im Ordner `.ssh` an oder öffne die bestehende. Unter Windows ist das `%USERPROFILE%\.ssh\config`, unter macOS und Linux `~/.ssh/config`. Speichere sie ohne Dateiendung. Verwende für `Host cookiecraft` genau einen Block.

~~~sshconfig
Host cookiecraft
    HostName cookiecraft.bizfactory.tech
    User cookieops
    IdentityFile ~/.ssh/cookiecraft_ed25519
    IdentitiesOnly yes
    HostKeyAlgorithms ssh-ed25519
    StrictHostKeyChecking ask
~~~

Passe `User` und `IdentityFile` an deinen Zugang an. Danach genügt `ssh cookiecraft` auf diesem Rechner. `ssh -G cookiecraft` zeigt, welcher Benutzername, Zielhost und Schlüsselpfad für den Alias wirksam sind. Falls ein SSH-Agent bereits viele Schlüssel lädt und der Server `Too many authentication failures` meldet, kannst du in diesem Alias zusätzlich `IdentityAgent none` setzen.

### 5.6 Zugang widerrufen oder Fehler eingrenzen

| Meldung oder Anlass | Vorgehen |
| --- | --- |
| `Permission denied (publickey)` | Zielkonto, ausgewählten Schlüssel und dessen Eintrag in `authorized_keys` prüfen. `.ssh` hat Modus `700`, `authorized_keys` Modus `600`. Beides gehört dem Zielkonto. |
| `Connection timed out` | Zieladresse und Erreichbarkeit von Port `22` prüfen. Falls nötig, die Anbieter-Konsole zur Diagnose verwenden. |
| Rechner verloren oder ausgemustert | Den Fingerabdruck des Geräteschlüssels zuordnen und genau dessen Zeile aus `authorized_keys` entfernen. Das sperrt neue Verbindungen mit diesem Schlüssel. Bei einem Verlust laufende Sitzungen mit dem zuständigen Admin prüfen. |
| Person scheidet aus | Persönlichen Zugang und gegebenenfalls `sudo`-Rechte entziehen. Laufende Sitzungen und weitere Berechtigungen prüfen. |

---

## 6. Erstinstallation und Konfiguration

Diese Schritte gelten für einen neuen Host oder einen Wiederaufbau. Für ein Release auf dem laufenden System gilt [Kapitel 8](#8-anwendung-sicher-aktualisieren).

### 6.1 Voraussetzungen

- Ubuntu-Host mit funktionierendem SSH und Internetzugang;
- Git, Docker Engine und Docker Compose Plugin;
- Nginx, Certbot und Postfix für die Host-Dienste;
- DNS-A-Record von `cookiecraft.bizfactory.tech` auf die Produktiv-IP;
- ausreichend Speicherplatz und eine vorher festgelegte Backup-Strategie.

Prüfen:

~~~bash
git --version
docker --version
docker compose version
df -h
~~~

### 6.2 Repository herunterladen

~~~bash
sudo mkdir -p /opt
cd /opt
sudo git clone https://github.com/vanthunder/CookieSuite.git cookiesuite
cd /opt/cookiesuite
~~~

Falls das Repository bereits existiert, prüfe in `/opt/cookiesuite` zuerst `sudo git status` und `sudo git remote -v`. Der produktive Checkout gehört `root`.

### 6.3 Produktive `.env`-Datei

~~~bash
cd /opt/cookiesuite
sudo cp -n .env.example .env
sudo chmod 600 .env
sudo nano .env
~~~

Die Vorlage `.env.example` enthält **Entwicklungswerte**. Insbesondere sind dort PostgreSQL-Port und Datenbankhost nicht für Docker Compose vorkonfiguriert.

**Beispiel – Werte ohne Geheimnisse:**

~~~dotenv
APP_NAME=CookieSuite
APP_ENV=production
APP_DEBUG=false
APP_URL=https://cookiecraft.bizfactory.tech

DB_CONNECTION=pgsql
DB_HOST=db
DB_PORT=5432
DB_DATABASE=cookiesuite
DB_USERNAME=cookiesuite
DB_PASSWORD=<GEHEIMES-ZUFAELLIGES-PASSWORT>

APP_KEY=base64:<BESTEHENDER-ODER-NEU-ERZEUGTER-APP-KEY>
SESSION_SECURE_COOKIE=true

MAIL_MAILER=smtp
MAIL_HOST=172.18.0.1
MAIL_PORT=25
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
MAIL_FROM_NAME="CookieSuite"
MAIL_FROM_ADDRESS=<FREIGEGEBENER-ABSENDER>
ADMIN_MAIL=cookieadmin@localhost.localdomain
~~~

**Diese Datei niemals ins Git-Repository committen.** Die produktive `.env` liegt nur auf dem Server. Der konkrete `MAIL_HOST` muss zum aktuellen Docker-Gateway passen.

Ein neues Datenbankpasswort lässt sich beispielsweise mit `openssl rand -hex 24` erzeugen. Ein Laravel-App-Key braucht 32 zufällige Bytes, als Base64 codiert (Format `base64:...`).

**Sehr wichtig:** Einen vorhandenen `APP_KEY` **nicht** bei normalen Deployments verändern. Er kann zur Ver- und Entschlüsselung vorhandener Anwendungsdaten und Cookies benötigt werden.

### 6.4 Docker-Container erstmalig starten

~~~bash
cd /opt/cookiesuite
sudo docker compose config -q
sudo docker compose up -d --build
sudo docker compose ps
sudo docker compose exec app php artisan migrate --force
~~~

Was geschieht?

1. Compose liest `compose.yaml` und die Umgebungsvariablen.
2. Docker baut das PHP-/Apache-Image anhand des `Dockerfile`.
3. PostgreSQL startet und absolviert seinen Healthcheck.
4. Laravel startet.
5. Die `migrate`-Anweisung erstellt die in den Laravel-Migrations definierten Tabellen.

`--force` ist erforderlich, weil Laravel im Produktionsmodus vor destruktionsrelevanten Operationen Rückfragen stellt.

### 6.5 Inbetriebnahme prüfen

~~~bash
sudo docker compose ps
sudo docker compose exec app php artisan migrate:status
curl -fsSI http://127.0.0.1:8080/up
curl -fsSI https://cookiecraft.bizfactory.tech/up
~~~

Öffne die Startseite im Browser. Prüfe bei der Inbetriebnahme mit einem eindeutig gekennzeichneten Testdatensatz das Anlegen, Anzeigen, Bearbeiten und Löschen sowie die lokale Benachrichtigung. Entferne den Testdatensatz nach der Prüfung.

---

## 7. Docker Compose im täglichen Betrieb

### 7.1 Nützliche Befehle

Zuerst ins Projektverzeichnis:

~~~bash
cd /opt/cookiesuite
~~~

| Aktion | Befehl |
| --- | --- |
| Status | `sudo docker compose ps` |
| Live-Logs | `sudo docker compose logs -f --tail=100 app` |
| Datenbank-Logs | `sudo docker compose logs --tail=100 db` |
| Nur App neu starten | `sudo docker compose restart app` |
| Gestoppte Container starten | `sudo docker compose up -d` |
| App neu bauen und starten | `sudo docker compose up -d --build app` |
| Migrationsstatus | `sudo docker compose exec app php artisan migrate:status` |
| Laravel-Cache leeren | `sudo docker compose exec app php artisan config:clear` |
| PostgreSQL-Verbindung prüfen | `sudo docker compose exec db pg_isready -U cookiesuite -d cookiesuite` |

### 7.2 Was bedeutet das alles?

- **Image:** Vorlage mit Anwendungscode und Laufzeitumgebung.
- **Container:** laufende Instanz des Images.
- **Service:** von Compose verwalteter Bestandteil, z. B. `app` oder `db`.
- **Volume:** dauerhaftes Datenverzeichnis, das Containerneustarts überlebt.
- **Healthcheck:** automatischer Test, ob ein Dienst eine definierte Grundfunktion erfüllt.
- **Restart Policy:** `restart: unless-stopped` versucht einen unerwartet beendeten Container automatisch wieder zu starten.

### 7.3 Welche Volumes existieren?

In `compose.yaml` sind zwei benannte Volumes definiert:

- `pgdata` – PostgreSQL-Daten und Datenbankdateien;
- `app_storage` – beschreibbarer Laravel-Storage-Bereich.

Der tatsächliche Docker-Volume-Name kann wegen des Compose-Projektnamens einen Präfix besitzen. Daher zunächst nachsehen:

~~~bash
sudo docker volume ls
sudo docker compose config --services
~~~

**Kein `docker compose down -v` für normale Wartung verwenden.** Schon ein gewöhnlicher `down` beendet die Dienste; mit `-v` würden zusätzlich die definierten Volumes gelöscht.

---

## 8. Anwendung sicher aktualisieren

### 8.1 Releaseablauf

~~~mermaid
flowchart LR
    code["Commit auf main"] --> ci["GitHub Actions"]
    ci --> release["Commit prüfen und freigeben"]
    release --> backup["Datenbank sichern"]
    backup --> rollout["Image bauen und App aktualisieren"]
    rollout --> check["Migration und Funktionstest"]
~~~

**Wichtig:** Aktuell ist das Ausrollen **manuell**. Die CI läuft automatisch; Continuous Deployment wurde bislang nicht eingerichtet.

### 8.2 Release vorbereiten

1. Geplante Änderung und Git-Commit identifizieren.
2. Prüfen, ob der CI-Lauf für genau diesen Commit erfolgreich war.
3. Backup erstellen und Sicherung auf Lesbarkeit prüfen.
4. Entscheiden, ob neue Datenbankmigrationen einen Rollback erschweren.
5. Wartungsfenster und Ansprechpartner klären, wenn die Anwendung genutzt wird.

### 8.3 Release manuell ausrollen

Halte den aktuellen Commit für die Rückkehr fest. Hole `main` und ermittle den Release-Commit. `FETCH_HEAD` verweist auf den letzten Git-Abruf in diesem Checkout. Führe während des Rollouts keine parallelen Git-Abrufe oder Deployments auf dem Host aus.

~~~bash
cd /opt/cookiesuite

sudo git status --short
sudo git rev-parse --short HEAD
sudo git fetch origin main && sudo git rev-parse FETCH_HEAD
~~~

Wenn `git status` lokale Änderungen zeigt, kläre zuerst, ob diese gesichert oder übernommen werden müssen. Übernimm den vollständigen SHA des freigegebenen Commits aus dem zugehörigen GitHub-Actions-Lauf in `release_commit`. Prüfe vor dem Rollout die Änderungen mit `sudo git log --oneline HEAD.."$release_commit"` und den Migrationsplan. Für Änderungen, die mit der laufenden App-Version inkompatibel sind, plane eine Schreibpause und die Reihenfolge des Rollouts separat.

~~~bash
release_commit=HIER_DEN_FREIGEGEBENEN_COMMIT_SHA_EINTRAGEN
cd /opt/cookiesuite &&
    working_tree=$(sudo git status --porcelain) &&
    test -z "$working_tree" &&
    sudo git merge-base --is-ancestor "$release_commit" FETCH_HEAD &&
    sudo git merge-base --is-ancestor HEAD "$release_commit" &&
    sudo /usr/local/sbin/cookiesuite-backup.sh &&
    sudo git merge --ff-only "$release_commit" &&
    sudo docker compose build app &&
    sudo docker compose up -d --no-deps app &&
    sudo docker compose exec -T app php artisan migrate --force &&
    sudo docker compose ps &&
    curl -fsSI https://cookiecraft.bizfactory.tech/up
~~~

Bei einem Fehler stoppt die Kette. Ermittle dann den erreichten Stand anhand von Git-Revision, Containerstatus, Logs und `migrate:status`. Prüfe nach einem erfolgreichen Rollout Startseite und Datenbankzugriff. Prüfe Schreibaktionen und die lokale Benachrichtigung mit einem gekennzeichneten Testdatensatz, wenn das Release diese Pfade ändert.

### 8.4 Wenn ein Update fehlschlägt: Rollback

1. Keine weiteren Deployments starten.
2. Logs lesen und Auswirkungen prüfen.
3. Vorherigen funktionierenden Commit oder Image-Stand identifizieren.
4. Nach Freigabe den funktionierenden Stand erneut deployen.
5. Healthcheck und Anwendung testen.

~~~bash
cd /opt/cookiesuite
sudo git log --oneline -n 10
sudo docker compose logs --tail=100 app
~~~

**Nicht automatisch `php artisan migrate:rollback` auf Produktion ausführen!** Datenbankänderungen können Daten löschen oder nicht rückwärtskompatibel sein. Bei einem Datenbankschema-Problem braucht es einen eigens geplanten Rückweg und ggf. ein Restore.

Im [Zielbild für den automatisierten Rollout](#13-automatisierte-releases) wird die vorherige Image-Version über ihren Digest eindeutig referenziert.

---

## 9. PostgreSQL-Backups und Wiederherstellung

### 9.1 Warum ein Backup trotz Docker-Volume?

Ein Volume schützt vor Datenverlust beim Neuerstellen eines Containers, **nicht** vor defekten Daten, versehentlichem Löschen oder vollständigem Serverausfall. Deshalb brauchen wir zusätzliche Backups.

### 9.2 Derzeitiger Backup-Mechanismus

- Skript: `/usr/local/sbin/cookiesuite-backup.sh`
- Zielverzeichnis: `/var/backups/cookiesuite/`
- Dateiformat: PostgreSQL `pg_dump -Fc`, Endung `.dump`
- Zeitplan: täglich **02:15 Uhr in der lokalen Server-Zeitzone**
- Cron-Datei: `/etc/cron.d/cookiesuite-backup`
- Lokale Aufbewahrung: ältere Dump-Dateien nach etwa sieben Tagen löschen
- Sicherheitsrechte: Dump-Dateien root-lesbar (`0600`)

Der erste manuelle Dump wurde erfolgreich erzeugt. Die Cron-Ausführung wurde konfiguriert; regelmäßige Laufkontrolle gehört zum Betrieb.

~~~cron
15 2 * * * root /usr/local/sbin/cookiesuite-backup.sh >> /var/log/cookiesuite-backup.log 2>&1
~~~

Die fünf Cron-Zeitfelder sind: **Minute, Stunde, Tag des Monats, Monat, Wochentag**. Der zusätzliche Eintrag `root` gibt bei einer Datei in `/etc/cron.d/` den ausführenden Systembenutzer an.

### 9.3 Backup manuell starten und prüfen

~~~bash
sudo /usr/local/sbin/cookiesuite-backup.sh
sudo ls -lh /var/backups/cookiesuite/
sudo tail -n 50 /var/log/cookiesuite-backup.log
sudo systemctl is-active cron
timedatectl
~~~

**Warum `pg_dump -Fc`?** Das PostgreSQL-Custom-Format ist für `pg_restore` geeignet. Mit `pg_restore --list` kann man die Archivstruktur lesen. Eine solche Prüfung ist **nicht** dasselbe wie ein kompletter Wiederherstellungstest.

### 9.4 Vorhandene Sicherung kontrollieren

~~~bash
cd /opt/cookiesuite

BACKUP=$(sudo find /var/backups/cookiesuite -maxdepth 1 \
  -type f -name 'cookiesuite-*.dump' | sort | tail -n 1)

test -n "$BACKUP" || { echo "Kein Backup gefunden"; exit 1; }

sudo cat "$BACKUP" | sudo docker compose exec -T db pg_restore --list | head -n 20
~~~

### 9.5 Wiederherstellung in einer separaten Datenbank testen

Führe die Restore-Probe bevorzugt in einer getrennten Umgebung mit PostgreSQL 16 aus. Die folgenden Befehle setzen dort denselben Compose-Dienst `db` und einen bereitgestellten Dump unter `/var/backups/cookiesuite/` voraus. Der Import startet erst, wenn die Datenbank `cookiesuite_restore_test` erfolgreich neu angelegt wurde.

~~~bash
(
    set -euo pipefail
    cd /opt/cookiesuite
    backup=$(sudo find /var/backups/cookiesuite -maxdepth 1 \
        -type f -name 'cookiesuite-*.dump' | sort | tail -n 1)
    test -n "$backup" || { printf 'Kein Dump gefunden.\n' >&2; exit 1; }

    sudo docker compose exec -T db sh -c \
        'createdb -U "$POSTGRES_USER" cookiesuite_restore_test'
    sudo cat "$backup" | sudo docker compose exec -T db sh -c \
        'pg_restore --exit-on-error --no-owner -U "$POSTGRES_USER" -d cookiesuite_restore_test'
    sudo docker compose exec -T db sh -c \
        'psql -v ON_ERROR_STOP=1 -U "$POSTGRES_USER" -d cookiesuite_restore_test -c "SELECT COUNT(*) FROM cookies;"'
)
~~~

Dokumentiere Dump-Datei, Zeitpunkt und Ergebnis der Datenprüfung. Entferne die Testdatenbank nach Abschluss der Probe, sobald sie für die Auswertung entbehrlich ist.

~~~bash
sudo docker compose exec -T db sh -c \
  'dropdb -U "$POSTGRES_USER" cookiesuite_restore_test'
~~~

Ein Restore in die **echte** Produktionsdatenbank ist eine separate, potenziell destruktive Notfallmaßnahme: Anwendung stoppen bzw. Schreibzugriffe verhindern, Wiederherstellungsziel und Verantwortliche bestätigen, vorhandenen Zustand sichern und Restore im Wartungsfenster durchführen.

### 9.6 Offene Maßnahmen für die Datensicherung

**Noch offen:**

- verschlüsselte Offsite-Backups auf ein getrenntes System / S3-kompatiblen Speicher;
- getrennte Zugriffsrechte und definierte Aufbewahrung (z. B. täglich, wöchentlich, monatlich);
- automatische Benachrichtigung, wenn ein Backup fehlschlägt;
- regelmäßiger dokumentierter Test einer vollständigen Wiederherstellung;
- Sicherung von `.env`, Nginx-, Postfix-, SSH-, Cron- und Zertifikatskonfigurationen in einem autorisierten, verschlüsselten Admin-Backup (nicht im öffentlichen Repo).

---

## 10. Mailserver: Postfix und lokale E-Mails

### 10.1 SMTP auf dem Ubuntu-Host

Postfix läuft **direkt auf dem Ubuntu-Host**. Laravel übergibt Nachrichten per SMTP über das private Docker-Netzwerk. Der Host hat aktuell für Postfix die Listener:

- `127.0.0.1:25` – lokale Prozesse auf Ubuntu;
- `172.18.0.1:25` – Laravel-Container über die Docker-Bridge.

Die UFW-Regel für SMTP ist auf das passende interne Docker-Interface und Quell-Subnetz begrenzt. Es gibt **keine öffentliche Freigabe** für TCP 25.

### 10.2 Was passiert beim Anlegen eines Kekses?

1. Benutzer öffnet das Formular in CookieSuite.
2. Laravel validiert den Inhalt und schreibt den Datensatz nach PostgreSQL.
3. Der `CookieController` ruft die Laravel-Mailfunktion auf.
4. Symfony/Laravel übergibt die Nachricht an Postfix.
5. Postfix stellt die Nachricht an das lokale Postfach `cookieadmin` zu.

Der relevante Anwendungscode liegt in `app/Http/Controllers/CookieController.php` und `app/Mail/CookieCreated.php`. Empfänger ist die Konfigurationsvariable `ADMIN_MAIL`.

Laravel speichert den Keks vor dem Mailversand. Tritt beim Absenden ein SMTP-Fehler auf, prüfe zuerst die Keksübersicht und die App-Logs. Ein erneutes Absenden kann einen weiteren Datensatz anlegen.

### 10.3 Lokale E-Mails lesen

~~~bash
# Lokales Postfach prüfen
sudo ls -lh /var/mail/cookieadmin

# Letzte Zeilen einer Nachricht anschauen
sudo tail -n 80 /var/mail/cookieadmin

# Postfach komplett durchblättern (q zum Beenden)
sudo less /var/mail/cookieadmin

# Mail-Warteschlange
sudo postqueue -p

# Postfix-Zustelllogs
sudo journalctl -t postfix/local --since "30 minutes ago" --no-pager
sudo journalctl -t postfix/smtp --since "30 minutes ago" --no-pager
~~~

Das lokale Admin-Postfach ist ein **Systempostfach auf dem Server** und nicht automatisch in einem Outlook-, Gmail- oder T-Online-Posteingang sichtbar.

### 10.4 SMTP-Konfiguration kontrollieren

~~~bash
sudo systemctl is-active postfix
sudo postconf inet_interfaces mynetworks mydestination relayhost
sudo ss -ltnp | grep ':25 '
sudo ufw status verbose
~~~

Erwartung: Nur `127.0.0.1:25` und die aktuelle private Docker-Gateway-IP; **nicht** `0.0.0.0:25`.

Die SMTP-Verbindung von Laravel zu Postfix nutzt auf der privaten Docker-Bridge kein STARTTLS. Bei einer Verlagerung des Maildienstes in ein anderes Netz muss die SMTP-Verbindung verschlüsselt und die Zertifikatsprüfung eingerichtet werden.

### 10.5 Externe Zustellung

Der aktive Benachrichtigungsweg führt in das lokale Systempostfach. Bei Tests des direkten externen Versands war TCP 25 erreichbar. Microsoft antwortete mit `421`. T-Online wies die Absender-IP mit `554` ab.

Für externe Empfänger einen freigegebenen SMTP-Relay und eine Absenderdomain festlegen. Die zur Versandroute passenden SPF-, DKIM- und DMARC-Einträge mit der zuständigen Stelle abstimmen. Bei direktem Versand gehört auch der PTR-Eintrag der sendenden IP dazu. Vor der Umstellung von `ADMIN_MAIL` die Zustellung an ein freigegebenes externes Testpostfach prüfen.

### 10.6 Fehlerbilder

| Symptom | Mögliche Ursache | Erste Prüfung |
| --- | --- | --- |
| Laravel meldet SMTP-Verbindungsfehler | Postfix nicht erreichbar / Firewall / Gateway geändert | `systemctl status postfix`, `ss -ltnp`, Docker-Netz |
| STARTTLS-Zertifikatsfehler | Interner SMTP-Dienst bietet Zertifikat, das nicht zum Hostnamen passt | Postfix-/Laravel-Konfiguration und Vertrauensmodell prüfen |
| Mail bleibt in Queue | Remote-Server hat Nachricht nicht angenommen | `postqueue -p`, `journalctl -t postfix/smtp` |
| Lokale Mail fehlt | Lokaler Nutzer/Empfängerdomain nicht richtig konfiguriert | `postconf mydestination`, `id cookieadmin`, `postfix/local`-Logs |

Wenn eine eindeutig identifizierte Nachricht entfernt werden muss, verwende ihre Queue-ID mit `sudo postsuper -d QUEUE_ID`. Dabei wird die Nachricht endgültig aus der Warteschlange gelöscht.

---

## 11. Nginx, HTTPS und Zertifikate

### 11.1 Aufgabe des Reverse Proxys

Nginx nimmt Anfragen über `cookiecraft.bizfactory.tech` an und leitet sie an `http://127.0.0.1:8080` weiter. Dort lauscht der Apache-Server **im App-Container**. So bleibt der PHP-/Apache-Port selbst nicht öffentlich.

Nginx ist mit dem Certbot-Nginx-Plugin und Let's Encrypt für HTTPS eingerichtet; HTTP leitet auf HTTPS weiter.

### 11.2 Wichtige Prüfkommandos

~~~bash
sudo nginx -t
sudo systemctl status nginx --no-pager
curl -I http://cookiecraft.bizfactory.tech
curl -I https://cookiecraft.bizfactory.tech
sudo certbot certificates
sudo certbot renew --dry-run
~~~

Erwartung:

- Auf HTTP kommt ein `301` oder `308` mit `Location: https://...`.
- Auf HTTPS antwortet die Anwendung, typischerweise mit `200 OK`.
- Das Zertifikat ist gültig und der automatische Erneuerungstest erfolgreich.

### 11.3 Zertifikatsverlängerung

Let's-Encrypt-Zertifikate laufen regelmäßig ab. Certbot verwendet üblicherweise einen systemd-Timer oder Cronjob zum Erneuern. Prüfen:

~~~bash
systemctl list-timers --all | grep -i certbot || true
sudo certbot renew --dry-run
~~~

Die Zertifikate und Schlüssel liegen unter `/etc/letsencrypt/`. Den **privaten Schlüssel niemals ins Repository kopieren**.

### 11.4 Nginx-Konfiguration ändern

~~~bash
sudo nano /etc/nginx/sites-available/cookiesuite
sudo nginx -t && sudo systemctl reload nginx
~~~

Nur nach erfolgreichem `nginx -t` neu laden. Bei Fehlern bleibt die bisherige funktionierende Konfiguration möglichst unverändert.

Laravel vertraut über `trustProxies(at: '*')` den Proxy-Headern. Der App-Port ist auf Host-Loopback gebunden, Nginx ist der öffentliche Einstiegspunkt. Vor einer Änderung dieses Netzwerkpfads die von Nginx gesetzten Weiterleitungsheader prüfen und das Vertrauen in Laravel auf die vorgesehenen Proxy-Adressen begrenzen.

---

## 12. Continuous Integration (CI)

### 12.1 CI einfach erklärt

**Continuous Integration** bedeutet: Bei Änderungen am Repository werden automatisch Qualitätsprüfungen ausgeführt. Statt dass jeder Entwickler alles manuell auf seinem PC testet, nutzt GitHub eine frische, reproduzierbare Testumgebung.

**Vorhandene Datei:** `.github/workflows/ci.yml`  
**GitHub Actions:** https://github.com/vanthunder/CookieSuite/actions

Für ein Release den Status der Workflow-Ausführung für den freizugebenden Commit in GitHub Actions prüfen.

### 12.2 Wann startet er?

- bei Push auf `main`;
- bei Pull Requests in Richtung `main`;
- manuell über `Run workflow`.

### 12.3 Was wird automatisch geprüft?

| Schritt | Zweck |
| --- | --- |
| Checkout | Quellcode laden |
| PHP 8.3 | Passende PHP-Laufzeit bereitstellen |
| PostgreSQL 16 Service | Temporäre Testdatenbank starten |
| Composer validate | Projekt-/Paketdefinition prüfen |
| Composer install | PHP-Abhängigkeiten installieren |
| Laravel `APP_KEY` | Testinstanz initialisieren |
| `php artisan migrate --force` | Datenbankschema für Tests anlegen |
| `php artisan test` | PHPUnit-/Laravel-Tests ausführen |

Die Datenbank und die Test-Umgebungsvariablen von GitHub Actions sind **nicht** die Produktionsdatenbank. Der Mailer ist in CI auf `array` gesetzt; Tests verschicken keine realen SMTP-Nachrichten.

### 12.4 Testabdeckung

Die CI prüft Composer, die Datenbankmigrationen und den HTTP-Status der Startseite. Anlegen, Bearbeiten und Löschen von Keksen, Formularvalidierung und SMTP-Zustellung sind bislang nicht automatisiert abgedeckt. Änderungen an diesen Pfaden erfordern einen Funktionstest nach Abschnitt 8.3.

### 12.5 Was mache ich bei einem roten Lauf?

1. GitHub → Repository → `Actions` öffnen.
2. Den fehlgeschlagenen Workflow auswählen.
3. Den roten Job und Schritt anklicken.
4. Den **ersten eigentlichen Fehler** im Log lesen (nicht nur den letzten Folgefehler).
5. Änderung lokal korrigieren, committen und pushen.
6. Erneuten Lauf überprüfen.

**CI ist nicht gleich Deployment:** Der vorhandene Workflow startet oder aktualisiert derzeit **keinen** Container auf dem Produktivserver.

---

## 13. Automatisierte Releases

Der automatische Rollout mit manueller Freigabe ist als nächster Ausbauschritt vorgesehen. Für laufende Releases gilt der [manuelle Ablauf](#83-release-manuell-ausrollen).

~~~mermaid
flowchart LR
    ci["CI erfolgreich"] --> image["Image bauen und in Registry veröffentlichen"]
    image --> staging["Image in Testumgebung prüfen"]
    staging --> approval["Produktionsfreigabe"]
    approval --> backup["Backup und Migrationsprüfung"]
    backup --> rollout["Geprüftes Image ausrollen"]
    rollout --> health{"Anwendung erreichbar und funktionsfähig?"}
    health -->|Ja| complete["Release dokumentieren"]
    health -->|Nein| rollback["Alarm auslösen und Rückweg prüfen"]
~~~

- Einmal pro Release bauen und dasselbe Image in Test und Produktion über seinen Digest referenzieren. Den Digest dem Git-Commit zuordnen.
- CI und Rollout getrennt ausführen. Den Produktivzugang auf den Deployment-Job begrenzen und die Freigabe im GitHub-Environment `production` abbilden.
- Vor Migrationen eine lesbare Sicherung und einen erprobten Wiederherstellungsweg vorhalten. Den vorherigen Image-Digest sichern und die Kompatibilität mit dem Datenbankschema prüfen.
- Nach dem Rollout `/up` und die betroffenen Anwendungspfade prüfen. Bei einem Fehler den Status melden und den Rückweg anhand des erreichten Migrationsstands wählen.

[GitHub-Dokumentation zu Deployment-Umgebungen](https://docs.github.com/en/actions/concepts/workflows-and-actions/deployment-environments)

---

## 14. Sicherheit und Benutzerverwaltung

### 14.1 Was bereits eingerichtet wurde

- SSH-Schlüsselanmeldung über `cookieops` mit `sudo` bei Bedarf;
- SSH-Passwortlogin und direkte Root-Anmeldung deaktiviert;
- UFW aktiv mit `deny incoming`;
- öffentlich nur SSH/HTTP/HTTPS vorgesehen;
- PostgreSQL nur im internen Docker-Netz;
- Postfix nur auf Loopback und privatem Docker-Gateway;
- `.env` ausschließlich auf Ubuntu mit eingeschränkten Dateirechten;
- Laravel `APP_DEBUG=false` und `SESSION_SECURE_COOKIE=true`;
- öffentliches Webroot von Apache ist nur `public/`;
- Nginx bedient HTTPS über Let's Encrypt.

### 14.2 SSH-Regeln prüfen und sicher ändern

`cookieops` verwendet für Wartung `sudo docker compose …`. Die Gruppe `docker` gewährt über den Docker-Socket praktisch dieselben Rechte wie `root` und ist für diesen Zugang nicht erforderlich.

In `/etc/ssh/sshd_config` steht `PermitRootLogin no` vor `Include`. Für globale Einstellungen verwendet OpenSSH den zuerst gelesenen Wert. Weitere Einträge in der Hauptdatei oder in `sshd_config.d/` können abweichen. Prüfe deshalb die wirksamen Werte mit `sudo sshd -T`. [OpenSSH-Serverkonfiguration](https://man.openbsd.org/sshd_config)

Halte bei einer Änderung die bestehende Admin-Sitzung offen und stelle sicher, dass die Anbieter-Konsole als Rückweg verfügbar ist. Prüfe die Syntax und lade die Konfiguration nur bei erfolgreicher Prüfung neu.

~~~bash
sudo sshd -t && sudo systemctl reload ssh
sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication) '
~~~

Erwartet werden `permitrootlogin no`, `passwordauthentication no`, `kbdinteractiveauthentication no` und `pubkeyauthentication yes`. Prüfe SSH-Login und `sudo whoami` in einem zweiten Terminal, bevor du die bisherige Sitzung beendest.

### 14.3 Firewall überprüfen

~~~bash
sudo ufw status verbose
sudo sshd -T | grep -E 'port |pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication|permitrootlogin'
sudo ss -ltnp
sudo docker compose ps
~~~

UFW verhindert nicht automatisch jede durch Docker selbst veröffentlichte Verbindung; bei sicherheitskritischen Änderungen aus einem **externen Netzwerk** prüfen, welche Ports tatsächlich erreichbar sind.

### 14.4 Wartung / Sicherheitsupdates

Sicherheitsupdates im regelmäßigen Wartungsfenster prüfen und einspielen.

1. Vorab Wartungsfenster und ggf. Snapshot/Offsite-Backup organisieren.
2. Paketänderungen ansehen: `sudo apt update` und `apt list --upgradable`.
3. Sicherheitsupdates kontrolliert einspielen.
4. Gegebenenfalls geplanten Neustart durchführen.
5. Danach SSH, Docker, Nginx, Postfix, Datenbank und HTTPS prüfen.

**Keine ungeplante Distribution-Aktualisierung** während eines produktiven Geschäftsbetriebs. Ein Upgrade zwischen Ubuntu-LTS-Versionen benötigt Vorbereitung und einen Rückfallplan.

### 14.5 Geheimnisse und Berechtigungen

**Niemals veröffentlichen:** SSH-Privatschlüssel, Datenbankpasswörter, `APP_KEY`, `.env`, SMTP-Zugangsdaten, TLS-Privatschlüssel, GitHub-Tokens, vollständige produktive Logs mit personenbezogenen Daten.

Bei automatisiertem Deployment: möglichst kurzlebige Berechtigungen, minimale Rechte und eine klare Trennung von CI-Test- und Produktions-Secrets verwenden.

---

## 15. Logs, Wartung und Fehlerbehebung

### 15.1 Die wichtigsten Diagnosebefehle

~~~bash
cd /opt/cookiesuite

# Container
sudo docker compose ps
sudo docker compose logs --tail=100 app db

# Laravel
sudo docker compose exec app php artisan migrate:status

# Webserver / TLS
sudo nginx -t
sudo systemctl is-active nginx
curl -I https://cookiecraft.bizfactory.tech
sudo certbot certificates

# Mail / Datenbank
sudo systemctl is-active postfix
sudo postqueue -p
sudo docker compose exec db pg_isready -U cookiesuite -d cookiesuite

# System
df -h
free -h
sudo ufw status verbose
~~~

### 15.2 Typische Fehler

| Problem | Wahrscheinliche Ursachen | Erste Maßnahme |
| --- | --- | --- |
| Website nicht erreichbar | Nginx aus, DNS falsch, TLS-Problem | `systemctl status nginx`, `sudo nginx -t`, `curl -I` |
| HTTP 502 Bad Gateway | App-Container nicht erreichbar | `sudo docker compose ps`, App-Logs, Port 8080 |
| HTTP 500 Server Error | Laravel-Konfiguration, Rechte, DB oder Codefehler | App-Logs, `migrate:status` |
| Datenbankverbindung fehlgeschlagen | DB-Container nicht Healthy, falscher Host/Port | Compose/DB-Logs, DB_HOST=`db`, Port `5432` |
| Nach Deploy fehlendes Feature | Altes Image, Migration fehlend | Commit-ID, Build, Migrationsstatus |
| Kein SMTP-Versand | Falsches Gateway, Postfix gestoppt, falscher Empfänger | `postconf`, Postfix-Logs, Queue |
| Kein Backup | Cron ausgefallen, Script-Fehler, Volume/Disk voll | Cron-Status, Backup-Log, `df -h` |
| Browser meldet TLS-Fehler | Zertifikat abgelaufen / Domain stimmt nicht | `certbot certificates`, `renew --dry-run` |
| SSH-Anmeldung gesperrt | Falscher Key, Rechte, Firewall, SSH-Regeln | Bestehende Sitzung / Anbieter-Konsole verwenden |

### 15.3 Besonders vorsichtig bei diesen Befehlen

| Befehl | Risiko |
| --- | --- |
| `docker compose down -v` | Löscht benannte Compose-Volumes |
| `docker system prune --volumes` | Kann nicht benutzte persistente Daten entfernen |
| `git reset --hard` | Verwirft lokale Änderungen |
| `php artisan migrate:fresh` | Erstellt Datenbankschema neu und löscht Tabellen |
| `php artisan migrate:rollback` | Kann Produktionsdaten bzw. Schema zurückrollen |
| `ufw reset` | Entfernt Firewall-Regeln |
| `postsuper -d ALL` | Löscht alle Nachrichten aus der Postfix-Queue |

**Grundsatz:** Bei Unklarheit erst `status`, `logs` und Konfiguration prüfen – nicht durch Löschen „reparieren“.

---

## 16. Notfall- und Übergabecheckliste

### 16.1 Wenn die Website ausfällt

1. Umfang feststellen: DNS, HTTPS, Nginx, Docker oder Datenbank?
2. Sicherstellen, dass SSH-Zugriff besteht.
3. In `/opt/cookiesuite` mit `sudo docker compose ps` und `sudo docker compose logs --tail=100 app db` den Containerzustand prüfen.
4. `sudo nginx -t` und den HTTPS-Healthcheck ausführen.
5. Bei Datenbankfehlern **keine Daten löschen**.
6. Letztes funktionierendes Deployment und aktuelles Backup identifizieren.
7. Fehler dokumentieren, Verantwortliche informieren, kontrolliert beheben.
8. Nach der Wiederherstellung HTTPS, `/up`, die Startseite und die vom Ausfall betroffenen Funktionen prüfen.

### 16.2 Regelmäßige Betriebsaufgaben

| Häufigkeit | Aufgabe |
| --- | --- |
| Nach jedem Deployment | HTTPS, `/up`, CRUD-Test, Logs, Migrationsstatus |
| Täglich | Backup-Erfolg, Disk, Container-Health und Mail-Queue |
| Wöchentlich | Sicherheitsupdates ansehen, Log-/Speicherwachstum prüfen |
| Monatlich | Restore-Probe, Zugriffsrechte, Zertifikatserneuerung und Patch-Plan |
| Nach Benutzerwechsel | SSH-Schlüssel und Berechtigungen aktualisieren |
| Bei Incident | Zeitpunkt, Auswirkung, Ursache, Fix und Vorbeugung dokumentieren |

### 16.3 Was bei der Übergabe bereitliegen muss

- aktueller Link zum GitHub-Repository und zur CI-Pipeline;
- [SSH-Zugangsprozess](#5-auf-den-server-zugreifen) und interner Kontakt für die Freigabe neuer öffentlicher Schlüssel;
- zuständiger Admin, verifizierter Server-Fingerabdruck und Notfallzugang über die Anbieter-Konsole;
- Standort der produktiven `.env` und sichere Übergabe der Secrets;
- Zugriff auf Host-/Nginx-/Postfix-/Docker-Konfigurationen;
- Backup-Speicher und bestätigter Restore-Prozess;
- Ansprechpartner für DNS, Domain, Zertifikate und Mail-Reputation;
- Liste offener Sicherheits- und Betriebsaufgaben.

### 16.4 Offene Betriebsmaßnahmen

| Bereich | Nächster Schritt |
| --- | --- |
| Datensicherung | Verschlüsselte Kopie auf getrenntem Speicher ablegen und einen vollständigen Restore dokumentieren. |
| Überwachung | Externen HTTPS-Check, Speicher- und Backup-Überwachung einrichten. Alarmweg und Empfänger mit einem Testalarm prüfen. |
| Externe Benachrichtigungen | Freigegebenen SMTP-Relay und Absenderdomain festlegen und die Zustellung prüfen. |
| Automatisierte Releases | Versionierte Images mit Freigabe, Funktionstest und Rückweg einführen. |

---

## 17. Glossar und weiterführende Links

| Begriff | Einfach erklärt |
| --- | --- |
| **SSH** | Verschlüsselte Fernanmeldung auf einem Server |
| **SSH-Key** | Schlüsselpaar zur Anmeldung ohne Serverpasswort |
| **sudo** | Erlaubt ausgewählte administrative Befehle mit erhöhten Rechten |
| **Dockerfile** | Bauanleitung für ein Docker-Image |
| **Image** | Vorlage, aus der Container gestartet werden |
| **Container** | Isolierte laufende Anwendung |
| **Docker Compose** | Konfiguration für mehrere gemeinsam betriebene Container |
| **Volume** | Dauerhafter Speicher außerhalb des kurzlebigen Container-Dateisystems |
| **Reverse Proxy** | Server, der Anfragen an interne Anwendungen weiterleitet |
| **TLS / HTTPS** | Verschlüsselte Verbindung zwischen Browser und Webserver |
| **Let's Encrypt** | Zertifizierungsstelle für kostenlose TLS-Zertifikate |
| **PostgreSQL** | Relationale SQL-Datenbank |
| **Migration** | Versionierte Änderung am Datenbankschema |
| **SMTP / Postfix** | Mail-Transport-Protokoll und Mailserver-Software |
| **MX** | DNS-Eintrag, der die Mailserver einer Empfängerdomain benennt |
| **SPF / DKIM / PTR** | Wichtige Mechanismen zur Prüfung von Mail-Absendern |
| **Cronjob** | Zeitgesteuerter Linux-Auftrag |
| **CI** | Automatische Tests und Qualitätsprüfungen bei Codeänderungen |
| **CD** | Automatisierte Vorbereitung oder Auslieferung geprüfter Software |
| **Rollback** | Gezielte Rückkehr zu einer zuvor funktionierenden Version |
| **Healthcheck** | Regelmäßiger Test, ob ein Dienst grundsätzlich funktioniert |

**Referenzen:**

- Repository: https://github.com/vanthunder/CookieSuite
- GitHub Actions: https://github.com/vanthunder/CookieSuite/actions
- Laravel 11 Deployment: https://laravel.com/docs/11.x/deployment
- Docker Compose: https://docs.docker.com/compose/
- Docker-Firewall-Regeln: https://docs.docker.com/engine/network/packet-filtering-firewalls/
- PostgreSQL 16 `pg_dump`: https://www.postgresql.org/docs/16/app-pgdump.html
- PostgreSQL 16 `pg_restore`: https://www.postgresql.org/docs/16/app-pgrestore.html
- GitHub Deployment Environments: https://docs.github.com/en/actions/concepts/workflows-and-actions/deployment-environments
- Certbot: https://certbot.eff.org/
