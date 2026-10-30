---
ticket: CB-1
stage: design
source: examples/tickets/cb-1-entwuerfe.md
intent: docs/work/CB-1/intent.md
---

# CB-1: Artikel als Entwurf speichern und veröffentlichen

## Ziel

Autoren können einen Artikel anlegen, der zunächst als Entwurf (published: false) gespeichert wird und für andere Nutzer unsichtbar bleibt. Über einen dedizierten Endpunkt kann der Autor den Artikel unwiderruflich veröffentlichen; danach ist er für alle sichtbar. Ein separater Drafts-Endpunkt gibt dem Autor eine Übersicht seiner unveröffentlichten Artikel.

## Gewählter Ansatz

Das Prisma-Schema erhält ein Boolean-Feld published (Standard: false) im Article-Modell. Neue Artikel werden immer als Entwurf angelegt. Ein dedizierter POST /articles/:slug/publish-Endpunkt setzt published auf true (irreversibel). Ein neuer GET /articles/drafts-Endpunkt listet die Entwürfe des authentifizierten Autors. Alle bestehenden Listing-Endpunkte werden um den Filter published: true erweitert.

**Begründung:** Approach B wurde in der Rückfrage bestätigt. Die Trennung von Erstellen und Veröffentlichen in zwei Endpunkte macht den Lebenszyklus explizit und verhindert versehentliche Veröffentlichungen. Das published-Feld im Schema ist die minimale Datenbankänderung und lässt sich in alle bestehenden Prisma-Abfragen durch einen zusätzlichen WHERE-Term einbauen, ohne die Schichtentrennung Controller → Service → Prisma zu durchbrechen.

**Verworfene Alternativen:**

- **published-Flag in bestehenden Endpunkten (Approach A)** — In der Rückfrage verworfen. Das Mitgeben von published im Body von POST /articles und PUT /articles als optionalem Flag würde den Lebenszyklus implizit lassen, mehr Request-Validierung erfordern und keinen sauberen Entwurf-Endpunkt liefern.

## Architektur und Komponenten

### schema.prisma – Article-Modell

- **Zweck:** Persistiert den Veröffentlichungsstatus jedes Artikels
- **Schnittstelle:** published Boolean @default(false)
- **Abhängigkeiten:** Keine.

### article.service.ts – buildFindAllQuery

- **Zweck:** Schränkt die öffentliche Artikelliste auf veröffentlichte Artikel ein
- **Schnittstelle:** Internes AND-Array erhält zusätzlichen Eintrag { published: { equals: true } }
- **Abhängigkeiten:** schema.prisma – Article-Modell

### article.service.ts – getFeed

- **Zweck:** Schränkt den personalisierten Feed auf veröffentlichte Artikel ein
- **Schnittstelle:** published: { equals: true } wird dem where-Objekt in prisma.article.count und prisma.article.findMany hinzugefügt
- **Abhängigkeiten:** schema.prisma – Article-Modell

### article.service.ts – getArticle

- **Zweck:** Verhindert den Zugriff auf Entwürfe durch Nicht-Autoren
- **Schnittstelle:** getArticle(slug: string, id?: number) — nach dem Fetch: falls article.published === false && article.authorId !== id → HttpException(404, { errors: { article: ['not found'] } })
- **Abhängigkeiten:** schema.prisma – Article-Modell

### article.service.ts – createArticle

- **Zweck:** Legt neue Artikel immer als Entwurf an
- **Schnittstelle:** published: false wird zum Prisma create-Datenobjekt hinzugefügt
- **Abhängigkeiten:** schema.prisma – Article-Modell

### article.service.ts – getDrafts

- **Zweck:** Liefert alle unveröffentlichten Artikel des authentifizierten Autors
- **Schnittstelle:** getDrafts(authorId: number, offset?: number, limit?: number) → Promise<{ articles: ArticleResponse[], articlesCount: number }> — filtert per authorId = X AND published = false
- **Abhängigkeiten:** schema.prisma – Article-Modell, article.mapper.ts

### article.service.ts – publishArticle

- **Zweck:** Setzt published auf true für einen Artikel des authentifizierten Autors (idempotent)
- **Schnittstelle:** publishArticle(slug: string, authorId: number) → Promise<ArticleResponse> — wirft HttpException(404) wenn nicht gefunden, HttpException(403) wenn nicht Autor; ist der Artikel bereits published, wird er unverändert zurückgegeben
- **Abhängigkeiten:** schema.prisma – Article-Modell, article.mapper.ts

### article.mapper.ts

- **Zweck:** Ergänzt das gemappte Artikelobjekt um das Feld published
- **Schnittstelle:** published: article.published wird zur bestehenden Rückgabestruktur hinzugefügt
- **Abhängigkeiten:** schema.prisma – Article-Modell

### article.controller.ts – GET /articles/drafts

- **Zweck:** Endpunkt für die Entwurfsliste des Autors, auth.required
- **Schnittstelle:** GET /api/articles/drafts → { articles, articlesCount } — muss vor der Route GET /articles/:slug registriert werden
- **Abhängigkeiten:** article.service.ts – getDrafts

### article.controller.ts – POST /articles/:slug/publish

- **Zweck:** Endpunkt zum Veröffentlichen eines Entwurfs, auth.required
- **Schnittstelle:** POST /api/articles/:slug/publish → { article } mit published: true
- **Abhängigkeiten:** article.service.ts – publishArticle

## Datenfluss

