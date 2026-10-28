# PLAN – Neoforo

Specs: `docs/specs/00-vision.md`, `docs/specs/01-datenmodell.md`, `docs/specs/02-betrieb.md`.
Jede Phase endet mit einem Gate. Die nächste Phase beginnt erst, wenn das Gate grün ist.
Commits nach Conventional Commits. Code und Bezeichner Englisch, UI-Texte über fluent (de, en).

---

## Phase 0 – Gerüst

**Ziel:** Leeres, aber vollständig lauffähiges Projekt mit grüner CI.

- [ ] Cargo-Workspace, Edition 2024, Crates:
  - `crates/neoforo-core` (Library, keine IO-Abhängigkeiten)
  - `crates/neoforo-db` (Library, sqlx)
  - `crates/neoforo` (Binary)
- [ ] `compose.yaml`: Postgres (aktuelle Major), Mailpit, MinIO (nur zum Testen des
      S3-Backends; Standard ist lokaler Storage)
- [ ] Konfiguration über Umgebungsvariablen mit typisierter Config-Struct, `.env.example`,
      Laden einer Env-Datei für Server und CLI gleichermaßen (siehe 02-betrieb)
- [ ] Instanz-Secret `NEOFORO_SECRET_KEY` (Pflicht, min. 32 Byte), zweckgebundene Schlüssel
      per HKDF (siehe 01-datenmodell, Konventionen)
- [ ] Erste Migration: Extensions `citext`, Enum `content_state`, Tabelle `instances`
- [ ] Axum-Server mit `/health`, `/health/ready` und einer Startseite über Askama
- [ ] Start-Prüfung des Migrationsstands, `serve --migrate`, Graceful Shutdown bei `SIGTERM`
- [ ] Heartbeat in `instances`, `LISTEN`-Verbindung pro Prozess mit interner Verteilung
- [ ] Client-IP nur über `NEOFORO_TRUSTED_PROXIES` aus `X-Forwarded-For`
- [ ] Statische Assets mit Inhalts-Hash im Dateinamen, `Cache-Control: immutable`
- [ ] Basis-Layout-Template, htmx 4 als gepinnte, vendorte Datei unter `static/`
- [ ] Alle Gestaltungswerte im Basis-Layout als CSS Custom Properties (Design-Tokens, E15),
      helle und dunkle Variante nach `prefers-color-scheme`
- [ ] fluent-Setup mit `de` und `en`, Sprache aus User-Einstellung → `Accept-Language` → `de`
- [ ] `tracing` mit JSON-Ausgabe in Produktion, lesbarer Ausgabe in Entwicklung
- [ ] Security-Header-Middleware: CSP ohne `unsafe-inline`, `X-Content-Type-Options`,
      `Referrer-Policy`, `Permissions-Policy`
- [ ] CLI-Struktur: `neoforo serve [--role all|web|worker]`, `neoforo migrate`,
      `neoforo doctor` (siehe E12; in Phase 0: Config, Secret, DB, Migrationsstand)
- [ ] Metriken im Prometheus-Format auf eigenem Listener (`NEOFORO_METRICS_ADDR`), zunächst
      HTTP-Requests, Latenzen und DB-Pool
- [ ] GitHub Actions: fmt, clippy `-D warnings`, Tests mit Postgres-Service,
      `cargo sqlx prepare --check`, `cargo deny check`
- [ ] Dockerfile (Multi-Stage, Distroless oder Alpine), inkl. `pg_dump`/`pg_restore`,
      Listen- und Metrik-Adresse auf `0.0.0.0`
- [ ] `deploy/systemd/` mit Units für `all`, `web` und `worker`
- [ ] Compose-Profil `scale`: zwei `web`, ein `worker`, Caddy als Load Balancer, MinIO
- [ ] `LICENSE` (AGPL-3.0), README mit Schnellstart

**Gate:** `docker compose up -d && cargo run -- migrate && cargo run -- serve` liefert die
Startseite in de und en aus. `neoforo doctor` meldet keine Fehler. `/metrics` ist nur
auf dem internen Listener erreichbar, nicht auf dem öffentlichen Port. Das Profil `scale`
startet, beide `web`-Instanzen erscheinen in `instances` und liefern die Startseite. CI grün.

---

