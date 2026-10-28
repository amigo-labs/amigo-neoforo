# 02 – Betrieb und Skalierung

Status: lebend · Zuletzt geändert: 2026-10-28 · Gilt ab Phase 0

Ergänzt E12 und E13 aus `00-vision.md`. Beschreibt, wie Neoforo konfiguriert, gestartet,
überwacht und gesichert wird, und welche Regeln der Code einhalten muss, damit die
`web`-Rolle horizontal skaliert.

## Konfiguration

Alle Einstellungen sind Umgebungsvariablen mit Präfix `NEOFORO_`. Server und CLI laden die
Konfiguration auf dieselbe Weise, damit z. B. `neoforo doctor` auf einem Server genau die
Einstellungen des laufenden Dienstes sieht.

Reihenfolge (spätere Quelle überschreibt frühere):

1. Env-Datei: `--env-file <pfad>`, sonst `NEOFORO_ENV_FILE`, sonst `/etc/neoforo/neoforo.env`
   falls vorhanden, sonst `./.env` (nur in Debug-Builds)
2. Echte Umgebungsvariablen

| Variable | Standard | Hinweis |
|---|---|---|
| `NEOFORO_DATABASE_URL` | – | Pflicht |
| `NEOFORO_DATABASE_LISTEN_URL` | = `DATABASE_URL` | Direkte Verbindung für `LISTEN`, falls `DATABASE_URL` über PgBouncer im Transaction-Modus läuft |
| `NEOFORO_DATABASE_MAX_CONNECTIONS` | 10 | Pro Prozess |
| `NEOFORO_SECRET_KEY` | – | Pflicht, min. 32 Byte, auf allen Instanzen identisch |
| `NEOFORO_BASE_URL` | – | Pflicht, z. B. `https://forum.example.org` |
| `NEOFORO_LISTEN_ADDR` | `127.0.0.1:8080` | Im Docker-Image `0.0.0.0:8080` |
| `NEOFORO_METRICS_ADDR` | `127.0.0.1:9100` | Im Docker-Image `0.0.0.0:9100`, auf dem Host nur an `127.0.0.1` veröffentlichen |
| `NEOFORO_TRUSTED_PROXIES` | leer | CIDR-Liste. Nur von diesen Adressen wird `X-Forwarded-For` ausgewertet (Basis für `ip_log` und Rate-Limits) |
| `NEOFORO_STORAGE` | `fs` | `fs` oder `s3` |
| `NEOFORO_STORAGE_PATH` | `/var/lib/neoforo/data` | Für `fs` |
| `NEOFORO_S3_*` | – | Endpoint, Bucket, Region, Zugangsdaten für `s3` |
| `NEOFORO_SMTP_URL` | – | `smtp://` oder `smtps://` |
| `NEOFORO_BACKUP_TARGET` | `/var/backups/neoforo` | Verzeichnis oder `s3://bucket/prefix` |
| `NEOFORO_BACKUP_MODE` | `builtin` | `builtin` oder `external` (siehe Große Installationen) |
| `NEOFORO_LOG_FORMAT` | `json` (Release), `pretty` (Debug) | |

Unbekannte `NEOFORO_*`-Variablen führen zu einer Warnung (Tippfehler-Schutz).

## CLI

| Befehl | Zweck | Ab Phase |
|---|---|---|
| `neoforo serve [--role all\|web\|worker] [--migrate]` | Server bzw. Worker starten | 0 |
| `neoforo migrate [--status]` | Ausstehende Migrationen ausführen bzw. anzeigen | 0 |
| `neoforo doctor [--json] [--send-test-mail <adresse>]` | Installation prüfen | 0 |
| `neoforo user promote <name>` | Nutzer zum Admin machen | 1 |
| `neoforo backup` | Backup erstellen | 2 |
| `neoforo restore <archiv> [--verify]` | Backup einspielen bzw. nur prüfen | 2 |
| `neoforo maintenance on\|off [--message <text>]` | Wartungsmodus | 3 |
| `neoforo import xenforo …` | Import (eigene Spec) | 3 |
| `neoforo dev seed` | Testdaten, nur in Debug-Builds | 2 |

