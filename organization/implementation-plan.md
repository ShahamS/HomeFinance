# HomeFinance Implementierungsplan

## Zielbild

HomeFinance soll eine lokale Haushalts-Finanzsoftware werden, mit der ein Haushalt Einnahmen, Ausgaben und monatlich verbleibendes Geld jahrbezogen verwalten kann. Alle Haushaltsmitglieder sollen denselben Stand sehen, sobald sie im Heim-WLAN sind. Die Anwendung soll bewusst lokal-first funktionieren: Die zentrale Datenhaltung liegt im Heimnetz, nicht bei einem externen Cloud-Anbieter.

## Kernfunktionen

- Jahre anlegen und pro Jahr zwölf Monatsansichten verwalten.
- Pro Monat Einnahmen und Ausgaben eintragen.
- Einnahmen und Ausgaben über eigene Klassen/Kategorien strukturieren.
- Wiederkehrende oder ausgewählte Einträge aus dem Vormonat übernehmen.
- Monatliches Restgeld berechnen: `Einnahmen - Ausgaben`.
- Dashboard für Monats- und Jahresübersicht anzeigen.
- Gemeinsamer Datenstand für alle Geräte im Haushalt über das Heim-WLAN.
- Später optional: Export, Backups, Diagramme, Mehrwährungsfähigkeit, Rollen/Rechte.

## Empfohlener Tech-Stack

### Backend

- Python mit FastAPI.
- SQLite als Datenbank für den Start.
- SQLAlchemy oder SQLModel für Datenmodelle und Migrationen.
- Alembic für Datenbankmigrationen, sobald das Schema stabiler wird.
- Uvicorn als lokaler App-Server.

FastAPI passt gut, weil es schnell zu entwickeln ist, saubere API-Dokumentation mitbringt und später sowohl Web-Frontend als auch mögliche Mobile-/Desktop-Clients bedienen kann.

### Frontend

Zwei sinnvolle Optionen:

1. Web-App mit React, TypeScript und Vite.
2. Einfacherer Start mit serverseitigen Templates wie Jinja2 und etwas JavaScript.

Empfehlung: Für ein dauerhaft angenehm nutzbares Dashboard ist React + TypeScript besser. Für einen sehr schnellen Prototyp kann Jinja2 reichen. Da das Projekt wahrscheinlich wachsen soll, ist eine getrennte FastAPI-API plus React-Frontend die bessere Struktur.

### Deployment im Heimnetz

- Anwendung läuft auf einem dauerhaft erreichbaren Gerät im Heim-WLAN, z. B. Raspberry Pi, Mini-PC, NAS oder Heimserver.
- Andere Geräte öffnen die App über eine lokale Adresse, z. B. `http://homefinance.local:8000`.
- Optional mDNS/Avahi für `homefinance.local`.
- Regelmäßige automatische SQLite-Backups auf eine zweite lokale Platte oder ein NAS-Verzeichnis.

## Architektur

```text
HomeFinance/
  backend/
    app/
      api/
      core/
      db/
      models/
      schemas/
      services/
      main.py
    tests/
    pyproject.toml
  frontend/
    src/
      components/
      pages/
      services/
      state/
      styles/
    package.json
  organization/
    implementation-plan.md
  docker/
  README.md
```

## Datenmodell

### Household

Repräsentiert einen Haushalt. Für den Anfang reicht wahrscheinlich genau ein Haushalt, das Modell sollte aber vorbereitet sein.

Wichtige Felder:

- `id`
- `name`
- `created_at`

### User

Repräsentiert Haushaltsmitglieder.

Wichtige Felder:

- `id`
- `household_id`
- `display_name`
- `role`
- `created_at`

Für den ersten Prototyp kann Authentifizierung einfach gehalten oder lokal deaktiviert sein. Später sollten mindestens PIN/Login oder passwortlose lokale Accounts ergänzt werden.

### Category

Klassen/Kategorien für Einnahmen und Ausgaben.

Wichtige Felder:

