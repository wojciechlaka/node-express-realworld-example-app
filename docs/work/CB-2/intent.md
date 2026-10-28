---
ticket: CB-2
stage: plan
source: examples/tickets/cb-2-lesezeit.md
---

# CB-2: Lesezeit pro Artikel in API-Response ergänzen

## Problem

Leser erhalten über die API aktuell keine Information darüber, wie lange das Lesen eines Artikels in Anspruch nehmen wird. Sie können die Länge eines Textes erst nach dem Öffnen einschätzen. Für Nutzer, die begrenzte Zeit haben oder Inhalte gezielt auswählen möchten, fehlt damit ein wesentlicher Orientierungshinweis.

## Gewünschtes Ergebnis

Jede API-Antwort, die einen Artikel enthält, liefert einen zusätzlichen Wert mit der geschätzten Lesezeit. Das Frontend kann diesen Wert ohne eigene Berechnung direkt darstellen. Ob ein Leser einen Artikel öffnet oder überspringt, kann damit bereits auf der Übersichtsseite entschieden werden.

## Betroffene

- Leser
- Article-API

## Offene Punkte

- Soll die Lesezeit in der Artikelliste (Feed) und beim Einzelabruf gleichermaßen erscheinen, oder nur an einer der beiden Stellen?
- Welcher Wortdurchsatz soll der Berechnung zugrunde liegen (z. B. 200 Wörter/Minute), und soll das Ergebnis als ganze Minutenzahl oder als Bereich (z. B. "3–5 min") geliefert werden?
- Gibt es eine Mindest-Lesezeit, die angezeigt werden soll, wenn ein Artikel sehr kurz ist (z. B. "< 1 min" statt 0)?

## Geklärte Rückfragen

Keine.
