---
ticket: CB-1
stage: design
source: examples/tickets/cb-1-entwuerfe.md
intent: docs/work/CB-1/intent.md
---

# CB-1: CB-1: Artikel als Entwurf speichern und veröffentlichen

## Ziel

Artikel erhalten ein boolesches Feld `published` (Standard: `false`). Unveröffentlichte Artikel (Entwürfe) sind ausschließlich für den Autor über einen dedizierten Endpunkt sichtbar und werden aus allen öffentlichen Listen und dem Feed herausgefiltert. Das Veröffentlichen eines Entwurfs ist ein irreversibler Zustandswechsel.

## Gewählter Ansatz

Das Feld `published` wird als boolesches Attribut am `Article`-Modell in Prisma ergänzt (Default `false`). Bestehende Endpunkte filtern Entwürfe serverseitig heraus. Für den Direktabruf eines Entwurfs durch Nicht-Autoren wird 403 zurückgegeben. Ein neuer, auth-geschützter Endpunkt `GET /api/articles/drafts` liefert ausschließlich die eigenen Entwürfe des aufrufenden Autors. Das Veröffentlichen erfolgt durch Übergabe von `published: true` im bestehenden `PUT /api/articles/:slug`-Endpunkt.

**Begründung:** Das Erweitern des bestehenden PUT-Endpunkts um das Feld `published` ist konsistent mit dem Muster in `updateArticle`, das alle anderen Felder (title, body, description) als optionale Partialupdates behandelt. Ein separater Publish-Endpunkt (z. B. POST /publish) wäre ein zusätzlicher Verb-basierter Sonderweg, den das Codebase-Muster nicht kennt. Die Filterlogik in `buildFindAllQuery` und `getFeed` lässt sich minimal und isoliert erweitern, ohne die Struktur zu brechen.

**Verworfene Alternativen:**

- **Dedizierter POST /api/articles/:slug/publish-Endpunkt** — Verlangt einen neuen verb-basierten Route-Typ, den das Codebase-Muster nicht kennt (alle anderen Zustandsänderungen laufen über PUT). Der Gewinn an Explizitheit rechtfertigt die Abweichung vom Muster nicht.
- **Filter-Parameter am GET /api/articles-Endpunkt (?draft=true) statt dediziertem /drafts-Endpunkt** — Vom Product Owner explizit abgelehnt (Bisherige Rückfragen, Ansatz B gewählt). Ein dedizierter Endpunkt macht die Autorisierungsgrenze klar und erfordert keine Query-Parameter-Logik in bestehenden Filtern.

## Architektur und Komponenten

### src/prisma/schema.prisma

- **Zweck:** Datenbankmodell um Feld `published` erweitern
- **Schnittstelle:** Article: published Boolean @default(false)
- **Abhängigkeiten:** Keine.

### Prisma-Migration (neue Datei)

- **Zweck:** Feld `published` zur Tabelle hinzufügen und alle bestehenden Artikel auf published = true setzen, damit kein vorhandener Inhalt versteckt wird
- **Schnittstelle:** SQL-Migration: ALTER TABLE + UPDATE bestehender Zeilen auf published = true
- **Abhängigkeiten:** src/prisma/schema.prisma

### buildFindAllQuery in article.service.ts

- **Zweck:** Öffentliche Artikel-Listen (GET /articles) immer auf published = true einschränken
- **Schnittstelle:** Internes Hilfsobjekt: queries.push({ published: { equals: true } })
- **Abhängigkeiten:** src/prisma/prisma-client.ts

### getFeed in article.service.ts

- **Zweck:** Feed-Abfrage auf published = true einschränken
- **Schnittstelle:** prisma.article.findMany where-Klausel: published: true (zusätzlich zum bestehenden followedBy-Filter)
- **Abhängigkeiten:** src/prisma/prisma-client.ts

### createArticle in article.service.ts

- **Zweck:** Optionales Feld `published` aus dem Request-Body lesen und in der DB speichern
- **Schnittstelle:** Payload: { title, description, body, tagList, published?: boolean } → Prisma create mit published: article.published ?? false
- **Abhängigkeiten:** src/prisma/prisma-client.ts

### updateArticle in article.service.ts

- **Zweck:** Veröffentlichen eines Entwurfs ermöglichen; Rücksetzung eines bereits veröffentlichten Artikels verhindern
- **Schnittstelle:** Wenn published: true im Payload und Artikel ist Entwurf → Prisma update mit published: true. Wenn Artikel bereits published und published: false übergeben → Feld wird ignoriert (kein Fehler).
- **Abhängigkeiten:** src/prisma/prisma-client.ts

### getArticle in article.service.ts

- **Zweck:** Direktabruf eines Entwurfs durch Nicht-Autoren mit 403 ablehnen
- **Schnittstelle:** Nach prisma.article.findUnique: wenn article.published === false und article.authorId !== id (oder id undefined) → throw HttpException(403, { errors: { article: ['forbidden'] } })
- **Abhängigkeiten:** src/prisma/prisma-client.ts, src/app/models/http-exception.model.ts