Exit-Codes: `0` Erfolg, `1` Fehler. `doctor` zusätzlich: `2` nur Warnungen.

### Start und Migrationen

- `serve` prüft beim Start den Migrationsstand:
  - Fehlen Migrationen, die dieses Binary kennt: Start wird verweigert mit Hinweis auf
    `neoforo migrate`. Mit `--migrate` führt `serve` sie selbst aus (für Einzel-Server).
  - Enthält die Datenbank neuere Migrationen als das Binary: Start mit Warnung erlaubt. Das
    ist der Zustand während eines Rolling Updates (siehe Migrationsregeln).
- Gleichzeitige `migrate`-Aufrufe sind sicher (Postgres Advisory Lock).
- `SIGTERM`: keine neuen Requests annehmen, laufende Requests bis zu 30 s abschließen,
  SSE-Verbindungen schließen (Clients verbinden sich neu), laufende Jobs zu Ende führen
  oder freigeben.
- `/health`: Prozess lebt (`200 ok`). `/health/ready`: Datenbank erreichbar und Prozess nicht
  im Shutdown, sonst `503`. Load-Balancer nutzen `/health/ready`.

## Hosting-Varianten

### Einzel-Server mit Docker Compose

```bash
docker compose up -d postgres
docker compose run --rm neoforo migrate
docker compose up -d neoforo
docker compose run --rm neoforo user promote alice

docker compose exec neoforo neoforo doctor
docker compose exec neoforo neoforo backup

# Restore
docker compose stop neoforo
docker compose run --rm neoforo restore /backups/neoforo-2026-10-28.tar
docker compose up -d neoforo

# Update
docker compose pull
docker compose run --rm neoforo migrate
docker compose up -d neoforo
```

### Einzel-Server mit systemd

Binary unter `/usr/local/bin/neoforo`, Konfiguration in `/etc/neoforo/neoforo.env`
(Rechte `0640`, Gruppe `neoforo`), Dienst als Nutzer `neoforo`, Reverse Proxy (Caddy oder
nginx) davor.

```bash
sudo -u neoforo neoforo migrate
sudo systemctl enable --now neoforo
sudo -u neoforo neoforo doctor
sudo -u neoforo neoforo backup
```

Die Unit-Datei liegt im Repository unter `deploy/systemd/` (Varianten: `neoforo.service`
mit `--role all`, sowie `neoforo-web.service` und `neoforo-worker.service`).

### Skaliert

```
              ┌──────────────┐
  Clients ──▶ │ Load Balancer│──┬──▶ neoforo serve --role web   (n Instanzen)
              └──────────────┘  ├──▶ neoforo serve --role web
                                └──▶ …
                                     neoforo serve --role worker (1–m Instanzen)
                                              │
                         PostgreSQL (+ optional PgBouncer, Read-Replica für Backups)
                         S3-kompatibler Storage
```

- Voraussetzungen: `NEOFORO_STORAGE=s3` (oder ein von allen Hosts geteiltes Dateisystem),
  identisches `NEOFORO_SECRET_KEY`, `NEOFORO_TRUSTED_PROXIES` auf den Load Balancer gesetzt.
- Rolling Update: zuerst `neoforo migrate` (einmal, z. B. als Kubernetes-Job oder
  Deploy-Schritt), dann die Instanzen nacheinander ersetzen.
- Kein Sticky-Session-Bedarf.

## Skalierungsregeln für den Code

Diese Regeln gelten ab Phase 0 und werden in Reviews geprüft.

1. **Kein prozesslokaler Zustand, der für Korrektheit nötig ist.** Sessions, CSRF-Tokens,
   Rate-Limits, Jobs, Render-Cache und Rechte-Cache liegen in Postgres bzw. im Storage.