- `id`
- `household_id`
- `name`
- `type`: `income` oder `expense`
- `color`
- `is_active`

Beispiele:

- Einnahmen: Gehalt, Kindergeld, Nebenjob, Rückzahlungen
- Ausgaben: Miete, Strom, Internet, Lebensmittel, Versicherungen, Freizeit

### FinancialEntry

Ein einzelner Einnahmen- oder Ausgabeneintrag.

Wichtige Felder:

- `id`
- `household_id`
- `category_id`
- `year`
- `month`
- `type`: `income` oder `expense`
- `title`
- `amount_cents`
- `currency`
- `booking_date`
- `notes`
- `is_recurring_candidate`
- `created_by_user_id`
- `created_at`
- `updated_at`

Beträge sollten intern als Cent-Integer gespeichert werden, nicht als Float.

### EntryTemplate

Optionales Modell für wiederkehrende Standardposten.

Wichtige Felder:

- `id`
- `household_id`
- `category_id`
- `type`
- `title`
- `amount_cents`
- `currency`
- `default_day`
- `is_active`

Dieses Modell ist nützlich für Miete, Internet, Versicherungen oder regelmäßige Einnahmen. Die Funktion "aus Vormonat übernehmen" kann aber zuerst direkt aus `FinancialEntry` gebaut werden.

## Wichtige API-Endpunkte

### Jahre und Monate

- `GET /api/years`
- `GET /api/years/{year}/summary`
- `GET /api/months/{year}/{month}`
- `GET /api/months/{year}/{month}/summary`

### Einträge

- `GET /api/entries?year=2026&month=10`
- `POST /api/entries`
- `PATCH /api/entries/{entry_id}`
- `DELETE /api/entries/{entry_id}`

### Kategorien

- `GET /api/categories`
- `POST /api/categories`
- `PATCH /api/categories/{category_id}`
- `DELETE /api/categories/{category_id}`

### Vormonat übernehmen

- `GET /api/months/{year}/{month}/copy-candidates`
- `POST /api/months/{year}/{month}/copy-from-previous`

Der Copy-Endpunkt sollte eine Liste ausgewählter Eintrags-IDs entgegennehmen. So kann der Nutzer bewusst entscheiden, welche Posten übernommen werden.

### Dashboard

- `GET /api/dashboard/monthly?year=2026`
- `GET /api/dashboard/yearly?year=2026`

## Frontend-Ansichten

### Jahresübersicht

- Auswahl des Jahres.
- Tabelle oder Diagramm mit allen Monaten.
- Pro Monat: Einnahmen, Ausgaben, verbleibendes Geld.
- Schneller Einstieg in einen Monat.

### Monatsansicht

- Einnahmenliste.
- Ausgabenliste.
- Kategorienfilter.
- Monatsbilanz.
- Button zum Übernehmen ausgewählter Vormonats-Einträge.
- Eintragsdialog zum Erstellen/Bearbeiten.

### Kategorienverwaltung

- Kategorien getrennt nach Einnahmen und Ausgaben.
- Farbe und Aktivstatus.
- Warnung, wenn eine Kategorie bereits in Einträgen verwendet wird.

### Dashboard

- Jahresbilanz.
- Monatlicher Verlauf von Einnahmen, Ausgaben und Restgeld.
- Top-Ausgabenkategorien.
- Vergleich zum Vormonat.

## Sync- und Heim-WLAN-Konzept

Für den Start sollte es keinen komplexen Peer-to-Peer-Sync geben. Besser ist ein zentraler lokaler Server im Heimnetz:

```text
Laptop / Handy / Tablet
        |
        | Heim-WLAN
        v
HomeFinance Server im lokalen Netz
        |
        v
SQLite Datenbank + Backups
```

Vorteile:

- Alle sehen sofort denselben Stand.
- Keine Konfliktauflösung zwischen mehreren lokalen Datenbanken nötig.
- Einfacher zu sichern.
- Einfacher zu debuggen.

