---
ticket: CB-1
stage: design
source: examples/tickets/cb-1-entwuerfe.md
intent: docs/work/CB-1/intent.md
---

# CB-1: Artikel als Entwurf speichern und veröffentlichen

## Ziel

Autoren können Artikel als Entwürfe anlegen, die ausschließlich für sie selbst sichtbar sind. Erst nach einer expliziten Veröffentlichungsaktion erscheint ein Artikel im globalen Feed, im Follower-Feed und über den direkten Slug für andere Nutzer. Das ermöglicht es, Inhalte vorzubereiten, ohne sie sofort zu veröffentlichen.

## Gewählter Ansatz

Ein Boolean-Feld `published` (Default `false`) wird zum `Article`-Modell hinzugefügt. Alle neuen Artikel entstehen automatisch als Entwurf. Eine neue dedizierte Route `POST /articles/:slug/publish` setzt `published: true` und folgt damit dem bestehenden Muster der Favoriten-Endpunkte. Die bestehenden List- und Feed-Abfragen werden um ein `published: true`-Filter ergänzt; ausgenommen davon ist die autorengefilterte Listenabfrage (`GET /articles?author=X`), wenn der Anfragesteller selbst der Autor ist.

**Begründung:** Ein Boolean ist die einfachste Darstellung der zwei im Intent beschriebenen Zustände. Der dedizierte Publish-Endpunkt passt zum Codebase-Muster: Favoriten nutzen ebenfalls `POST/DELETE /articles/:slug/favorite`. Die Filterung direkt in den Service-Funktionen (`buildFindAllQuery`, `getFeed`, `getArticle`) hält die Controller schlank und entspricht der bestehenden Konvention (vgl. Demo-User-Filterung).

**Verworfene Alternativen:**

- **publishedAt: DateTime? statt Boolean** — Ein optionaler Timestamp (null = Entwurf) würde den Zeitpunkt der Veröffentlichung festhalten, aber der Intent verlangt kein Publish-Datum. Der Boolean ist ausreichend und einfacher.
- **published-Feld über PUT /articles/:slug steuern** — Den bestehenden Update-Endpunkt um ein optionales `published`-Feld zu erweitern vermischt Inhaltsänderungen mit Statusübergängen. Ein dedizierter Endpunkt kommuniziert die Absicht klarer und folgt dem Codebase-Muster der Favoriten.

## Architektur und Komponenten

### src/prisma/schema.prisma

- **Zweck:** Fügt das Feld `published Boolean @default(false)` zum `Article`-Modell hinzu, um den Entwurfs- und Veröffentlichungsstatus zu speichern.
- **Schnittstelle:** Article { ..., published Boolean @default(false) }
- **Abhängigkeiten:** Keine.

### src/prisma/migrations/[timestamp]_add_published_to_article/migration.sql

- **Zweck:** Prisma-Migration: fügt die `published`-Spalte hinzu und setzt alle bestehenden Artikel auf `TRUE`, damit keine existierenden Artikel versehentlich zu Entwürfen werden.
- **Schnittstelle:** ALTER TABLE "Article" ADD COLUMN "published" BOOLEAN NOT NULL DEFAULT FALSE; UPDATE "Article" SET "published" = TRUE;
- **Abhängigkeiten:** src/prisma/schema.prisma

### article.service.ts → createArticle

- **Zweck:** Legt jeden neuen Artikel mit `published: false` an. Der Caller übergibt kein `published`-Feld; der Entwurfsstatus ist immer der Startzustand.
- **Schnittstelle:** createArticle(article: { title, description, body, tagList }, userId: number): Promise<ArticleResponse>
- **Abhängigkeiten:** src/prisma/schema.prisma

### article.service.ts → buildFindAllQuery + getArticles

- **Zweck:** Blendet Entwürfe in der globalen Artikelliste aus. Wenn `query.author` gesetzt ist und die Anfrage vom selben Nutzer kommt (Autorenname wird per Prisma-Join mit der userId abgeglichen), werden eigene Entwürfe über `OR: [{ published: true }, { authorId: userId }]` eingeschlossen. Für alle anderen Fälle gilt `published: true` als harte Bedingung.
- **Schnittstelle:** getArticles(query: { offset?, limit?, tag?, author?, favorited? }, userId?: number): Promise<{ articles, articlesCount }>
- **Abhängigkeiten:** src/prisma/schema.prisma, article.mapper.ts

### article.service.ts → getFeed