1. Autor sendet POST /articles → createArticle speichert Artikel mit published: false → Antwort enthält published: false. 2. Autor ruft GET /articles/drafts ab → getDrafts filtert per authorId = X AND published = false → gibt paginierte Liste zurück. 3. Autor sendet POST /articles/:slug/publish → publishArticle prüft Existenz und Autorenschaft → setzt published = true → gibt aktualisierten Artikel zurück. 4. Öffentlicher GET /articles → buildFindAllQuery enthält published: true → nur veröffentlichte Artikel werden zurückgegeben. 5. GET /articles/:slug auf Entwurf von Nicht-Autor → getArticle prüft published-Status und authorId → wirft HttpException(404).

## Fehlerbehandlung

GET /articles/:slug: Entwurf eines anderen Autors oder unauthentifiziert → 404 (kein Hinweis auf Existenz des Entwurfs). POST /articles/:slug/publish: Artikel nicht gefunden → 404; Anfrage von Nicht-Autor → 403; bereits veröffentlicht → 200 (idempotent). Alle auth.required-Routen: kein oder ungültiger Token → 401 (durch bestehende auth-Middleware in auth.ts).

## Akzeptanzkriterien

- POST /articles mit gültigem Auth-Token erzeugt einen Artikel mit published: false in der Antwort
- GET /articles gibt keinen Entwurf zurück – auch nicht für den Autor selbst
- GET /articles/feed gibt keinen Entwurf eines gefolgten Autors zurück
- GET /articles/:slug auf einen Entwurf von einem Nicht-Autor (oder unauthentifiziert) liefert HTTP 404
- GET /articles/:slug auf den eigenen Entwurf durch den Autor liefert HTTP 200 mit published: false
- GET /articles/drafts (auth required) liefert ausschließlich unveröffentlichte Artikel des aktuellen Nutzers
- GET /articles/drafts gibt keine Entwürfe anderer Autoren zurück
- POST /articles/:slug/publish durch den Autor setzt published: true und gibt den Artikel mit published: true zurück
- Nach Veröffentlichung erscheint der Artikel in GET /articles und GET /articles/feed (bei gefolgtem Autor)
- POST /articles/:slug/publish durch einen Nicht-Autor liefert HTTP 403
- POST /articles/:slug/publish auf einen nicht vorhandenen Artikel liefert HTTP 404
- Die Antwort aller Artikel-Endpunkte enthält das Feld published als Boolean

## Tests

Unit-Tests (Jest, Prisma-Mock in src/tests/services/article.service.test.ts): getDrafts gibt nur Artikel mit published: false des korrekten Autors zurück; publishArticle wirft HttpException(403) bei fremdem Autor; publishArticle setzt published auf true; getArticle wirft HttpException(404) wenn Artikel Entwurf und Anfragender nicht Autor ist. E2E (e2e/src/server/server.spec.ts): Vollständiger Lebenszyklus – Artikel anlegen (prüfen: published: false, nicht in GET /articles), GET /articles/drafts liefert ihn, veröffentlichen über POST /articles/:slug/publish (prüfen: published: true, in GET /articles sichtbar); Zugriff auf Entwurf per Slug von anderem Nutzer liefert 404.

## Betroffene Dateien

- `src/prisma/schema.prisma` — published Boolean @default(false) zum Article-Modell hinzufügen
- `src/app/routes/article/article.service.ts` — buildFindAllQuery und getFeed um published-Filter erweitern; getArticle um Zugriffsprüfung für Entwürfe ergänzen; createArticle um published: false im create-Objekt ergänzen; neue Funktionen getDrafts und publishArticle hinzufügen
- `src/app/routes/article/article.controller.ts` — Neue Routen GET /articles/drafts (vor /:slug) und POST /articles/:slug/publish registrieren
- `src/app/routes/article/article.mapper.ts` — published: article.published zur gemappten Artikelstruktur hinzufügen
- `src/app/routes/article/article.model.ts` — published: boolean zur Article-Interface hinzufügen
- `src/tests/services/article.service.test.ts` — Unit-Tests für getDrafts, publishArticle und die erweiterte Zugriffslogik in getArticle hinzufügen

## Risiken

- [hoch] Bestehende Artikel erhalten durch das Prisma-Migrations-Standardverhalten (DEFAULT false) den Wert published = false und verschwinden damit aus dem öffentlichen Feed — In der generierten Prisma-Migrationsdatei nach dem ALTER TABLE-Statement ein UPDATE Article SET published = true einfügen, sodass alle bisher angelegten Artikel als veröffentlicht gelten
- [mittel] GET /articles/drafts muss vor GET /articles/:slug in article.controller.ts registriert werden, sonst interpretiert Express 'drafts' als Slug-Parameter — Registrierreihenfolge in article.controller.ts einhalten: /articles → /articles/feed → /articles/drafts → /articles/:slug

## Offene Punkte

- Soll favoriteArticle / unfavoriteArticle auf Entwürfe beschränkt oder blockiert werden? Die Intent-Erfolgskriterien adressieren dies nicht; aktuell ist Favorisieren eines eigenen Entwurfs durch den Autor technisch möglich.

## Geklärte Rückfragen

**F:** Kann ein bereits veröffentlichter Artikel vom Autor wieder in den Entwurfsstatus zurückversetzt werden (d. h. ist Veröffentlichen eine umkehrbare Aktion), oder ist der Übergang von Entwurf zu veröffentlicht eine Einbahnstraße?\
**A:** Nein, veröffentlicht ist endgültig. Ein veröffentlichter Artikel kann nicht zurück in den Entwurfsstatus.

**F:** Wie wird der Entwurf-/Veröffentlichungs-Lebenszyklus in der API abgebildet? (Angeboten: A: published-Flag in bestehenden Endpunkten; B: Dedizierter Publish-Endpunkt + Drafts-Endpunkt)\
**A:** Ansatz B: Dedizierter Publish-Endpunkt + Drafts-Endpunkt