### getDrafts in article.service.ts (neu)

- **Zweck:** Eigene Entwürfe des aufrufenden Autors abrufen
- **Schnittstelle:** getDrafts(id: number): Promise<{ articles, articlesCount }> — prisma.article.findMany where: { authorId: id, published: false }, inkl. gleicher include-Struktur wie getArticles
- **Abhängigkeiten:** src/prisma/prisma-client.ts, article.mapper.ts

### articleMapper in article.mapper.ts

- **Zweck:** Feld `published` in die API-Response aufnehmen
- **Schnittstelle:** articleMapper gibt zusätzlich published: article.published zurück
- **Abhängigkeiten:** Keine.

### article.model.ts

- **Zweck:** TypeScript-Interface um `published` ergänzen
- **Schnittstelle:** Article: published: boolean
- **Abhängigkeiten:** Keine.

### article.controller.ts

- **Zweck:** Route GET /api/articles/drafts ergänzen (vor /:slug registriert)
- **Schnittstelle:** GET /api/articles/drafts, auth.required → getDrafts(req.auth.user.id) → res.json(result)
- **Abhängigkeiten:** getDrafts in article.service.ts, src/app/routes/auth/auth.ts

### src/tests/services/article.service.test.ts

- **Zweck:** Unit-Tests für neue und geänderte Service-Funktionen
- **Schnittstelle:** Tests für getDrafts, getArticle (403-Fall), createArticle mit published, updateArticle (Veröffentlichen + Versuch Rücksetzung)
- **Abhängigkeiten:** src/tests/prisma-mock.ts

### e2e/src/server/server.spec.ts

- **Zweck:** End-to-End-Szenarien für den vollständigen Entwurfs-Lifecycle
- **Schnittstelle:** Entwurf anlegen → nicht im Feed → per /drafts abrufbar → Direktabruf durch anderen Nutzer liefert 403 → veröffentlichen → im Feed sichtbar
- **Abhängigkeiten:** Keine.

## Datenfluss

POST /api/articles mit optionalem published-Feld → createArticle speichert published (Default false) → articleMapper gibt published zurück. GET /api/articles und GET /api/articles/feed → buildFindAllQuery / getFeed ergänzt published: true Filter → nur veröffentlichte Artikel. GET /api/articles/drafts (auth.required) → getDrafts(authorId) → Prisma filtert authorId = caller UND published = false → articleMapper. GET /api/articles/:slug → getArticle → wenn Entwurf UND caller ≠ author → 403; sonst artikelMapper mit published-Feld. PUT /api/articles/:slug mit published: true → updateArticle setzt published = true wenn vorher false; bei published: false auf bereits veröffentlichtem Artikel: Feld wird nicht in das Prisma-Update-Objekt aufgenommen.

## Fehlerbehandlung

GET /api/articles/:slug auf einen Entwurf durch Nicht-Autor oder unauthentifizierten Nutzer → HttpException 403. GET /api/articles/drafts ohne gültigen JWT → 401 (bestehende auth.required-Middleware). PUT /api/articles/:slug mit published: false auf einem bereits veröffentlichten Artikel → kein Fehler, Feld wird ignoriert. Alle anderen Fehler (404 nicht gefunden, 403 falscher Autor beim Update) folgen den bestehenden Mustern.

## Akzeptanzkriterien

- POST /api/articles ohne published-Feld legt einen Artikel mit published: false an; der Artikel erscheint weder in GET /api/articles noch in GET /api/articles/feed.
- POST /api/articles mit published: true legt einen Artikel mit published: true an; er ist sofort in GET /api/articles sichtbar.
- GET /api/articles liefert niemals Artikel mit published: false, unabhängig davon, ob der Aufrufer authentifiziert ist oder nicht.
- GET /api/articles/feed liefert niemals Artikel mit published: false.
- GET /api/articles/drafts (mit gültigem JWT) liefert ausschließlich die eigenen Entwürfe des aufrufenden Nutzers (published: false, authorId = caller).
- GET /api/articles/drafts ohne gültigen JWT liefert 401.
- GET /api/articles/:slug auf einen Entwurf durch einen anderen authentifizierten Nutzer liefert 403.
- GET /api/articles/:slug auf einen Entwurf durch einen unauthentifizierten Nutzer liefert 403.
- GET /api/articles/:slug auf einen eigenen Entwurf durch den Autor liefert den Artikel mit published: false.
- PUT /api/articles/:slug mit { article: { published: true } } durch den Autor wechselt den Artikel von Entwurf zu veröffentlicht; anschließend erscheint er in GET /api/articles.
- PUT /api/articles/:slug mit { article: { published: false } } auf einem bereits veröffentlichten Artikel lässt published: true unverändert.
- Jede Artikel-Response (Einzel- und Listenendpunkte) enthält das Feld published als booleschen Wert.

