---
ticket: CB-2
stage: plan
source: examples/tickets/cb-2-lesezeit.md
---

# CB-2: Lesezeit in der Artikel-API zurückgeben

## Problem

Leser erhalten beim Abrufen von Artikeln über die API aktuell keine Information darüber, wie lange das Lesen eines Artikels ungefähr dauert. Dadurch können Leser nicht vorab entscheiden, ob sie zu diesem Zeitpunkt genug Zeit für den Artikel haben. Das Frontend kann diesen Wert nicht anzeigen, weil er in keiner API-Antwort enthalten ist.

## Gewünschtes Ergebnis

Die Artikel-API liefert für jeden Artikel einen zusätzlichen Wert mit der geschätzten Lesezeit. Frontend-Clients können diesen Wert auslesen und dem Leser anzeigen, ohne ihn selbst berechnen zu müssen.

## Betroffene

- Leser
- Frontend-Clients (Artikel-API-Konsumenten)
- Artikel-API

## Offene Punkte

- Soll die Lesezeit auch in der Artikel-Listenansicht (GET /articles) zurückgegeben werden, oder nur beim Abrufen eines einzelnen Artikels?
- In welchem Format soll die Lesezeit zurückgegeben werden – als Ganzzahl (Minuten), als Dezimalzahl (Sekunden) oder als formatierter String (z. B. '5 min read')?
- Gibt es eine vorgegebene Leserate (Wörter pro Minute), die als Berechnungsgrundlage dienen soll, oder soll der Industriestandard (~200–250 Wörter/Minute) verwendet werden?
- Soll der Wert dauerhaft in der Datenbank gespeichert oder bei jedem Request dynamisch aus dem Artikelinhalt berechnet werden?

## Geklärte Rückfragen

Keine.
