# Matrix_DJT

Ansible-Playbooks zum Aufbau einer selbst gehosteten Matrix-Synapse-Umgebung mit PostgreSQL, Caddy und einer internen Smallstep-Zertifizierungsstelle.

Das Repository bildet derzeit eine Lab-Umgebung mit drei Funktionsgruppen ab:

- `App`: Matrix Synapse und Caddy
- `DB`: PostgreSQL als Synapse-Datenbank
- `CA`: Smallstep `step-ca` für interne TLS-Zertifikate

> [!WARNING]
> Das Projekt enthält fest eingetragene interne IP-Adressen, Hostnamen, einen SSH-Public-Key und deaktiviertes SSH-Host-Key-Checking. Passe diese Werte vor der Verwendung an. Die Playbooks sollten zunächst ausschließlich in einer Testumgebung ausgeführt werden.

## Inhaltsverzeichnis

- [Architektur](#architektur)
- [Repository-Struktur](#repository-struktur)
- [Voraussetzungen](#voraussetzungen)
- [Konfiguration](#konfiguration)
- [Installation](#installation)
- [Playbooks](#playbooks)
- [Validierung und Betrieb](#validierung-und-betrieb)
- [Sicherheit](#sicherheit)
- [Bekannte Einschränkungen](#bekannte-einschränkungen)
- [KI-Nutzung](#ki-nutzung)
- [Lizenz](#lizenz)

## Architektur

| Inventargruppe | Standardhost | IP-Adresse im Beispiel | Aufgabe |
|---|---|---:|---|
| `App` | `matrix-app-01` | `10.31.42.121` | Synapse, Caddy und `step-cli` |
| `DB` | `matrix-db-01` | `10.31.42.120` | PostgreSQL und Datenbank `synapse` |
| `CA` | `matrix-ca-01`, `matrix-ca-02` | `10.31.42.126`, `10.31.42.132` | Smallstep CA und interne PKI |

Die Hauptvariante verwendet folgende internen Namen und Dienste:

| Dienst | Adresse/Port |
|---|---|
| Matrix | `matrix.djt.lan` über Caddy/TLS |
| Synapse intern | Port `8008` |
| PostgreSQL | `10.31.42.120:5432` |
| Smallstep CA | `https://ca.lan:8443` |
| CRL-Endpunkt | `http://ca.lan:9001/1.0/crl` |

Die Lite-Variante verwendet stattdessen `matrix.pan-lab.de` und SQLite.

## Repository-Struktur

```text
Matrix_DJT/
├── Inventar.yaml               # Hosts und Ansible-Verbindungsdaten
├── ansible.cfg                 # Verwendet automatisch Inventar.yaml
├── base.yaml                   # Grundkonfiguration einer VM
├── Postgresql_EM.yaml          # PostgreSQL für Synapse
├── Synapse_Server.yaml         # Synapse mit PostgreSQL, Caddy und step-cli
├── Synapse_Server_Lite.yaml    # Synapse-Lite-Variante mit SQLite
├── Synapse_Server_Zerti.yaml   # Zertifikat für den Matrix-Host
├── CertAuthority.yaml          # Früher Entwicklungsstand der CA
├── CertAuthority_2.yaml        # CA mit ersten CRL-Erweiterungen
├── CertAuthority_3.yaml        # Aktuellster CA-Stand mit CRL-Templates
├── templates/
│   ├── Caddyfile.js2
│   ├── intermediate.tpl.j2
│   ├── leaf.tpl.j2
│   ├── password.js2
│   └── step-ca.service.js2
├── vars/
│   └── credentials.yaml        # Mit Ansible Vault verschlüsselte Secrets
└── LICENSE                     # GNU AGPL v3
```

Die Dateiendung `.js2` wird im Repository für einige Jinja2-Templates verwendet und ist beabsichtigt.

## Voraussetzungen

### Ansible-Control-Node

- Linux, macOS oder WSL
- Python 3
- eine aktuelle Ansible-Core-Version
- SSH-Zugriff auf die Zielsysteme
- Passwort beziehungsweise Vault-Password-Datei für `vars/credentials.yaml`
- Namensauflösung für `matrix.djt.lan` und `ca.lan`

Benötigte Collections:

```bash
ansible-galaxy collection install ansible.posix community.postgresql
```

### Zielsysteme

Die Playbooks sind auf Debian-/Ubuntu-artige Systeme mit `apt` ausgelegt. Sie erwarten unter anderem:

- Python 3 auf allen Zielsystemen
- SSH-Zugriff als `admin001` beziehungsweise initial als `root`
- `sudo`-Berechtigungen
- erreichbare Paketquellen von Matrix.org und Smallstep
- PostgreSQL 17 bei Verwendung von `Postgresql_EM.yaml`

`Postgresql_EM.yaml` bearbeitet ausdrücklich `/etc/postgresql/17/main/`. Bei einer anderen PostgreSQL-Version müssen diese Pfade angepasst werden.

## Konfiguration

### 1. Repository klonen

```bash
git clone https://github.com/Pand0ra98/Matrix_DJT.git
cd Matrix_DJT
```

### 2. Inventar anpassen

Passe in `Inventar.yaml` mindestens folgende Werte an:

- Hostnamen und IP-Adressen
- `ansible_user`
- Zuordnung zu den Gruppen `App`, `DB` und `CA`

Die mitgelieferte `ansible.cfg` setzt bereits:

```ini
[defaults]
inventory=./Inventar.yaml
forks=15
host_key_checking=False
```

Für produktive Systeme sollte `host_key_checking=True` verwendet werden. Hinterlege die Hostschlüssel zuvor kontrolliert in `known_hosts`.

### 3. DNS und feste Werte anpassen

Die folgenden Werte sind momentan direkt in Playbooks oder Templates eingetragen:

- `matrix.djt.lan`
- `matrix.pan-lab.de`
- `ca.lan`
- IP-Adressen `10.31.42.120`, `10.31.42.121`, `10.31.42.126` und `10.31.42.132`
- PostgreSQL-Pfad für Version 17
- Benutzer `admin001` beziehungsweise `admin002`
- SSH-Public-Key in `base.yaml`

Prüfe diese Werte vor jedem Lauf mit:

```bash
rg -n 'matrix\.djt\.lan|matrix\.pan-lab\.de|ca\.lan|10\.31\.42\.|admin00' .
```

### 4. Vault-Secrets pflegen

`vars/credentials.yaml` ist bereits mit Ansible Vault (`AES256`) verschlüsselt. Die Playbooks erwarten darin diese Variablen:

| Variable | Verwendung |
|---|---|
| `DB_USER` | PostgreSQL-Benutzer für Synapse |
| `DB_PASS` | Passwort des Datenbankbenutzers |
| `A_SECRET` | Shared Secret für die lokale Matrix-Benutzerregistrierung |
| `MATRIX_ADMIN_NAME` | Name des Matrix-Administrators in der PostgreSQL-Variante |
| `MATRIX_ADMIN_PW` | Passwort des Matrix-Administrators |
| `step_ca_PW` | Passwort für Smallstep CA und Provisioner |

Vault-Datei bearbeiten:

```bash
ansible-vault edit vars/credentials.yaml
```

Wenn das vorhandene Vault-Passwort nicht vorliegt, erstelle eine neue verschlüsselte Datei mit denselben Variablennamen:

```bash
mv vars/credentials.yaml vars/credentials.yaml.bak
ansible-vault create vars/credentials.yaml
```

Die Sicherung darf nicht in Git eingecheckt werden.

### 5. Erreichbarkeit prüfen

```bash
ansible all -m ansible.builtin.ping --ask-vault-pass
```

Alternativ kann bei allen folgenden Befehlen eine geschützte Vault-Password-Datei verwendet werden:

```bash
ansible-playbook PLAYBOOK.yaml --vault-password-file /sicherer/pfad/vault-password
```

## Installation

Es gibt derzeit kein übergeordnetes `site.yml`. Die Komponenten werden einzeln ausgeführt. Prüfe vor einem Lauf immer Syntax und Änderungen:

```bash
ansible-playbook PLAYBOOK.yaml --syntax-check --ask-vault-pass
ansible-playbook PLAYBOOK.yaml --check --diff --ask-vault-pass
```

### PostgreSQL-Variante

Vorgesehene Reihenfolge:

1. Hosts im Inventar und feste Projektwerte anpassen.
2. Zielsysteme vorbereiten; `base.yaml` ist derzeit fest auf `matrix-ca-02` begrenzt.
3. CA mit dem gewünschten Entwicklungsstand aufbauen.
4. PostgreSQL installieren und konfigurieren.
5. Synapse installieren und konfigurieren.
6. Matrix-Zertifikat ausstellen und für Caddy bereitstellen.

```bash
ansible-playbook base.yaml --ask-become-pass
ansible-playbook CertAuthority_3.yaml --ask-vault-pass --ask-become-pass
ansible-playbook Postgresql_EM.yaml --ask-vault-pass --ask-become-pass
ansible-playbook Synapse_Server.yaml --ask-vault-pass --ask-become-pass
ansible-playbook Synapse_Server_Zerti.yaml --ask-vault-pass --ask-become-pass
```

> [!IMPORTANT]
> `Synapse_Server.yaml` validiert die Caddy-Konfiguration bereits gegen Zertifikatsdateien, die erst `Synapse_Server_Zerti.yaml` erzeugt. Bei einer Erstinstallation müssen Zertifikatsbereitstellung und Caddy-Aktivierung deshalb koordiniert oder die Playbooks zuvor entsprechend angepasst werden.

### SQLite-Lite-Variante

Für eine kleine Testinstallation ohne separaten PostgreSQL-Host:

```bash
ansible-playbook Synapse_Server_Lite.yaml --ask-vault-pass --ask-become-pass
```

Die Lite-Variante nutzt `matrix.pan-lab.de`, SQLite unter `/var/lib/matrix-synapse/homeserver.db` und bindet Synapse nur an `127.0.0.1:8008`. Das Caddy-Template referenziert jedoch weiterhin die Zertifikatsdateien für `matrix.djt.lan`; passe Template und Zertifikatsablauf vor der Verwendung an.

## Playbooks

| Datei | Ziel | Beschreibung |
|---|---|---|
| `base.yaml` | `matrix-ca-02` | SSH-Key, `sudo`, Benutzergruppe und SSH-Konfiguration |
| `Postgresql_EM.yaml` | `DB` | PostgreSQL, Benutzer, Datenbank und Zugriff des App-Hosts |
| `Synapse_Server.yaml` | `App` | Synapse mit PostgreSQL, Caddy, `step-cli` und Matrix-Admin |
| `Synapse_Server_Lite.yaml` | `App` | Synapse mit SQLite, Caddy und festem Admin `admin002` |
| `Synapse_Server_Zerti.yaml` | `App` | CA-Bootstrap, Zertifikatsanforderung und Ablage für Caddy |
| `CertAuthority.yaml` | `CA` | Früher, teilweise interaktiver CA-Aufbau |
| `CertAuthority_2.yaml` | `matrix-ca-02` | Standalone-CA und CRL-Konfiguration |
| `CertAuthority_3.yaml` | `matrix-ca-02` | Neuester Stand mit Leaf-/Intermediate-Templates und CRL |

Für neue Installationen ist `CertAuthority_3.yaml` der vollständigste Stand. Die älteren CA-Playbooks bleiben als Entwicklungsstände enthalten und sollten nicht zusätzlich auf derselben CA ausgeführt werden.

Einzelne vorhandene Tags können mit `--list-tags` angezeigt werden:

```bash
ansible-playbook Synapse_Server.yaml --list-tags --ask-vault-pass
```

## Validierung und Betrieb

### Statische Prüfungen

```bash
ansible-playbook base.yaml --syntax-check
ansible-playbook CertAuthority_3.yaml --syntax-check --ask-vault-pass
ansible-playbook Postgresql_EM.yaml --syntax-check --ask-vault-pass
ansible-playbook Synapse_Server.yaml --syntax-check --ask-vault-pass
ansible-playbook Synapse_Server_Zerti.yaml --syntax-check --ask-vault-pass
ansible-lint
yamllint .
```

### Health Checks

Nach dem Deployment sollten mindestens geprüft werden:

```bash
curl --cacert /pfad/zur/root_ca.crt https://matrix.djt.lan/_matrix/client/versions
curl --cacert /pfad/zur/root_ca.crt https://ca.lan:8443/health
```

Auf den Zielsystemen:

```bash
systemctl status matrix-synapse
systemctl status caddy
systemctl status postgresql
systemctl status step-ca
```

### Backup

Ein vollständiges Backup muss mindestens enthalten:

- PostgreSQL-Datenbank `synapse`
- `/etc/matrix-synapse/` und `/var/lib/matrix-synapse/`
- `/etc/caddy/`
- `/var/lib/step-ca/` einschließlich CA-Datenbank, Konfiguration und verschlüsselter Schlüssel
- `Inventar.yaml` und die verschlüsselte `vars/credentials.yaml`

CA-Schlüssel und Vault-Passwort müssen getrennt, verschlüsselt und zugriffsgeschützt aufbewahrt werden. Wiederherstellungen sollten regelmäßig in einer isolierten Umgebung getestet werden.

## Sicherheit

- Ändere alle mitgelieferten Lab-Adressen, Benutzernamen und Schlüssel.
- Setze in `ansible.cfg` für den produktiven Betrieb `host_key_checking=True`.
- Bewahre das Vault-Passwort niemals im Repository auf.
- Verwende für Secrets weiterhin `no_log: true`; insbesondere die Admin-Erstellung sollte vor einem produktiven Einsatz überprüft werden.
- Beschränke PostgreSQL-Port `5432` per Firewall auf den App-Host.
- Schütze die Root CA besonders; für den Alltag sollte eine Intermediate CA verwendet werden.
- Entferne temporäre Passwortdateien nach der Zertifikatsausstellung.
- Prüfe vor Veröffentlichung die Git-Historie auf versehentlich eingecheckte Secrets.

Sicherheitslücken sollten über die GitHub-Funktion **Security Advisories** des Repositories und nicht als öffentliches Issue gemeldet werden.

## Bekannte Einschränkungen

- Viele Umgebungswerte sind noch fest in Playbooks und Templates eingetragen.
- `base.yaml` bereitet nur `matrix-ca-02` vor und enthält einen konkreten SSH-Public-Key.
- Es fehlt ein zentrales `site.yml` für einen durchgängigen Deployment-Ablauf.
- Es gibt keine `requirements.yml` mit festgeschriebenen Collection-Versionen.
- Das Caddy-Template und die Lite-Variante verwenden unterschiedliche Matrix-Domains.
- Die Reihenfolge von Caddy-Konfiguration und Zertifikatsausstellung ist bei der Erstinstallation nicht vollständig automatisiert.
- `CertAuthority.yaml` führt `step ca init` interaktiv aus; `_2` und `_3` automatisieren diesen Teil.
- CA-, PostgreSQL- und Synapse-Pfade setzen konkrete Softwareversionen und Debian-Layouts voraus.
- Die Playbooks wurden nicht als vollständig idempotent ausgewiesen; nutze `--check --diff` und eine Testumgebung.

## KI-Nutzung

KI-Werkzeuge dürfen zur Analyse, Nutzung, Bearbeitung und Weiterentwicklung dieses Projekts eingesetzt werden, auch kommerziell. KI-generierte Beiträge unterliegen denselben Qualitäts-, Sicherheits-, Urheberrechts- und Lizenzanforderungen wie manuell erstellte Beiträge.

Vor der Übernahme KI-generierter Inhalte ist zu prüfen, dass sie keine Secrets, personenbezogenen Daten oder inkompatibel lizenzierte Bestandteile enthalten. Die Pflichten der AGPL gelten unabhängig davon, ob Änderungen durch Menschen oder mit Unterstützung von KI erstellt wurden.

## Lizenz

Dieses Projekt steht gemäß der vorhandenen [`LICENSE`](LICENSE) unter der **GNU Affero General Public License v3.0 (`AGPL-3.0-only`)**.

Die AGPL erlaubt insbesondere private und kommerzielle Nutzung, Änderung und Weitergabe. Veränderte Versionen müssen unter denselben Bedingungen verfügbar bleiben. Wird eine veränderte Version über ein Netzwerk angeboten, muss den Nutzern außerdem der zugehörige Quellcode zugänglich gemacht werden.

Die Lizenz verbietet den Einsatz von KI-Werkzeugen nicht. Der Einsatz von KI hebt jedoch weder die AGPL-Pflichten noch Rechte Dritter auf.

> Diese Zusammenfassung ist keine Rechtsberatung. Maßgeblich ist ausschließlich der vollständige Lizenztext in `LICENSE`.