## Tests

Unit-Tests (Jest + Prisma-Mock in src/tests/services/article.service.test.ts): getDrafts gibt nur Entwürfe des aufrufenden Autors zurück; getDrafts gibt leeres Array zurück wenn keine Entwürfe vorhanden; getArticle wirft HttpException 403 wenn Artikel Entwurf und caller nicht Autor; getArticle gibt Entwurf zurück wenn caller der Autor ist; createArticle speichert published: false wenn nicht angegeben; createArticle speichert published: true wenn angegeben; updateArticle setzt published: true wenn Artikel Entwurf war; updateArticle lässt published: true unverändert wenn bereits veröffentlicht und published: false übergeben. End-to-End (e2e/src/server/server.spec.ts): vollständiger Lifecycle — Entwurf anlegen, prüfen dass er nicht im öffentlichen Feed erscheint, per /drafts abrufen, Direktabruf durch anderen Nutzer ergibt 403, veröffentlichen per PUT, anschließend im öffentlichen Feed prüfen.

## Betroffene Dateien

- `src/prisma/schema.prisma` — Feld published Boolean @default(false) zum Article-Modell hinzufügen
- `src/prisma/migrations/add_published_to_article/migration.sql` (neu) — Neue Migration: Spalte published hinzufügen und alle bestehenden Zeilen auf published = true setzen
- `src/app/routes/article/article.service.ts` — buildFindAllQuery: published: true Filter ergänzen; getFeed: published: true Filter ergänzen; createArticle: published-Feld aus Payload lesen und speichern; updateArticle: published: true setzen wenn Entwurf, false ignorieren wenn bereits veröffentlicht; getArticle: 403 für Nicht-Autoren bei Entwürfen; getDrafts: neue Funktion
- `src/app/routes/article/article.mapper.ts` — Feld published in das gemappte Rückgabeobjekt aufnehmen
- `src/app/routes/article/article.model.ts` — published: boolean zum Article-Interface hinzufügen
- `src/app/routes/article/article.controller.ts` — Route GET /api/articles/drafts (auth.required) vor der /:slug-Route registrieren; getDrafts importieren
- `src/tests/services/article.service.test.ts` — Unit-Tests für getDrafts, getArticle (403-Fall), createArticle mit published-Feld, updateArticle Publish/Unpublish-Schutz hinzufügen
- `e2e/src/server/server.spec.ts` — E2E-Szenarien für Entwurf anlegen, Sichtbarkeitsregeln und Publish-Lifecycle ergänzen

## Risiken

- [hoch] Bestehende Artikel in der Datenbank würden nach der Migration den Default-Wert false erhalten und wären damit unsichtbar, sofern die Migration nicht explizit alle vorhandenen Zeilen auf published = true setzt. — Die Migration muss vor dem Setzen des Defaults alle bestehenden Artikel-Zeilen auf published = true aktualisieren. In der Prisma-Migration wird zuerst ein UPDATE auf bestehende Daten ausgeführt, dann die Spalte mit Default false angelegt.
- [niedrig] Route-Reihenfolge: GET /articles/drafts muss vor GET /articles/:slug im Express-Router registriert sein, sonst fängt der Slug-Parameter 'drafts' ab. — Analog zum bestehenden /articles/feed-Route, die bereits vor /:slug steht. Gleiche Reihenfolge für /articles/drafts einhalten.
- [mittel] Bestehende API-Clients, die POST /api/articles ohne published-Feld aufrufen, erhalten künftig einen Entwurf statt eines veröffentlichten Artikels. — In der API-Dokumentation und im PR kommunizieren. Clients müssen published: true übergeben, um das bisherige Verhalten beizubehalten. Da dies ein explizites Feature ist, ist das Breaking-Change-Risiko bekannt und akzeptiert.

## Offene Punkte

- Soll der Autor seine eigenen Entwürfe auch über GET /api/articles?author=<username> sehen können, oder ist GET /api/articles/drafts der einzige Zugang? Die Spec geht davon aus, dass GET /api/articles für alle Nutzer (inkl. dem Autor selbst) immer nur veröffentlichte Artikel liefert.

## Geklärte Rückfragen

**F:** Wie soll der Autor auf seine eigenen Entwürfe zugreifen? (Angeboten: A: Filter-Parameter am bestehenden Endpunkt; B: Dedizierter Endpunkt GET /api/articles/drafts)\
**A:** Ansatz B: Dedizierter Endpunkt GET /api/articles/drafts

**F:** Kann ein bereits veröffentlichter Artikel vom Autor wieder in den Entwurfsstatus zurückversetzt werden? Falls ja: Wird der Artikel dadurch sofort für alle anderen Nutzer unsichtbar (inkl. Feed und Direktabruf)?\
**A:** Nein, veröffentlicht ist endgültig. Ein veröffentlichter Artikel kann nicht zurück in den Entwurfsstatus.