## Phase 1 – Accounts

**Ziel:** Registrieren, Anmelden, Abmelden, inklusive Vorbereitung auf XenForo-Passwörter,
Zwei-Faktor-Anmeldung, Passkeys und Spam-Abwehr bei der Registrierung.

- [ ] Migrationen: `users`, `user_profiles`, `user_fields`, `user_auth`, `user_tfa`,
      `user_recovery_codes`, `user_passkeys`, `user_groups`, `user_group_members`, `sessions`,
      `ip_log`, `spam_log` (siehe 01-datenmodell)
- [ ] Systemgruppen per Migration anlegen
- [ ] Passwort-Modul in `neoforo-core`:
  - Trait für Hash-Schemata, Implementierungen `argon2id` und `xf2_bcrypt`
  - Verifizieren → bei Erfolg und Nicht-argon2id: Rehash signalisieren
  - Unit-Tests mit festen Test-Hashes für jedes Schema
- [ ] Sessions: zufälliges 256-Bit-Token im Cookie (`HttpOnly`, `Secure`, `SameSite=Lax`),
      in der DB nur SHA-256 davon
- [ ] CSRF-Schutz für alle zustandsändernden Requests (Synchronizer Token aus der Session),
      funktioniert auch bei htmx-Requests
- [ ] Seiten: Registrierung, Login, Logout, eigenes Profil (Anzeige), Sicherheitseinstellungen
      (Passwort, zweiter Faktor, Passkeys, aktive Sessions mit Abmelden)
- [ ] Rate-Limit für Login und Registrierung über `rate_limits` (instanzübergreifend)
- [ ] E-Mail-Bestätigung über Mailpit (Versand über Job-Queue)
- [ ] Bestätigung der Datenschutzerklärung bei der Registrierung, Version wird gespeichert
      (Link und Version vorerst aus der Config, als `page`-Knoten ab Phase 4)
- [ ] Spam-Abwehr bei der Registrierung (siehe E10): Honeypot, Mindestzeit über signierten
      Zeitstempel, optionale Sperrliste für Wegwerf-Domains, Eintrag in `spam_log`
- [ ] Zwei-Faktor-Anmeldung (siehe E11):
  - TOTP mit verschlüsseltem Secret, Schutz gegen Wiederverwendung eines Codes
  - Wiederherstellungscodes
  - Passkeys (WebAuthn) als zweiter Faktor und als passwortlose Anmeldung
  - Login in zwei Schritten: nach dem Passwort eine kurzlebige Zwischen-Session, die nur den
    zweiten Faktor erlaubt
  - `require_tfa` pro Gruppe: betroffene Nutzer müssen nach dem Login einen zweiten Faktor
    einrichten, bevor sie weitermachen können
- [ ] E-Mail-Benachrichtigung bei neuer Anmeldung und bei Änderung von Anmeldedaten
- [ ] `ip_log`-Einträge für Registrierung und Login, Aufräum-Job für die Aufbewahrungsfrist
- [ ] Postgres-Job-Queue (`jobs`, siehe 01-datenmodell) mit `SKIP LOCKED`, Retry mit
      Backoff, Freigabe von Jobs toter Instanzen, Scheduler mit `dedupe_key`; läuft in den
      Rollen `all` und `worker`
- [ ] `sessions.last_seen_at` gedrosselt aktualisieren
- [ ] Ersten User per CLI zum Admin machen: `neoforo user promote <name>`
- [ ] `doctor` prüft zusätzlich den Mailversand, Metriken für die Job-Queue

**Gate:** Integrationstests für Registrierung, Login (beide Hash-Schemata inklusive Rehash),
Logout und CSRF-Ablehnung, TOTP (inkl. abgelehnter Wiederverwendung), Wiederherstellungscode,
Passkey-Registrierung und -Login (mit Software-Authenticator), Honeypot- und
Mindestzeit-Ablehnung, Löschung abgelaufener `ip_log`-Einträge. Mit zwei Instanzen: eine
Session gilt auf beiden, das Login-Rate-Limit wird gemeinsam gezählt, eine geplante Aufgabe
läuft bei zwei Workern genau einmal, ein Job einer beendeten Instanz wird wieder freigegeben.
CI grün.

---

## Phase 2 – Lesen

