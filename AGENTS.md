# AGENTS.md — crowdsec-mailcow-f2b-feed

## Überblick

Custom Crowdsec-Scenario-Repo (`Guezli/crowdsec-mailcow-f2b-feed`, GitHub-Remote `Guezli/crowdsec-mailcow-f2b-feed`). Kein offizielles Crowdsec-Hub-Paket — Installation nur über das mitgelieferte `install.sh`, nicht über `cscli scenarios install`.

Zweck (verifiziert aus README + YAML): Mailcows internes Fail2Ban-Äquivalent `netfilter-mailcow` bannt IPs (ausgelöst über Redis `F2B_CHANNEL` von SOGo, Dovecot, Postfix, Mailcow-UI) nur innerhalb des Mailcow-Docker-Netzwerks. Diese Bans erreichen Crowdsec sonst nicht. Dieses Repo liest den `docker logs`-Stream des `netfilter-mailcow`-Containers, parst die finalen Ban-Zeilen und erzeugt daraus lokale Crowdsec-Decisions, die der host-seitige nftables-Bouncer dann ebenfalls durchsetzt. Bans bleiben laut README strikt lokal (kein Push an CAPI/Remote-LAPI).

## Tech-Stack

- Bash (`install.sh`, ~120 Zeilen, `set -euo pipefail`)
- Crowdsec YAML-Konfiguration: Parser (`parsers/s01-parse/`), Scenario (`scenarios/`), Acquisition wird vom Installer generiert (nicht im Repo als Datei, nur als Heredoc in `install.sh`)
- Crowdsec Grok-Pattern-Syntax für Parsing
- `cscli hubtest`-Format für Tests (`tests/mailcow-f2b-feed/`)

## Setup / Build / Test — im Repo verifizierbar

Es gibt kein Build. Relevante Befehle, die im Repo selbst dokumentiert bzw. als Testfixtures vorhanden sind:

Installation (produktiv, auf einem Host mit laufendem Crowdsec + Mailcow):
```bash
curl -fsSL https://raw.githubusercontent.com/Guezli/crowdsec-mailcow-f2b-feed/main/install.sh | sudo bash
```
Manuelle/inspizierbare Variante ist in der README Schritt für Schritt aufgeführt (Acquisition-Heredoc, `curl` der Parser-/Scenario-Datei, `systemctl reload crowdsec`).

Verifikation nach Install (aus README):
```bash
sudo cscli scenarios list | grep mailcow-f2b-feed
sudo cscli parsers list | grep mailcow-f2b-bans
sudo cscli alerts list --scenario Guezli/mailcow-f2b-feed --since 24h
```

Test (aus Kommentar in `tests/mailcow-f2b-feed/scenarios.yaml`, Format ist Standard-`cscli hubtest`-Fixture — Befehl selbst wurde in diesem Repo-Checkout nicht ausgeführt, daher unklar/prüfen ob `cscli hubtest` hier lokal lauffähig ist ohne vollständige Crowdsec-Hub-Struktur):
```bash
cscli hubtest run mailcow-f2b-feed --clean
```
Testfixture-Inhalt: `tests/mailcow-f2b-feed/log.txt` enthält eine Beispielzeile (`CRIT: Banning 198.51.100.42/32 for 60 minutes`), `parsers.yaml` listet die Parser-Kette (`crowdsecurity/syslog-logs`, `crowdsecurity/dateparse-enrich`, `crowdsecurity/geoip-enrich`, `Guezli/mailcow-f2b-bans`), `scenarios.yaml` erwartet 1 Overflow (trigger-Typ) aus 1 Ban-Zeile.

Es gibt kein Lint-/CI-Skript, keine `package.json`, kein Makefile im Repo.

## Architektur-Kurzüberblick

Pipeline (aus README-Diagramm, mit Dateireferenzen verifiziert):

