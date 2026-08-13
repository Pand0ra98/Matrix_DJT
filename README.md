# DJT Infrastructure

Infrastructure-as-Code für den automatisierten Betrieb einer Matrix-Umgebung mit PostgreSQL und einer Smallstep-Zertifizierungsstelle. Die Bereitstellung, Konfiguration und Wartung der Systeme erfolgt reproduzierbar mit Ansible.

> **Projektstatus:** Diese Dokumentation enthält die vorgesehene Grundstruktur. Repository-spezifische Namen, Hosts und Befehle müssen ergänzt werden, sobald die Ansible-Dateien im Repository vorliegen.

## Inhalt

- [Ziel und Funktionsumfang](#ziel-und-funktionsumfang)
- [Architektur](#architektur)
- [Voraussetzungen](#voraussetzungen)
- [Schnellstart](#schnellstart)
- [Konfiguration](#konfiguration)
- [Betrieb](#betrieb)
- [Sicherheit](#sicherheit)
- [Backup und Wiederherstellung](#backup-und-wiederherstellung)
- [Tests und Qualitätssicherung](#tests-und-qualitätssicherung)
- [Mitwirken](#mitwirken)
- [KI-Nutzung](#ki-nutzung)
- [Lizenz](#lizenz)

## Ziel und Funktionsumfang

Das Repository soll die benötigte Infrastruktur deklarativ und wiederholbar bereitstellen. Dazu gehören insbesondere:

- Installation und Konfiguration der Matrix-Komponenten
- Bereitstellung und Absicherung von PostgreSQL
- Aufbau und Betrieb einer privaten PKI mit Smallstep CA (`step-ca`)
- Zertifikatsausstellung und -erneuerung für interne Dienste
- Verwaltung von Hosts, Gruppenvariablen und Umgebungen mit Ansible
- reproduzierbare Updates und Wartungsabläufe

## Architektur

| Komponente | Aufgabe |
|---|---|
| Ansible | Provisionierung und Konfigurationsmanagement |
| Matrix | Föderierte Echtzeitkommunikation |
| PostgreSQL | Persistente Datenbank für Matrix und gegebenenfalls weitere Dienste |
| Smallstep CA | Interne Zertifizierungsstelle und automatisierte Zertifikatsverwaltung |

Die konkrete Verteilung der Rollen auf Hosts ist im Ansible-Inventar definiert. Netzwerkports, DNS-Namen und Vertrauensbeziehungen sollten dort beziehungsweise in den zugehörigen Gruppenvariablen dokumentiert werden.

## Voraussetzungen

### Steuerungsrechner

- Linux, macOS oder WSL
- Python 3
- Ansible Core beziehungsweise die im Projekt festgelegte Ansible-Version
- SSH-Zugriff auf alle Zielsysteme
- `ansible-galaxy` für externe Collections und Rollen
- optional: `ansible-lint` und `yamllint`

### Zielsysteme

- unterstützte Linux-Distribution mit Python 3
- administrativer Zugriff über `sudo`
- funktionierende DNS-Auflösung und Zeitsynchronisation
- erreichbare Paketquellen
- ausreichend persistenter Speicher für PostgreSQL und Matrix-Medien

Die exakten Versionen und unterstützten Betriebssysteme sollten in einer `requirements.yml` beziehungsweise in den Rollen-Metadaten festgeschrieben werden.

## Schnellstart

1. Repository klonen und in das Projektverzeichnis wechseln:

   ```bash
   git clone <REPOSITORY-URL>
   cd <REPOSITORY-VERZEICHNIS>
   ```

2. Benötigte Collections und Rollen installieren:

   ```bash
   ansible-galaxy install -r requirements.yml
   ```

3. Inventar und Variablen für die gewünschte Umgebung konfigurieren. Beispiel:

   ```text
   inventories/
   ├── production/
   │   ├── hosts.yml
   │   ├── group_vars/
   │   └── host_vars/
   └── staging/
       ├── hosts.yml
       ├── group_vars/
       └── host_vars/
   ```

4. Verbindung zu den Zielsystemen prüfen:

   ```bash
   ansible all -i inventories/production/hosts.yml -m ping
   ```

5. Änderungen zunächst simulieren:

   ```bash
   ansible-playbook -i inventories/production/hosts.yml site.yml --check --diff
   ```

6. Konfiguration anwenden:

   ```bash
   ansible-playbook -i inventories/production/hosts.yml site.yml
   ```

> Passe Inventarpfad und Playbook-Namen an die tatsächliche Repository-Struktur an.

## Konfiguration

Konfigurationen sollten nach Umgebung getrennt und ohne Klartext-Secrets versioniert werden. Sinnvolle Variablengruppen sind:

- öffentliche DNS-Namen und Matrix-Servername
- Matrix- und Föderationseinstellungen
- PostgreSQL-Datenbank, Benutzer und Verbindungsparameter
- Pfade, Laufzeiten und Erneuerungsintervalle der Zertifikate
- Smallstep-CA-URL, Provisioner und Root-Fingerprint
- Firewall-Regeln, Reverse-Proxy- und TLS-Einstellungen
- Backup-Ziele und Aufbewahrungsfristen

Geheimnisse gehören verschlüsselt in Ansible Vault oder in einen angebundenen Secret Manager:

```bash
ansible-vault create inventories/production/group_vars/all/vault.yml
ansible-vault edit inventories/production/group_vars/all/vault.yml
```

Eine unverschlüsselte Beispieldatei wie `vault.example.yml` darf ausschließlich erfundene Werte und eine Beschreibung aller erforderlichen Variablen enthalten.

## Betrieb

Vor Änderungen an der Produktionsumgebung:

1. Changelog und Upstream-Migrationshinweise prüfen.
2. Aktuelles, wiederherstellbares Backup erstellen.
3. Playbook mit `--syntax-check` sowie `--check --diff` ausführen.
4. Änderung zunächst in einer Staging-Umgebung testen.
5. Produktionslauf protokollieren und anschließend Health Checks ausführen.

Empfohlene Health Checks:

- Matrix Client-Server API ist erreichbar
- Matrix-Föderation funktioniert, sofern aktiviert
- PostgreSQL akzeptiert Verbindungen und meldet keine Replikations- oder Speicherfehler
- ausgestellte Zertifikate bilden eine gültige Kette zur erwarteten Root CA
- Zertifikate werden vor Ablauf automatisch erneuert
- Backups laufen erfolgreich und werden regelmäßig testweise wiederhergestellt


## Backup und Wiederherstellung

Ein vollständiges Backup-Konzept sollte mindestens umfassen:

- konsistente PostgreSQL-Backups inklusive Rollen und Berechtigungen
- Matrix-Konfiguration, Signaturschlüssel und Mediendaten
- Smallstep-CA-Konfiguration, Datenbank und verschlüsselte Schlüssel
- Ansible-Inventare und verschlüsselte Vault-Dateien
- dokumentierte Wiederherstellungsreihenfolge und regelmäßige Restore-Tests

Backups müssen verschlüsselt, zugriffsgeschützt und getrennt von den Produktivsystemen gespeichert werden. Für Root- und Intermediate-CA-Schlüssel gelten zusätzlich die Vorgaben des PKI-Betriebskonzepts.

## Tests und Qualitätssicherung

Vor einem Merge sollten mindestens folgende Prüfungen erfolgreich sein:

```bash
ansible-playbook site.yml --syntax-check
ansible-lint
yamllint .
```

Inventarpfade und zusätzliche Parameter sind entsprechend der Projektstruktur zu ergänzen. Rollen sollten nach Möglichkeit mit Molecule oder einer vergleichbaren isolierten Testumgebung geprüft werden.

## Mitwirken

1. Issue mit Fehlerbeschreibung oder Änderungsvorschlag anlegen.
2. Einen Branch vom aktuellen Hauptbranch erstellen.
3. Änderungen klein und nachvollziehbar halten.
4. Dokumentation, Tests und Beispielkonfigurationen aktualisieren.
5. Pull Request mit Motivation, Testnachweis und Hinweisen zu möglichen Migrationen öffnen.

Committe weder echte Infrastrukturwerte noch personenbezogene oder geheime Daten.

## KI-Nutzung

KI-Werkzeuge dürfen zur Nutzung, Analyse, Bearbeitung und Weiterentwicklung dieses Projekts eingesetzt werden. Das gilt auch im kommerziellen Umfeld. KI-generierte Beiträge müssen dieselben Qualitäts-, Sicherheits-, Urheberrechts- und Lizenzanforderungen erfüllen wie manuell erstellte Beiträge.

Wer Änderungen veröffentlicht oder eine veränderte Version als Netzwerkdienst betreibt, muss die Pflichten der unten genannten Lizenz einhalten. Vor dem Übernehmen KI-generierter Inhalte ist insbesondere zu prüfen, dass keine fremden Geheimnisse, personenbezogenen Daten oder inkompatibel lizenzierten Bestandteile enthalten sind.

## Lizenz

Für dieses Projekt wird die **GNU Affero General Public License v3.0 oder später (`AGPL-3.0-or-later`)** empfohlen.

Sie erlaubt unter anderem:

- private und kommerzielle Nutzung
- Änderung und Weitergabe
- Einsatz von KI-Werkzeugen bei Nutzung und Entwicklung

Veränderte, weitergegebene Versionen müssen unter denselben Lizenzbedingungen verfügbar bleiben. Bei veränderten Versionen, die Nutzern über ein Netzwerk angeboten werden, muss ebenfalls der zugehörige Quellcode bereitgestellt werden. Das passt besonders zu serverseitiger Infrastruktur und verhindert, dass Verbesserungen ausschließlich in einem geschlossenen gehosteten Fork verbleiben.

Die vollständige Lizenz sollte als Datei `LICENSE` im Repository abgelegt werden. Für eine eindeutige Kennzeichnung kann jede Quelldatei folgenden SPDX-Identifier tragen:

```text
SPDX-License-Identifier: AGPL-3.0-or-later
```

> Diese Lizenzempfehlung ist keine Rechtsberatung. Vor einer Veröffentlichung sollte geprüft werden, ob alle Abhängigkeiten, Rollen, Collections und übernommenen Konfigurationen mit der AGPL-3.0-or-later kompatibel sind.