**Ziel:** Ein mit Testdaten gefülltes Forum lässt sich vollständig lesen, im XenForo-URL-Schema.

- [ ] Migrationen: `nodes`, `forums`, `threads`, `posts`, `polls`, `poll_responses`,
      `poll_votes`, `bb_codes`, `attachment_data`, `attachments`
- [ ] Storage-Trait mit Backends `fs` (Standard) und `s3`, Auswahl über die Config,
      `doctor` prüft den Storage; Anhänge bei `s3` per Redirect auf vorsignierte URLs
- [ ] `thread_views` mit Aggregations-Job
- [ ] Cache-Invalidierung über `NOTIFY` für prozesslokale Caches (z. B. `bb_codes`)
- [ ] `neoforo backup` und `neoforo restore [--verify]` (siehe E12), geplante Backups als Job
      mit Aufbewahrungsregel, Metrik und `doctor`-Warnung zum Alter des letzten Backups
- [ ] Seed-Befehl `neoforo dev seed` mit realistischen Fake-Daten (Baum mit Kategorien und
      Foren, mehrere tausend Threads, einige Threads mit > 1000 Posts, BBCode in allen Varianten,
      alle Thread-Typen inkl. Umfragen, eigene BBCodes, Signaturen)
- [ ] BBCode in `neoforo-core`:
  - Parser zu einem AST, tolerant gegenüber kaputtem Markup (nicht geschlossene Tags bleiben
    Text)
  - Renderer AST → HTML, alle Ausgaben escaped
  - Tag-Set v1: `b i u s`, `color`, `size`, `font`, `url`, `email`, `img`, `quote` (inkl.
    Attribution `name, post: id, member: id`), `code`, `icode`, `list`/`*`, `left center
    right`, `indent`, `spoiler`, `ispoiler`, `table tr th td`, `media` (Whitelist an
    Anbietern), `attach`, `user`, `plain`, `hr`
  - Unbekannte Tags werden als Text ausgegeben, nicht verworfen
  - Eigene BBCodes aus `bb_codes` über Ersetzungs-Templates, Template-Prüfung beim Speichern
    (siehe E1)
  - `img` mit externer URL über den Bild-Proxy, `media` als 2-Klick-Platzhalter (siehe E8)
  - Tests: Snapshot-Tests pro Tag, XSS-Testsuite (`javascript:`-URLs, Attribut-Ausbrüche,
    verschachtelte Tags, eigene BBCodes mit bösartigen Optionen), Fuzzing mit `cargo fuzz`
    für den Parser
- [ ] Render-Cache mit `render_version`, nur betrachterunabhängiges HTML, Platzhalter für
      Anhänge und externe Bilder werden pro Request ersetzt
- [ ] Bild-Proxy `/proxy/image` mit HMAC-signierten URLs: nur `http`/`https`, keine
      privaten/internen Zieladressen (SSRF), Größen- und Zeitlimit, nur Bild-MIME-Types,
      Zwischenspeicher im Storage
- [ ] Thread-Ansicht je Thread-Typ: Umfrageergebnisse, markierte Lösung bei `question`,
      Signaturen unter Posts
- [ ] Routen und Seiten:
  - `/` – Knotenübersicht
  - `/forums/{slug}.{id}/` mit Pagination (`page-{n}`)
  - `/threads/{slug}.{id}/` mit Pagination über `position`
  - `/posts/{id}/` → 302 auf Thread-Seite mit Anker
  - `/members/{name}.{id}/`
  - Falscher Slug → 301 auf kanonische URL
  - `/index.php?…` → 301 auf die entsprechende neue URL
  - Weitere XenForo-URLs nach E3 (`/attachments/…`, `/goto/post`, `…/post-{id}`, RSS),
    bewusst nicht unterstützte → 410
- [ ] Breadcrumbs über `nodes.path`
- [ ] Alle Seiten ohne JavaScript benutzbar
- [ ] Keine Requests an Dritte beim Lesen (siehe E8), per CSP erzwungen
- [ ] SEO nach E14: kanonische URLs, Meta-Daten und OpenGraph, JSON-LD
      (`DiscussionForumPosting`, `BreadcrumbList`, `ProfilePage`, `QAPage`), `sitemap.xml`
      als Job, `robots.txt`, RSS und Atom, `noindex`-Regeln, 410 für gelöschte Threads
