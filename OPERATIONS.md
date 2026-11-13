# CookieSuite – Betriebshandbuch (Operations Guide)

> **Stand:** 13. November 2026 · **Projekt:** CookieCraft / DevOps-Challenge  
> **Zielgruppe:** neue Administratorinnen und Administratoren, Entwicklerinnen und Entwickler sowie Personen ohne tiefere Linux-/Docker-Erfahrung  
> **Quellcode:** https://github.com/vanthunder/CookieSuite  
> **Produktiv-URL:** https://cookiecraft.bizfactory.tech  
> **Wichtig:** Diese Datei ist für ein **öffentliches Repository** geschrieben. Passwörter, private Schlüssel, produktive `.env`-Dateien und persönliche Zugangsdaten dürfen nicht eingecheckt werden.

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
13. [Continuous Delivery / Deployment (CD)](#13-continuous-delivery--deployment-cd)
14. [Sicherheit und Benutzerverwaltung](#14-sicherheit-und-benutzerverwaltung)
15. [Logs, Wartung und Fehlerbehebung](#15-logs-wartung-und-fehlerbehebung)
16. [Notfall- und Übergabecheckliste](#16-notfall--und-übergabecheckliste)
17. [Glossar und weiterführende Links](#17-glossar-und-weiterführende-links)

---

## 1. Schnellstart: Die wichtigsten Befehle

**Diese Anleitung geht davon aus, dass der spätere Administrator einen SSH-Key und `sudo`-Rechte besitzt.** Im derzeitigen Challenge-Aufbau wird noch ein Root-SSH-Konto verwendet; der dedizierte Administrator `cookieops` ist **geplant, aber noch nicht eingerichtet**.

| Ziel | Befehl bzw. Ort |
| --- | --- |
| Website öffnen | https://cookiecraft.bizfactory.tech |
| Anmeldung über einen *lokal eingerichteten* SSH-Alias | `ssh cookiecraft` |
| Anmeldung ohne Alias, nach Einrichtung des neuen Benutzers | `ssh cookieops@cookiecraft.bizfactory.tech` |
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

**Goldene Regel:** Vor einem riskanten Update zuerst den Zustand prüfen, ein Datenbank-Backup erstellen und überlegen, wie das Update zurückgenommen werden kann. Die Befehle `docker compose down -v` und `docker volume rm` **nicht** für gewöhnliche Updates verwenden: Sie können gespeicherte Daten löschen.

### Was wird wo ausgeführt?

- **Windows-PC:** Programmierung mit VS Code, GitHub Desktop, SSH-Verbindung.
- **Ubuntu-Host:** Nginx, Certbot, Postfix, UFW, SSH, Backup-Skript und Docker-Engine.
- **App-Container:** PHP, Laravel, Apache und Artisan-Befehle.
- **Datenbank-Container:** PostgreSQL, `pg_dump`, `pg_restore` und Datenbankoperationen.
- **GitHub:** Quellcode, Versionshistorie und automatisierte Tests.

---

## 2. Aktueller Betriebsstand

Die folgenden Angaben dokumentieren den **zuletzt während der Challenge geprüften Stand**. Sie ersetzen keine neue Prüfung nach späteren Änderungen.

| Bestandteil | Beobachteter Stand | Bedeutung |
| --- | --- | --- |
| Betriebssystem | Ubuntu 24.04 LTS | Produktiv-VM |
| Serverressourcen | 4 vCPU, 8 GB RAM, ca. 160 GB nomineller Speicher | Ausreichend für die Demo |
| Docker / Compose | Installiert und funktionierend | Ausführung der Container |
| Laravel / Apache | PHP 8.3 im App-Container | Anwendung |
| PostgreSQL | PostgreSQL 16; Healthcheck erfolgreich | Datenbank |
| Datenbankmigrationen | Vier Migrationen als `Ran` bestätigt | Schema vorhanden |
| Reverse Proxy | Host-Nginx auf HTTP/HTTPS | Leitet Anfragen an Docker weiter |
| HTTPS | Let's-Encrypt-Zertifikat; HTTP 301 → HTTPS 200 bestätigt | Verschlüsselte Website |
| SMTP | Postfix auf dem Ubuntu-Host | Annahme von Laravel-Mails |
| Lokale Mailzustellung | Funktioniert laut abgeschlossenem Funktionstest | Demonstrationsbetrieb |
| Externe Mailzustellung | **Nicht zuverlässig funktionsfähig** | Empfänger lehnten die Cloud-Absender-IP ab |
| Backups | Manueller Datenbankdump erfolgreich; täglicher Cronjob eingerichtet | Lokale Datensicherung |
| SSH | Schlüsselanmeldung eingerichtet, Passwortlogin deaktiviert | Administrationszugang |
| UFW | Aktiv; öffentlich vorgesehen: 22, 80, 443/TCP | Netzwerkzugriff einschränken |
| GitHub Actions | Workflow auf `main`; grüner Lauf bestätigt | CI |
| Continuous Deployment | **Nur als Konzept vorhanden** | Noch kein automatischer Produktiv-Rollout |
| Eigener Admin-Benutzer | **Noch nicht angelegt** | Nächster Sicherheits-Schritt |
| Offsite-Backup / Restore-Probe | **Noch nicht nachgewiesen** | Offener Produktionspunkt |
| Kubernetes | Getrennte Challenge; nicht Teil dieses Produktivsystems | Noch separat bearbeiten |

**Wichtige Einschränkung beim SMTP-Häkchen:** Die lokale Zustellung beweist, dass Laravel → Postfix → lokales Admin-Postfach funktioniert. Das ist **nicht gleichbedeutend** mit erfolgreichem Versand an beliebige externe Adressen.

---

## 3. Architektur und Datenfluss

### 3.1 Gesamtbild

~~~text
                  Nutzer / Mitarbeiter
                          |
          http://cookiecraft.bizfactory.tech
          https://cookiecraft.bizfactory.tech
                          |
                          v
                 NGINX (Ubuntu-Host)
               HTTP :80 / HTTPS :443
                 TLS via Let's Encrypt
                          |
                  127.0.0.1:8080
                          |
                          v
                +-------------------+
                | DOCKER COMPOSE    |
                |                   |
                | app               |
                | PHP 8.3 + Apache  |
                | Laravel CookieSuite|
                |        |          |
                |        +-------> db (PostgreSQL 16)
                |        |          |  Port 5432 intern
                |        |          +-- Volume: pgdata
                |        |
                |        +-------> Host-Gateway:25
                |                   |
                +-------------------+
                                    |
                             Postfix (Host)
                              lokale Mailbox
                                    |
                           /var/mail/cookieadmin

Ubuntu-Host zusätzlich:
- UFW-Firewall / SSH
- Certbot / Nginx
- Backup-Cronjob → /var/backups/cookiesuite/
- Docker-Volumes für Daten

GitHub:
- vanthunder/CookieSuite
- GitHub Actions CI (Tests)
- zukünftig: Versionierung von Images und CD
~~~

### 3.2 Warum verwenden wir getrennte Dienste?

- **Nginx** nimmt öffentlich zugängliche Anfragen an und kümmert sich um TLS-Zertifikate. Er läuft bewusst **außerhalb von Docker**, weil dies Teil der Challenge ist.
- **Apache/PHP/Laravel** verarbeiten Webanfragen im Anwendungscontainer. Apache verwendet als öffentliches Verzeichnis `/var/www/html/public`.
- **PostgreSQL** ist ein eigener Container. Die Datenbank bekommt keine öffentliche Portfreigabe.
- **Docker Compose** definiert diese Container, das interne Netzwerk, ihre Abhängigkeiten und Volumes.
- **Postfix** ist ein echter SMTP-Server auf Ubuntu. Laravel sendet an ihn über die private Docker-Bridge, nicht über einen öffentlichen Mailport.
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

**Achtung:** `172.18.0.1` ist die während der Challenge beobachtete Docker-Gateway-IP. Docker-Netzwerke können sich ändern. Nach einer Netzwerkkonfigurationsänderung immer IP, Interface und UFW-Regel erneut prüfen.

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
| `/etc/ssh/sshd_config.d/00-cookiecraft.conf` | SSH-Härtung |
| `/root/.ssh/authorized_keys` | Derzeitiger öffentlicher Root-SSH-Key-Zugang |

**Wichtig:** Git alleine sichert **nicht** diese Host-Dateien oder die produktiven Docker-Volumes. Bei einem Neuaufbau müssen sie separat wiederhergestellt beziehungsweise neu erzeugt werden.

---

## 5. Auf den Server zugreifen

### 5.1 Anmeldung mit SSH

SSH ist ein verschlüsseltes Protokoll, über das man einen Linux-Server aus der Ferne administrieren kann.

Auf dem eingerichteten Windows-PC funktioniert derzeit:

~~~powershell
ssh cookiecraft
~~~

`cookiecraft` ist ein **lokaler SSH-Alias** in `%USERPROFILE%\.ssh\config`, kein weltweit gültiger Hostname. Andere Administratoren benötigen einen eigenen SSH-Key und eine eigene Konfiguration.

Nach Einrichtung des künftigen Benutzers `cookieops` kann die eigene Windows-Konfiguration ungefähr so aussehen:

~~~sshconfig
Host cookiecraft
    HostName cookiecraft.bizfactory.tech
    User cookieops
    IdentityFile ~/.ssh/cookiecraft_ed25519
    IdentitiesOnly yes
    IdentityAgent none
    StrictHostKeyChecking ask
~~~

Der tatsächlich verwendete private Schlüssel kann anders heißen. Bei einem neuen Server oder geänderten Host-Key **immer den Fingerprint aus vertrauenswürdiger Quelle prüfen**, statt Warnungen blind zu bestätigen.

### 5.2 Was ist ein SSH-Key?

Ein SSH-Schlüsselpaar besteht aus:

- **privatem Schlüssel:** bleibt ausschließlich auf dem eigenen PC und darf nicht weitergegeben werden;
- **öffentlichem Schlüssel:** steht auf dem Server in der Datei `authorized_keys`.

Der Server erlaubt die Anmeldung nur, wenn der passende private Schlüssel nachgewiesen wird. Es wurde konfiguriert, dass Passwort-Anmeldungen über SSH nicht erlaubt sind.

### 5.3 Aktueller und zukünftiger Benutzer

**Aktuell:** In der Challenge wurde mit `root` gearbeitet. Die Schlüsselanmeldung ist eingerichtet und ein Passwortlogin deaktiviert.

**Geplant:** Ein eigener Admin-Benutzer `cookieops` mit `sudo`-Rechten. Dieser ist **noch nicht** angelegt. Er ist nicht derselbe Benutzer wie `cookieadmin`, der nur lokale E-Mails empfängt.

**Sichere Reihenfolge beim späteren Wechsel:**

1. Vorhandene SSH-Sitzung offen lassen.
2. `cookieops` anlegen und `sudo`-Rechte gezielt vergeben.
3. Öffentlichen SSH-Key für `cookieops` hinterlegen, Dateiberechtigungen kontrollieren.
4. In einer **neuen** Verbindung SSH-Key-Login und `sudo` testen.
5. Erst danach `PermitRootLogin no` setzen und SSH-Konfiguration neu laden.
6. Erneut eine neue Verbindung testen, bevor die alte geschlossen wird.

**Keinen Benutzer vorschnell zur Gruppe `docker` hinzufügen:** Wer direkten Zugriff auf den Docker-Socket hat, kann damit praktisch Host-Root-Rechte erlangen. Für Wartung lieber kontrolliert `sudo docker compose ...` verwenden.

---

## 6. Erstinstallation und Konfiguration

Dieser Abschnitt hilft bei einem **frischen, freigegebenen Server**. Er ist **keine Aufforderung**, die bereits laufende Produktivinstallation neu aufzusetzen.

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

Falls das Repository bereits existiert, **nicht nochmals in dasselbe Verzeichnis klonen**, sondern zunächst `git status` und `git remote -v` prüfen.

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

### 6.5 Ersten Erfolg prüfen

~~~bash
sudo docker compose ps
sudo docker compose exec app php artisan migrate:status
curl -fsSI http://127.0.0.1:8080/up
curl -fsSI https://cookiecraft.bizfactory.tech/up
~~~

Im Browser anschließend **einen Testkeks erstellen, anzeigen und bearbeiten**. Ein HTTP-200 beweist zunächst nur, dass der Server antwortet; es ersetzt keinen Anwendungstest.

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

### 8.1 Überblick: Wie kommt neuer Code auf den Server?

~~~text
VS Code / lokale Entwicklung
        |
        v
GitHub Desktop → Commit → Push
        |
        v
GitHub-Fork / Branch main
        |
        v
GitHub Actions → Composer / Migration / Laravel-Tests
        |
        v
Freigegebener Code → Git Pull auf Ubuntu
        |
        v
Docker-Image neu bauen → Container aktualisieren
        |
        v
Datenbankmigrationen → Healthcheck → Funktionstest
~~~

**Wichtig:** Aktuell ist das Ausrollen **manuell**. Die CI läuft automatisch; Continuous Deployment wurde bislang nicht eingerichtet.

### 8.2 Checkliste VOR einem Update

1. Geplante Änderung und Git-Commit identifizieren.
2. Prüfen, ob der CI-Lauf grün ist.
3. Backup erstellen und Sicherung auf Lesbarkeit prüfen.
4. Entscheiden, ob neue Datenbankmigrationen einen Rollback erschweren.
5. Wartungsfenster und Ansprechpartner klären, wenn die Anwendung genutzt wird.

### 8.3 Konkreter manueller Update-Ablauf

~~~bash
cd /opt/cookiesuite

# 1. Bestehenden Stand prüfen
sudo git status --short
sudo git rev-parse --short HEAD

# 2. Sicherheitskopie der Datenbank
sudo /usr/local/sbin/cookiesuite-backup.sh

# 3. Neue Commits holen und ansehen
sudo git fetch origin main
sudo git log --oneline HEAD..origin/main

# 4. Nur bei geprüftem, sauberem Stand aktualisieren
sudo git pull --ff-only origin main

# 5. Image neu bauen und App aktualisieren
sudo docker compose build app
sudo docker compose up -d --no-deps app

# 6. Datenbankschema aktualisieren
sudo docker compose exec app php artisan migrate --force

# 7. Technische Kontrolle
sudo docker compose ps
curl -fsSI https://cookiecraft.bizfactory.tech/up
~~~

Danach **im Browser** die wichtigsten Funktionen testen: Startseite, Keks anlegen, speichern, bearbeiten und lokaler Mailversand.

Falls `git status --short` Änderungen zeigt: **nicht blind `git reset --hard` ausführen**. Erst herausfinden, wem die Änderungen gehören.

### 8.4 Wenn ein Update fehlschlägt: Rollback

Für einen kleinen Demo-Server ist die praktikable Erstmaßnahme:

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

Für einen späteren echten CD-Prozess sollten Images mit Commit-SHA oder Digest unveränderlich versioniert werden. Dann ist ein Application-Rollback wesentlich einfacher als bei einem erneuten lokalen Build.

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

### 9.5 Wiederherstellungstest in eine SEPARATE Datenbank

**Dieser Test darf die echte Produktionsdatenbank nicht überschreiben.** Die folgenden Befehle erzeugen die zusätzliche Datenbank `cookiesuite_restore_test`. Vorher sicherstellen, dass dieser Name noch nicht verwendet wird.

~~~bash
cd /opt/cookiesuite

BACKUP=$(sudo find /var/backups/cookiesuite -maxdepth  \
  -type f -name 'cookiesuite-*.dump' | sort | tail -n 1)

test -n "$BACKUP" || { echo "Kein Backup gefunden"; exit 1; }

# Eigene leere Testdatenbank erstellen
sudo docker compose exec -T db sh -c \
  'createdb -U "$POSTGRES_USER" cookiesuite_restore_test'

# Archiv in die TESTDATENBANK importieren
set -o pipefail
sudo cat "$BACKUP" | sudo docker compose exec -T db sh -c \
  'pg_restore --exit-on-error --no-owner -U "$POSTGRES_USER" -d cookiesuite_restore_test'

# Beispieldaten prüfen
sudo docker compose exec -T db sh -c \
  'psql -U "$POSTGRES_USER" -d cookiesuite_restore_test -c "SELECT COUNT(*) FROM cookies;"'
~~~

Erst nachdem man das Ergebnis geprüft und **sicher ist, dass die Testdatenbank nicht gebraucht wird**, kann man sie entfernen:

~~~bash
sudo docker compose exec -T db sh -c \
  'dropdb -U "$POSTGRES_USER" cookiesuite_restore_test'
~~~

Ein Restore in die **echte** Produktionsdatenbank ist eine separate, potenziell destruktive Notfallmaßnahme: Anwendung stoppen bzw. Schreibzugriffe verhindern, Wiederherstellungsziel und Verantwortliche bestätigen, vorhandenen Zustand sichern und Restore im Wartungsfenster durchführen.

### 9.6 Was gehört noch zu einem professionellen Backup-Konzept?

**Noch offen:**

- verschlüsselte Offsite-Backups auf ein getrenntes System / S3-kompatiblen Speicher;
- getrennte Zugriffsrechte und definierte Aufbewahrung (z. B. täglich, wöchentlich, monatlich);
- automatische Benachrichtigung, wenn ein Backup fehlschlägt;
- regelmäßiger dokumentierter Test einer vollständigen Wiederherstellung;
- Sicherung von `.env`, Nginx-, Postfix-, SSH-, Cron- und Zertifikatskonfigurationen in einem autorisierten, verschlüsselten Admin-Backup (nicht im öffentlichen Repo).

---

## 10. Mailserver: Postfix und lokale E-Mails

### 10.1 Was wurde eingerichtet?

Postfix läuft **direkt auf dem Ubuntu-Host**. Laravel übergibt Nachrichten per SMTP über das private Docker-Netzwerk. Der Host hat aktuell für Postfix die Listener:

- `127.0.0.1:25` – lokale Prozesse auf Ubuntu;
- `172.18.0.1:25` – Laravel-Container über die Docker-Bridge.

Die UFW-Regel für SMTP ist auf das passende interne Docker-Interface und Quell-Subnetz begrenzt. Es gibt **keine öffentliche Freigabe** für TCP 25.

### 10.2 Was passiert beim Anlegen eines Kekses?

1. Benutzer öffnet das Formular in CookieSuite.
2. Laravel validiert den Inhalt und schreibt den Datensatz nach PostgreSQL.
3. Der `CookieController` ruft die Laravel-Mailfunktion auf.
4. Symfony/Laravel übergibt die Nachricht an Postfix.
5. Postfix nimmt die E-Mail an und liefert sie – in der Demo – an ein lokales Admin-Postfach aus.

Der relevante Anwendungscode liegt in `app/Http/Controllers/CookieController.php` und `app/Mail/CookieCreated.php`. Empfänger ist die Konfigurationsvariable `ADMIN_MAIL`.

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

Die SMTP-Verbindung von Laravel zu Postfix nutzt im aktuellen internen Demo-Setup keine STARTTLS-Aushandlung. Das ist nur vertretbar, wenn der Transportpfad auf das vertrauenswürdige lokale Netz beschränkt bleibt. Für andere Netzwerke ist eine angemessene TLS-Konfiguration erforderlich.

### 10.5 Warum funktionieren externe Mails noch nicht zuverlässig?

Während der Challenge wurde der direkte SMTP-Versand auf Port 25 getestet; IPv4 und IPv6 konnten externe SMTP-Server erreichen.

Trotzdem ist **Netzwerk-Erreichbarkeit keine Zustellgarantie**:

- Ein Microsoft-Empfänger meldete vorübergehend eine `421`-Ablehnung.
- T-Online wies die Quell-IP mit `554 ... Dialup/transient IP not allowed` zurück.

Eine funktionierende lokale E-Mail wird deshalb nicht als gelungene externe Zustellung ausgegeben. Für echten Produktivbetrieb braucht CookieCraft eine **freigegebene Absenderdomain**, passende SPF-/DKIM- und PTR-/Reverse-DNS-Einträge und ggf. einen **vom Unternehmen freigegebenen** SMTP-Relay. **Keine privaten E-Mail-Zugangsdaten** auf dem fremden Challenge-Server hinterlegen.

### 10.6 Fehlerbilder

| Symptom | Mögliche Ursache | Erste Prüfung |
| --- | --- | --- |
| Laravel meldet SMTP-Verbindungsfehler | Postfix nicht erreichbar / Firewall / Gateway geändert | `systemctl status postfix`, `ss -ltnp`, Docker-Netz |
| STARTTLS-Zertifikatsfehler | Interner SMTP-Dienst bietet Zertifikat, das nicht zum Hostnamen passt | Postfix-/Laravel-Konfiguration und Vertrauensmodell prüfen |
| Mail bleibt in Queue | Remote-Server hat Nachricht nicht angenommen | `postqueue -p`, `journalctl -t postfix/smtp` |
| Lokale Mail fehlt | Lokaler Nutzer/Empfängerdomain nicht richtig konfiguriert | `postconf mydestination`, `id cookieadmin`, `postfix/local`-Logs |

**Nur gezielt** eine bekannte Testmail anhand ihrer Queue-ID mit `postsuper -d QUEUE_ID` löschen. Nicht pauschal die Warteschlange leeren.

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
sudo nginx -t
sudo systemctl reload nginx
~~~

Nur nach erfolgreichem `nginx -t` neu laden. Bei Fehlern bleibt die bisherige funktionierende Konfiguration möglichst unverändert.

Laravel vertraut in `bootstrap/app.php` derzeit Proxy-Headern über `trustProxies(at: '*')`. Das funktioniert im kontrollierten Demo-Setup, sollte aber beim späteren Netzwerkausbau auf konkret vertrauenswürdige Proxy-Adressen begrenzt werden.

---

## 12. Continuous Integration (CI)

### 12.1 CI einfach erklärt

**Continuous Integration** bedeutet: Bei Änderungen am Repository werden automatisch Qualitätsprüfungen ausgeführt. Statt dass jeder Entwickler alles manuell auf seinem PC testet, nutzt GitHub eine frische, reproduzierbare Testumgebung.

**Vorhandene Datei:** `.github/workflows/ci.yml`  
**GitHub Actions:** https://github.com/vanthunder/CookieSuite/actions

Der Workflow wurde auf dem Branch `main` eingerichtet. Ein erfolgreicher (grüner) Lauf wurde während der Challenge bestätigt.

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

### 12.4 Was bedeutet „grüner Lauf“?

Die **konfigurierten** Schritte haben den Test bestanden. Das beweist nicht, dass jede Funktion getestet ist. Zusätzliche sinnvolle Tests wären:

- Keks erstellen, bearbeiten und löschen;
- Validierungsfehler (fehlender Name, ungültige Menge);
- ausgelöste Admin-Mail mit Laravel `Mail::fake()` nachweisen;
- PHP-Formatierung und statische Analyse;
- Docker-Image in CI bauen und einen Smoke-Test durchführen.

### 12.5 Was mache ich bei einem roten Lauf?

1. GitHub → Repository → `Actions` öffnen.
2. Den fehlgeschlagenen Workflow auswählen.
3. Den roten Job und Schritt anklicken.
4. Den **ersten eigentlichen Fehler** im Log lesen (nicht nur den letzten Folgefehler).
5. Änderung lokal korrigieren, committen und pushen.
6. Erneuten Lauf überprüfen.

**CI ist nicht gleich Deployment:** Der vorhandene Workflow startet oder aktualisiert derzeit **keinen** Container auf dem Produktivserver.

---

## 13. Continuous Delivery / Deployment (CD)

### 13.1 Unterschied zwischen CI, Delivery und Deployment

- **CI:** Code automatisch testen und prüfen.
- **Continuous Delivery:** erfolgreich geprüfter Stand steht jederzeit für eine **bewusste Freigabe** bereit.
- **Continuous Deployment:** erfolgreich geprüfter Stand wird **automatisch ohne manuellen Freigabeschritt** ausgerollt.

Für CookieCraft ist bisher **CI implementiert**. Das nachfolgende CD-Verfahren ist ein **Konzept, noch nicht produktiv eingerichtet**.

### 13.2 Vorschlag für einen sicheren CD-Prozess

~~~text
GitHub Push / Pull Request
         |
         v
CI: Tests + Codeanalyse + Image-Build
         |
         v
Versioniertes Docker-Image (Git SHA / Digest)
         |
         v
Container Registry, z. B. GitHub Container Registry (GHCR)
         |
         v
Staging-Deployment + Smoke-Test
         |
         v
Freigabe für Produktion (Continuous Delivery)
         |
         v
Produktions-Backup + Wartungs-/Migrationsprüfung
         |
         v
Versioniertes Image auf Ubuntu ausrollen
         |
         v
Healthchecks + kurzer Funktionstest + Monitoring
         |
         +---- Fehler? → Code-Rollback / Notfallverfahren
~~~

### 13.3 Technische Umsetzung – Empfehlungen

1. Docker-Image **einmal in CI bauen**, nicht auf jedem Server neu aus dem Quellcode.
2. Image mit Commit-SHA und/oder Digest versionieren, nicht ausschließlich `latest` verwenden.
3. Image in eine Registry wie GHCR hochladen.
4. Produktionszugang über **separaten, eingeschränkten Deployment-Zugang**, nicht Root-SSH.
5. CI-Workflow und Produktiv-Deployment in **getrennten Jobs/Workflows** halten.
6. GitHub-Environment `production` mit Freigabe- und Secret-Regeln einsetzen.
7. Vor Datenbankmigrationen einen nachweisbaren Backup-/Restore-Pfad vorhalten.
8. Container aktualisieren und `/up` + Anwendungsszenarien prüfen.
9. Bei Fehlern Alarm auslösen; vorige Image-Version bereit halten.
10. Datenbankmigrationen hinsichtlich **Abwärtskompatibilität** planen. Ein Container-Rollback rollt eine Datenbank nicht automatisch zurück.

Für einen überschaubaren Produktivserver würde ich zunächst **Continuous Delivery mit manueller Freigabe** verwenden, bevor Vollautomatisierung sinnvoll ist.

### 13.4 Beispiel für den Vortrag

> „Unsere CI-Pipeline läuft bereits erfolgreich auf GitHub. Als nächsten Schritt würde ich nach den Tests ein versioniertes Docker-Image in GHCR veröffentlichen. Ein separater Deployment-Job könnte dieses Image nach Freigabe auf den Produktivserver ausrollen. Vor dem Rollout würde er ein Backup anstoßen; danach würden Healthchecks und Funktionstests laufen. Wichtig wäre ein klarer Rollback-Prozess, besonders bei Datenbankmigrationen.“

Weiterführend: https://docs.github.com/en/actions/concepts/workflows-and-actions/deployment-environments

---

## 14. Sicherheit und Benutzerverwaltung

### 14.1 Was bereits eingerichtet wurde

- SSH-Key-Authentifizierung (Passwortlogin deaktiviert);
- UFW aktiv mit `deny incoming`;
- öffentlich nur SSH/HTTP/HTTPS vorgesehen;
- PostgreSQL nur im internen Docker-Netz;
- Postfix nur auf Loopback und privatem Docker-Gateway;
- `.env` ausschließlich auf Ubuntu mit eingeschränkten Dateirechten;
- Laravel `APP_DEBUG=false` und `SESSION_SECURE_COOKIE=true`;
- öffentliches Webroot von Apache ist nur `public/`;
- Nginx bedient HTTPS über Let's Encrypt.

### 14.2 Zusätzlicher Administrator – noch ausstehend

**Geplant: `cookieops`** als normaler Benutzer mit bedarfsabhängiger `sudo`-Berechtigung. Vorgehen:

1. Benutzer anlegen;
2. öffentlichen SSH-Key installieren;
3. Rechte von `~/.ssh` und `authorized_keys` kontrollieren;
4. `sudo`-Berechtigung prüfen;
5. zweite SSH-Sitzung mit dem neuen Benutzer testen;
6. erst anschließend Root-SSH-Login vollständig verbieten.

An diesem Punkt **nichts blind an `sshd_config` ändern**, bevor ein alternativer, getesteter Zugang besteht.

### 14.3 Firewall überprüfen

~~~bash
sudo ufw status verbose
sudo sshd -T | grep -E 'port |pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication|permitrootlogin'
sudo ss -ltnp
sudo docker compose ps
~~~

UFW verhindert nicht automatisch jede durch Docker selbst veröffentlichte Verbindung; bei sicherheitskritischen Änderungen aus einem **externen Netzwerk** prüfen, welche Ports tatsächlich erreichbar sind.

### 14.4 Wartung / Sicherheitsupdates

Zum Zeitpunkt der Einrichtung meldete Ubuntu zahlreiche ausstehende Updates. Empfohlenes Vorgehen:

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
| Website nicht erreichbar | Nginx aus, DNS falsch, TLS-Problem | `systemctl status nginx`, `nginx -t`, `curl -I` |
| HTTP 502 Bad Gateway | App-Container nicht erreichbar | `docker compose ps`, App-Logs, Port 8080 |
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
3. `docker compose ps` und Logs prüfen.
4. `nginx -t` und HTTPS-Healthcheck ausführen.
5. Bei Datenbankfehlern **keine Daten löschen**.
6. Letztes funktionierendes Deployment und aktuelles Backup identifizieren.
7. Fehler dokumentieren, Verantwortliche informieren, kontrolliert beheben.
8. Nach der Wiederherstellung einen echten Anwendungstest durchführen.

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
- dokumentierter SSH-Zugangsprozess, **nicht** der private SSH-Key;
- verantwortlicher Admin-Benutzer und Notfallzugang über die Anbieter-Konsole;
- Standort der produktiven `.env` und sichere Übergabe der Secrets;
- Zugriff auf Host-/Nginx-/Postfix-/Docker-Konfigurationen;
- Backup-Speicher und bestätigter Restore-Prozess;
- Ansprechpartner für DNS, Domain, Zertifikate und Mail-Reputation;
- Liste offener Sicherheits- und Betriebsaufgaben.

### 16.4 Offene Punkte nach dem Challenge-Stand

- [x] Docker-Deployment mit Laravel und PostgreSQL
- [x] Reverse Proxy mit HTTPS und HTTP-Weiterleitung
- [x] Automatisiertes lokales Datenbank-Backup eingerichtet
- [x] SSH-Key, deaktivierter Passwortlogin und UFW
- [x] Postfix installiert und lokale Mailzustellung nachgewiesen
- [x] GitHub-Fork und CI-Workflow mit erfolgreichem Testlauf
- [x] CD-Verfahren als Konzept ausgearbeitet
- [x] Betriebshandbuch im Repository
- [ ] Dedizierten Administrator `cookieops` erstellen und Root-SSH endgültig sperren
- [ ] Produktiven externen Mailversand mit Unternehmensfreigabe realisieren
- [ ] Verschlüsselte Offsite-Backups einrichten
- [ ] Vollständigen Restore-Test dokumentieren
- [ ] Monitoring, Alarmierung und Sicherheitsupdates operationalisieren
- [ ] Automatisierten CD-Workflow mit Freigaben/Versionierung umsetzen
- [ ] Kubernetes-Aufgaben der separaten Challenge bearbeiten

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

---

*Dieses Dokument beschreibt die nachweisbar eingerichtete Challenge-Umgebung und unterscheidet ausdrücklich zwischen funktionierenden Komponenten, lokalen Demonstrationstests und noch offenen Produktionsanforderungen. Bei künftigen Änderungen bitte die Dokumentation zusammen mit der Infrastruktur aktualisieren.*