Spätere Erweiterung:

- Offline-Modus mit lokaler Queue.
- Synchronisation über Änderungslog.
- Konfliktlösung nach `updated_at` und Eintragsversion.

Für Version 1 ist ein immer erreichbarer Heimserver die bessere Entscheidung.

## Sicherheit

- Anwendung nur im Heimnetz erreichbar machen.
- Keine Portfreigabe ins Internet.
- Optional Basic Auth oder lokales Login.
- Datenbankdatei und Backups nicht öffentlich freigeben.
- Regelmäßige Backups einplanen.
- Später HTTPS im LAN optional über Reverse Proxy, z. B. Caddy.

## Entwicklungsphasen

### Phase 1: Projektbasis

- Repository-Struktur anlegen.
- FastAPI-Backend initialisieren.
- SQLite-Verbindung einrichten.
- Grundmodelle für Kategorien und Einträge erstellen.
- Minimaltests für Berechnungslogik schreiben.

### Phase 2: Finanzlogik

- CRUD für Kategorien.
- CRUD für Einnahmen und Ausgaben.
- Monats- und Jahresberechnungen implementieren.
- Restgeld pro Monat berechnen.
- Validierungen für Betrag, Monat, Jahr und Kategorie ergänzen.

### Phase 3: Vormonatsübernahme

- Einträge des Vormonats als Kandidaten anzeigen.
- Auswahl einzelner Kandidaten ermöglichen.
- Ausgewählte Einträge in den aktuellen Monat kopieren.
- Kopierte Einträge klar als neue Einträge speichern, nicht als Referenzen.

### Phase 4: Web-Frontend

- React/Vite-Projekt anlegen.
- Jahresübersicht bauen.
- Monatsansicht bauen.
- Eintragsdialoge und Kategorienverwaltung ergänzen.
- API-Client zentral kapseln.

### Phase 5: Dashboard

- Monats- und Jahreskarten.
- Diagramme für Einnahmen, Ausgaben und Restgeld.
- Kategorieauswertung.
- Vergleich zum Vormonat.

### Phase 6: Heimnetz-Betrieb

- Start per Docker Compose oder systemd Service.
- Lokalen Hostnamen konfigurieren.
- Backup-Strategie einrichten.
- Kurze Setup-Dokumentation schreiben.

### Phase 7: Polishing

- Authentifizierung und Nutzerrollen.
- Export als CSV/PDF.
- Such- und Filterfunktionen.
- Wiederkehrende Templates.
- Import bestehender Tabellen.

## Erste konkrete Meilensteine

1. Backend-Skeleton mit Healthcheck-Endpunkt.
2. SQLite-Schema für Kategorien und Einträge.
3. API für Kategorien und Einträge.
4. Service-Funktion für Monatsbilanz.
5. Testdaten-Seed für ein Beispieljahr.
6. Erste Monatsübersicht im Frontend.
7. Vormonatsübernahme.
8. Jahresdashboard.
9. Docker-Setup für den Heimserver.

## Offene Entscheidungen

- Soll es direkt mehrere Benutzer mit Login geben oder reicht anfangs ein gemeinsamer Haushaltszugang?
- Soll die App primär am Desktop genutzt werden oder auch regelmäßig am Handy?
- Soll es Budgets pro Kategorie geben oder nur Ist-Werte?
- Soll ein Datenimport aus CSV/Excel früh unterstützt werden?
- Wo soll die Anwendung im Heimnetz dauerhaft laufen?

## Empfehlung für Version 1

Version 1 sollte bewusst schlank bleiben:

- Ein Haushalt.
- Lokaler Server im Heim-WLAN.
- Kategorien.
- Einnahmen und Ausgaben pro Monat.
- Vormonatsübernahme.
- Jahresdashboard.
- SQLite mit automatischen Backups.

Damit entsteht schnell ein brauchbares Produkt, ohne sich zu früh in komplexem Sync, Authentifizierung oder Mobile-Offline-Logik zu verlieren.