- [ ] Theme-Grundlage nach E15: Tabelle `themes`, Standard-Theme aus Tokens, Umschalter
      hell/dunkel/auto ohne JS, Tokens werden als CSS-Datei mit Inhalts-Hash ausgeliefert

**Gate:** Snapshot- und XSS-Tests grün, Fuzzer läuft 10 Minuten ohne Fund, Thread-Seite mit
warmem Cache < 20 ms. Test belegt, dass der Render-Cache keine betrachterabhängigen
Inhalte enthält. SSRF-Tests für den Bild-Proxy grün. Ein Backup der Seed-Daten lässt sich mit
`restore --verify` wiederherstellen, Zeilenzahlen und Prüfsummen stimmen. Eine Änderung an
`bb_codes` wirkt auf allen Instanzen. Lasttest gegen das Profil `scale` ist dokumentiert
(`bench/`). JSON-LD der Thread-Seite validiert gegen Schema.org (Snapshot-Test), Sitemap
enthält keine für Gäste unsichtbaren Inhalte. CI grün.

---

## Ausblick (noch nicht detailliert)

- **Phase 3 – Importer v1:** eigene Spec `03-xenforo-import.md` vorher.
  Voraussetzung: echte XenForo-Testinstallation mit Daten. Enthält Wartungsmodus für den
  finalen Delta-Import und Link-Prüfung alter URLs im Bericht (siehe Migrationsanforderungen).
  Legt die Tabellen für Unterhaltungen an und importiert sie (Oberfläche folgt in Phase 6).
- **Phase 4 – Schreiben, Rechte, Moderation:** Editor (Tiptap → BBCode), Antworten, neue
  Threads, Rechtemodell (E4), Melden, Löschen, Verschieben, Bans, Audit-Log.
  Spam-Abwehr für Inhalte ist Voraussetzung für öffentliches Schreiben (siehe E10):
  Moderation der ersten Beiträge neuer Nutzer, Link- und Bild-Sperre bis zur Schwelle,
  `spam_rules`, Spam-Bereinigung mit Rückgängigmachen, `ip_log` für Beiträge, optionale
  Proof-of-Work-Prüfung und optionale externe Prüfdienste (beide standardmäßig aus).
  Datenschutz (siehe E9): Datenexport, Kontolöschung mit Trennung der Inhalte, Rechtstexte
  als `page`-Knoten.
  Anhänge und Avatare hochladen nach E19: Upload im Editor und ohne JS, Prüfung per Magic
  Bytes, Neukodierung ohne Metadaten, WebP-Vorschaubilder per Job, Grenzen pro Gruppe,
  Aufräumen verwaister Uploads, optional ClamAV. Gate u. a.: Testbilder mit GPS-EXIF
  kommen ohne Metadaten an, als Bild getarnte Dateien werden abgelehnt.
- **Phase 5 – Suche:** Postgres-Volltextsuche.
- **Phase 6 – Echtzeit und Kommunikation:** Benachrichtigungen (SSE), private
  Unterhaltungen nach E18 (Thread- und Chat-Ansicht, live per SSE, Lesebestätigungen,
  Privatsphäre-Einstellungen, Melden mit Kontext, E-Mail bei Abwesenheit), Chat nach E17 (Räume als Knoten, SSE über `LISTEN/NOTIFY`,
  Anwesenheit, Stummschalten, Aufbewahrungsfrist). Gate u. a.: Nachricht erreicht Clients
  auf einer anderen `web`-Instanz, verpasste Nachrichten kommen nach Reconnect an,
  Nicht-Teilnehmer (auch Moderatoren) erhalten keine Unterhaltungsinhalte, Lasttest mit
  vielen gleichzeitigen SSE-Verbindungen.
- **Phase 7 – Erweiterbarkeit und Gestaltung:** Webhooks nach E16 (inkl. Signatur nach
  Standard Webhooks, Outbox über die Job-Queue, SSRF-Schutz, Protokoll), Theme-Verwaltung
  nach E15 (Editor für Tokens, eigenes CSS, Assets, Vererbung, Import/Export als ZIP,
  Layout-Slots), optional IndexNow.
- Danach: Reaktionen, Präfixe, Plugins (WASM).