2. **Prozesslokale Caches nur mit Invalidierung über `NOTIFY`.** Jeder Prozess hält genau eine
   `LISTEN`-Verbindung (Kanal `neoforo_events`) und verteilt Ereignisse intern, z. B. an
   SSE-Clients oder zum Leeren von Caches (Einstellungen, `bb_codes`, Rechte). Nie eine
   `LISTEN`-Verbindung pro Client.
3. **Geplante Aufgaben laufen genau einmal**, egal wie viele Worker laufen: Der Scheduler
   legt Jobs mit `dedupe_key` (Aufgabe + Zeitfenster) an, Duplikate werden per
   `ON CONFLICT DO NOTHING` verworfen.
4. **Keine Schreibzugriffe pro Lesezugriff auf heiße Zeilen.**
   - Thread-Aufrufe landen in der `UNLOGGED`-Tabelle `thread_views` und werden von einem Job
     minütlich in `threads.view_count` übernommen.
   - `sessions.last_seen_at` und `users.last_activity_at` werden höchstens alle 5 Minuten
     aktualisiert.
5. **Migrationen sind rückwärtskompatibel zur Vorversion** (Expand/Contract): Spalten und
   Tabellen werden erst hinzugefügt und befüllt, entfernt oder umbenannt wird frühestens
   eine Version später. So laufen alte und neue Instanzen während eines Rolling Updates
   gleichzeitig.
6. **Dateien nie über lokale Pfade adressieren**, immer über den Storage-Trait. Beim
   `s3`-Backend liefert Neoforo Anhänge nach der Rechteprüfung per Redirect auf eine
   kurzlebige, vorsignierte URL aus, statt die Bytes durchzureichen.
7. **Statische Assets haben einen Inhalts-Hash im Dateinamen** und werden mit
   `Cache-Control: immutable` ausgeliefert, damit ein CDN davor gesetzt werden kann.
8. **Jede Instanz meldet sich in `instances` an** (Heartbeat alle 15 s). `doctor` und die
   Admin-Seite zeigen alle Instanzen mit Rolle, Version und letztem Heartbeat. Jobs von
   Instanzen ohne Heartbeat seit 2 Minuten werden wieder freigegeben. `doctor` warnt bei
   gemischten Versionen außerhalb eines Updates und bei `fs`-Storage mit Instanzen auf
   mehreren Hosts.

## Große Installationen

- **Verbindungen:** Anzahl Prozesse × `NEOFORO_DATABASE_MAX_CONNECTIONS` muss unter
  `max_connections` von Postgres bleiben, sonst PgBouncer davor. Neoforo nutzt keine
  sitzungsgebundenen Features außer `LISTEN` (dafür `NEOFORO_DATABASE_LISTEN_URL`) und
  Advisory Locks bei Migrationen (die direkt laufen).
- **Backups:** `neoforo backup` (`pg_dump`) ist für kleine und mittlere Foren gedacht. Große
  Installationen sichern Postgres mit Point-in-Time-Recovery (z. B. pgBackRest, WAL-G) und
  den Storage über S3-Versionierung. `NEOFORO_BACKUP_MODE=external` schaltet die geplanten
  Backups ab, `doctor` prüft dann nur noch, dass der Modus bewusst gesetzt ist.
- **Read-Replicas** sind vorerst kein Ziel: Forenseiten müssen eigene Schreibvorgänge sofort
  sehen. Eine Replica kann aber als Quelle für `pg_dump` dienen.
- **Lasttest:** Ein Compose-Profil `scale` startet zwei `web`-Instanzen, einen `worker` und
  Caddy als Load Balancer. Ein Lastskript (`oha`) gegen Thread-Seiten liegt unter
  `bench/`. Die Ergebnisse werden pro Phase im Repository festgehalten.