- **Zweck:** Ergänzt die Follower-Feed-Abfrage um `published: true`, damit Entwürfe gefolgerter Autoren nicht im Feed erscheinen.
- **Schnittstelle:** getFeed(offset: number, limit: number, userId: number): Promise<{ articles, articlesCount }>
- **Abhängigkeiten:** src/prisma/schema.prisma, article.mapper.ts

### article.service.ts → getArticle

- **Zweck:** Prüft nach dem Datenbankaufruf, ob der Artikel unveröffentlicht ist. Wenn `published === false` und der Anfragesteller nicht der Autor ist, wird HttpException(404) geworfen, um die Existenz des Entwurfs nicht preiszugeben.
- **Schnittstelle:** getArticle(slug: string, userId?: number): Promise<ArticleResponse>
- **Abhängigkeiten:** src/prisma/schema.prisma, article.mapper.ts

### article.service.ts → publishArticle

- **Zweck:** Neue Service-Funktion: setzt `published: true` für einen Artikel. Prüft vorher, ob der Anfragesteller der Autor ist.
- **Schnittstelle:** publishArticle(slug: string, userId: number): Promise<ArticleResponse>
- **Abhängigkeiten:** src/prisma/schema.prisma, article.mapper.ts

### article.controller.ts

- **Zweck:** Fügt die neue Route `POST /articles/:slug/publish` hinzu, gesichert mit `auth.required`. Ruft `publishArticle` auf und gibt das Artikel-Objekt zurück.
- **Schnittstelle:** POST /articles/:slug/publish (auth.required) → { article: ArticleResponse }
- **Abhängigkeiten:** article.service.ts → publishArticle

### article.mapper.ts

- **Zweck:** Ergänzt den gemappten Artikel-Output um das Feld `published: article.published`, damit Clients den Status erkennen können.
- **Schnittstelle:** articleMapper(article, id?) → { slug, title, description, body, tagList, createdAt, updatedAt, favorited, favoritesCount, author, published }
- **Abhängigkeiten:** src/prisma/schema.prisma

## Datenfluss

Erstellen: POST /articles → Controller → createArticle(payload, userId) → prisma.article.create({ published: false }) → articleMapper → 201 { article: { ..., published: false } }. Globale Liste: GET /articles → getArticles(query, userId?) → WHERE published = true (AND bestehende Demo/Author-Filter) → articleMapper pro Artikel → Response. Autorenliste (eigene Entwürfe): GET /articles?author=X (als X eingeloggt) → getArticles({ author: X }, userId) → WHERE (published = true OR authorId = userId) AND author.username = X → schließt eigene Entwürfe ein → articleMapper → Response. Feed: GET /articles/feed → getFeed(offset, limit, userId) → WHERE followedBy.some(id) AND published = true → articleMapper → Response. Einzelabruf (Entwurf, Fremd): GET /articles/:slug → getArticle(slug) → Artikel aus DB → published === false AND userId ≠ authorId → HttpException(404). Einzelabruf (Entwurf, eigen): GET /articles/:slug → getArticle(slug, userId) → published === false AND userId === authorId → articleMapper → { article: { ..., published: false } }. Veröffentlichen: POST /articles/:slug/publish → publishArticle(slug, userId) → Autorenprüfung → prisma.article.update({ published: true }) → articleMapper → { article: { ..., published: true } }.

## Fehlerbehandlung

getArticle: Entwurf + Anfragesteller ist nicht Autor → HttpException(404, { errors: { article: ['not found'] } }) (kein 403, um Existenz nicht preiszugeben). publishArticle: Artikel nicht gefunden → 404; Anfragesteller ist nicht Autor → 403; Artikel bereits veröffentlicht → idempotent, gibt aktuellen Zustand zurück (kein Fehler). Fehlende Authentifizierung an auth.required-Routen → 401 (durch bestehendes express-jwt-Middleware, kein zusätzlicher Code nötig).

## Akzeptanzkriterien

