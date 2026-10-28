# PLAN – Neoforo

Specs: `docs/specs/00-vision.md`, `docs/specs/01-datenmodell.md`.
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
- [ ] Konfiguration über Umgebungsvariablen mit typisierter Config-Struct, `.env.example`
- [ ] Erste Migration: Extensions `citext`, Enum `content_state`
- [ ] Axum-Server mit `/health` (prüft DB-Verbindung) und einer Startseite über Askama
- [ ] Basis-Layout-Template, htmx 4 als gepinnte, vendorte Datei unter `static/`
- [ ] fluent-Setup mit `de` und `en`, Sprache aus User-Einstellung → `Accept-Language` → `de`
- [ ] `tracing` mit JSON-Ausgabe in Produktion, lesbarer Ausgabe in Entwicklung
- [ ] Security-Header-Middleware: CSP ohne `unsafe-inline`, `X-Content-Type-Options`,
      `Referrer-Policy`
- [ ] CLI-Struktur: `neoforo serve`, `neoforo migrate`
- [ ] GitHub Actions: fmt, clippy `-D warnings`, Tests mit Postgres-Service,
      `cargo sqlx prepare --check`, `cargo deny check`
- [ ] Dockerfile (Multi-Stage, Distroless oder Alpine)
- [ ] `LICENSE` (AGPL-3.0), README mit Schnellstart

**Gate:** `docker compose up -d && cargo run -- migrate && cargo run -- serve` liefert die
Startseite in de und en aus. CI grün.

---

## Phase 1 – Accounts

**Ziel:** Registrieren, Anmelden, Abmelden, inklusive Vorbereitung auf XenForo-Passwörter.

- [ ] Migrationen: `users`, `user_profiles`, `user_fields`, `user_auth`, `user_groups`,
      `user_group_members`, `sessions` (siehe 01-datenmodell)
- [ ] Systemgruppen per Migration anlegen
- [ ] Passwort-Modul in `neoforo-core`:
  - Trait für Hash-Schemata, Implementierungen `argon2id` und `xf2_bcrypt`
  - Verifizieren → bei Erfolg und Nicht-argon2id: Rehash signalisieren
  - Unit-Tests mit festen Test-Hashes für jedes Schema
- [ ] Sessions: zufälliges 256-Bit-Token im Cookie (`HttpOnly`, `Secure`, `SameSite=Lax`),
      in der DB nur SHA-256 davon
- [ ] CSRF-Schutz für alle zustandsändernden Requests (Synchronizer Token aus der Session),
      funktioniert auch bei htmx-Requests
- [ ] Seiten: Registrierung, Login, Logout, eigenes Profil (Anzeige)
- [ ] Rate-Limit für Login und Registrierung
- [ ] E-Mail-Bestätigung über Mailpit (Versand über Job-Queue)
- [ ] Einfache Postgres-Job-Queue (`jobs`-Tabelle, Worker mit `SKIP LOCKED`) im Binary
- [ ] Ersten User per CLI zum Admin machen: `neoforo user promote <name>`

**Gate:** Integrationstests für Registrierung, Login (beide Hash-Schemata inklusive Rehash),
Logout und CSRF-Ablehnung. CI grün.

---

## Phase 2 – Lesen

**Ziel:** Ein mit Testdaten gefülltes Forum lässt sich vollständig lesen, im XenForo-URL-Schema.

- [ ] Migrationen: `nodes`, `forums`, `threads`, `posts`, `polls`, `poll_responses`,
      `poll_votes`, `bb_codes`, `attachment_data`, `attachments`
- [ ] Storage-Trait mit Backends `fs` (Standard) und `s3`, Auswahl über die Config
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

**Gate:** Snapshot- und XSS-Tests grün, Fuzzer läuft 10 Minuten ohne Fund, Thread-Seite mit
warmem Cache < 20 ms. Test belegt, dass der Render-Cache keine betrachterabhängigen
Inhalte enthält. SSRF-Tests für den Bild-Proxy grün. CI grün.

---

## Ausblick (noch nicht detailliert)

- **Phase 3 – Importer v1:** eigene Spec `02-xenforo-import.md` vorher.
  Voraussetzung: echte XenForo-Testinstallation mit Daten. Enthält Wartungsmodus für den
  finalen Delta-Import und Link-Prüfung alter URLs im Bericht (siehe Migrationsanforderungen).
- **Phase 4 – Schreiben, Rechte, Moderation:** Editor (Tiptap → BBCode), Antworten, neue
  Threads, Rechtemodell (E4), Melden, Löschen, Verschieben, Bans, Audit-Log.
  Spam-Abwehr ist Voraussetzung für öffentliches Schreiben: Honeypot und Zeitprüfung bei der
  Registrierung, Moderation der ersten Beiträge neuer Nutzer, Link-Limit für neue Nutzer,
  Spam-Bereinigung (Nutzer und Inhalte in einem Schritt). DSGVO: Datenexport und
  Kontolöschung mit Anonymisierung.
- **Phase 5 – Suche:** Postgres-Volltextsuche.
- Danach: Benachrichtigungen (SSE), Anhänge hochladen, Unterhaltungen, Reaktionen, Präfixe,
  Plugins (WASM).
