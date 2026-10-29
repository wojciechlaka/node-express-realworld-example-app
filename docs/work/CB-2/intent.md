---
ticket: CB-2
stage: plan
source: examples/tickets/cb-2-lesezeit.md
---

# CB-2: Lesezeit in der Artikel-API-Antwort bereitstellen

## Problem

Leser können vor dem Öffnen eines Artikels nicht einschätzen, wie viel Zeit sie investieren müssen. Die aktuelle Artikel-API liefert keinen Hinweis auf die Lesedauer, sodass das Frontend diesen Wert nicht anzeigen kann. Das betrifft sowohl die Artikelliste als auch die Einzelansicht.

## Gewünschtes Ergebnis

Jede Artikel-Antwort der API enthält einen Wert für die geschätzte Lesezeit. Das Frontend kann diesen Wert ohne eigene Berechnung direkt anzeigen, sodass Leser vor dem Öffnen eines Artikels sehen, wie lange das Lesen ungefähr dauert.

## Betroffene

- Leser
- Artikel-API

## Offene Punkte

- Soll die Lesezeit nur in der Detailansicht eines Artikels erscheinen oder auch in der Listenansicht (GET /articles)?
- In welchem Format soll der Wert geliefert werden — als Ganzzahl in Minuten (z. B. 5) oder als fertig formatierter String (z. B. "5 min")?
- Welche Wörter-pro-Minute-Rate soll als Grundlage dienen, und wie soll auf volle Minuten gerundet werden (z. B. immer aufrunden, kaufmännisch runden, Minimum 1 Minute)?

## Geklärte Rückfragen

Keine.