1. Mailcow-Container (`netfilter-mailcow`) schreibt Ban-Zeilen auf stdout, z. B. `CRIT: Banning 1.2.3.4/32 for 60 minutes` oder `Added host/network <IP> to denylist` (manueller Perm-Ban via `F2B_BLACKLIST`).
2. Crowdsec Docker-Acquisition liest per `docker logs` (Datei wird von `install.sh` nach `/etc/crowdsec/acquis.d/mailcow-f2b-feed.yaml` generiert, `source: docker`, `container_name` = autodetektierter Containername, `labels.type: mailcow-f2b`).
3. `crowdsecurity/non-syslog` (Hub-Standardparser, s00-raw) setzt `program=mailcow-f2b`.
4. `parsers/s01-parse/mailcow-f2b-bans.yaml` (dieses Repo) — Grok-Pattern matcht beide Ban-Varianten, extrahiert `meta.source_ip`, `meta.log_type = mailcow_f2b_ban`, `meta.ban_duration_min`, `meta.service = mailcow-f2b`.
5. `scenarios/mailcow-f2b-feed.yaml` (dieses Repo) — `type: trigger`, Filter auf `evt.Meta.log_type == 'mailcow_f2b_ban'`, `groupby: evt.Meta.source_ip`, `blackhole: 1h` (verhindert Re-Trigger bei Mailcow-Ban-Renewal binnen einer Stunde).
6. Lokale Crowdsec-LAPI registriert die Decision, host-seitiger nftables-Bouncer setzt sie um.

Installer-Logik (`install.sh`, 6 Schritte, idempotent):
1. prüft `cscli` auf PATH
2. autodetektiert `netfilter-mailcow`-Container per `docker ps` Namens-Grep (`netfilter.*mailcow|mailcow.*netfilter`)
3. schreibt Acquisition (überspringt falls Datei existiert)
4. lädt Parser via `curl` von `raw.githubusercontent.com/Guezli/crowdsec-mailcow-f2b-feed/main/...` (überspringt falls byte-identisch)
5. lädt Scenario analog
6. `systemctl reload crowdsec`, verifiziert Parser/Scenario via `cscli scenarios list` / `cscli parsers list`

Ziele: `/etc/crowdsec/acquis.d/mailcow-f2b-feed.yaml`, `/etc/crowdsec/parsers/s01-parse/mailcow-f2b-bans.yaml`, `/etc/crowdsec/scenarios/mailcow-f2b-feed.yaml`.

## Bekannte Konventionen / Fallstricke

- Kein Hub-Paket: `cscli scenarios install Guezli/mailcow-f2b-feed` funktioniert nicht, nur `install.sh` oder manuelles Kopieren gemäß README.
- Default-Containername `mailcowdockerized-netfilter-mailcow-1` — überlebt laut README Mailcow-Updates. Bei umbenanntem Mailcow-Compose-Projekt muss `acquis.d/mailcow-f2b-feed.yaml` manuell angepasst werden (kein Auto-Update durch dieses Repo nach Erstinstallation).
- Zwei unabhängige Ban-Layer nebeneinander (Mailcow-eigenes netfilter am Docker-Edge + dieser Feed am Host-nftables-Edge) — laut README bewusst redundant, kein Konflikt.
- Mailcow-Allowlist und Crowdsec-Whitelist sind getrennte Systeme; README warnt, kritische IPs müssen in beiden gepflegt werden.
- `blackhole: 1h` ist bewusst konservativ gewählt (verhindert Re-Trigger-Spam bei Ban-Renewal); die tatsächliche Crowdsec-Bandauer kommt aus dem separaten `profiles.yaml`, nicht aus diesem Scenario.
- Parser matcht zwei Zeilenformen (Commit `e4860a5`): automatische F2B-Trigger-Bans (`Banning <IP>/<N> for <M> minutes`) und manuelle Permanent-Bans (`Added host/network <IP> to denylist`, kein CIDR-Suffix, keine Minutenangabe).
- Git-Remote (`.git/config`, nicht committen/verändern) enthält einen Klartext-GitHub-PAT in der Origin-URL — bei jeglicher Git-Operation in diesem Repo (Klonen, Anzeigen der Remote-URL, Logs teilen) darauf achten, das Token nicht zu leaken. Siehe Memory-Eintrag zu Gitea-Token-Klartext für das analoge Muster in anderen Repos dieses Nutzers.
- Repo hat keinen CI-Workflow-Ordner (`.github/workflows` nicht vorhanden) — unklar/prüfen ob Tests irgendwo automatisiert laufen oder nur manuell via `cscli hubtest`.