- POST /articles erstellt einen Artikel mit published: false; die Antwort enthält das Feld `published: false`.
- GET /articles gibt Entwürfe für keinen Anfragesteller zurück — weder für unangemeldete Nutzer noch für den Autor selbst.
- GET /articles/feed gibt Entwürfe gefolgerter Autoren nicht zurück.
- GET /articles?author=X gibt die eigenen Entwürfe des Autors X zurück, wenn der Anfragesteller als X eingeloggt ist.
- GET /articles?author=X gibt keine Entwürfe zurück, wenn der Anfragesteller ein anderer Nutzer oder unangemeldet ist.
- GET /articles/:slug gibt 404 zurück, wenn der Artikel ein Entwurf ist und der Anfragesteller nicht der Autor ist.
- GET /articles/:slug gibt den Artikel mit published: false zurück, wenn der Anfragesteller der Autor ist.
- POST /articles/:slug/publish setzt published: true; danach ist der Artikel über GET /articles/:slug für alle abrufbar und erscheint in GET /articles sowie GET /articles/feed.
- POST /articles/:slug/publish gibt 403 zurück, wenn der Anfragesteller nicht der Autor des Artikels ist.
- POST /articles/:slug/publish gibt 401 zurück, wenn kein gültiges JWT mitgeschickt wird.
- Alle Artikel-Antwort-Objekte enthalten ein Boolean-Feld `published`.

## Tests

Unit (src/tests/services/article.service.test.ts): Testfall für getArticle — wirft 404, wenn Artikel unveröffentlicht und userId ≠ authorId; Testfall für publishArticle — gibt Artikel mit published: true zurück (Happy Path) und wirft 403, wenn userId ≠ authorId. Alle bestehenden Mock-Fixtures (z. B. mockedArticleResponse in favoriteArticle-Tests) um `published: true` erweitern. E2E (e2e/src/server/server.spec.ts): Entwurf erstellen → prüfen, dass er nicht in GET /articles erscheint; als Autor über GET /articles?author=X prüfen, dass Entwurf sichtbar ist; als anderer Nutzer GET /articles/:slug aufrufen und 404 erwarten; Entwurf veröffentlichen und anschließend Sichtbarkeit in globaler Liste und Feed verifizieren; Publish-Versuch durch Nicht-Autor → 403.

## Betroffene Dateien

- `src/prisma/schema.prisma` — Fügt `published Boolean @default(false)` zum Article-Modell hinzu.
- `src/prisma/migrations/[timestamp]_add_published_to_article/migration.sql` (neu) — Neue Migration: Spalte hinzufügen, bestehende Artikel auf published = TRUE setzen.
- `src/app/routes/article/article.service.ts` — createArticle: setzt published: false. getArticles/buildFindAllQuery: published-Filter + Sonderfall Eigenautoren. getFeed: published: true Filter. getArticle: 404-Guard für Entwürfe. Neue Funktion publishArticle.
- `src/app/routes/article/article.controller.ts` — Neue Route POST /articles/:slug/publish mit auth.required, ruft publishArticle auf.
- `src/app/routes/article/article.mapper.ts` — Ergänzt published: article.published im gemappten Output.
- `src/tests/services/article.service.test.ts` — Bestehende Mock-Fixtures um published: true erweitern; neue Testfälle für getArticle (404 für Entwurf) und publishArticle.

## Risiken

- [hoch] Bestehende Artikel werden nach der Migration zu Entwürfen, wenn das UPDATE-Statement in der Migrations-SQL fehlt oder fehlschlägt. — Die Migration muss vor dem ALTER TABLE ein UPDATE "Article" SET published = TRUE enthalten. Dieses Statement ist Teil der Akzeptanzkriterien für die Migration selbst.
- [mittel] Bestehende Unit-Test-Mocks (z. B. mockedArticleResponse in article.service.test.ts) enthalten kein published-Feld; nach der Mapper-Änderung liefern betroffene Tests undefined statt eines Boolean. — Alle Mock-Fixtures im Unit-Test-File um published: true erweitern; dies ist explizit in den Test-Anforderungen spezifiziert.
- [niedrig] Die Endpunkte addComment und favoriteArticle prüfen den published-Status nicht — ein Nutzer, der einen Entwurfs-Slug kennt, könnte theoretisch kommentieren oder favorisieren. — Der Entwurfs-Slug ist nicht öffentlich exponiert (GET /articles/:slug gibt 404 zurück); das Risiko ist gering. Die Klärung ist als Open Point dokumentiert.

## Offene Punkte

- Kann ein bereits veröffentlichter Artikel wieder in den Entwurfsstatus zurückversetzt werden? Der aktuelle Entwurf behandelt die Veröffentlichung als Einbahnstraße. Ein Unpublish-Mechanismus wäre eine separate Anforderung.
- Dürfen andere Nutzer einen Entwurf favorisieren oder kommentieren, wenn sie den Slug kennen? Die addComment- und favoriteArticle-Endpunkte sind bisher nicht gegen Entwürfe abgesichert.

## Geklärte Rückfragen

Keine.
