---
ticket: CB-2
stage: plan
source: examples/tickets/cb-2-lesezeit.md
---

# CB-2: Lesezeit in Artikel-API-Antworten ergänzen

## Problem

Leser sehen beim Stöbern in der Artikelliste nicht, wie lang ein Artikel ist. Sie müssen ihn erst öffnen, um zu entscheiden, ob sie gerade genug Zeit haben. Das erhöht die Hemmschwelle und kann zu Frustration führen, wenn die verfügbare Zeit nicht reicht. Die API liefert aktuell keinen Hinweis auf den zeitlichen Umfang eines Artikels.

## Gewünschtes Ergebnis

Die API gibt für Artikel einen geschätzten Lesezeit-Wert zurück, den das Frontend in der Artikelliste und/oder im Artikel-Detail anzeigen kann — ohne selbst eine Berechnung implementieren zu müssen.

## Erfolgskriterien

- Leser sehen die geschätzte Lesezeit, bevor sie einen Artikel öffnen.
- Die Lesezeit ist als maschinenlesbarer Wert in der API-Antwort enthalten.
- Die angezeigte Lesezeit stimmt erkennbar mit der tatsächlichen Länge des Artikels überein.

## Betroffene

- Leser
- Frontend
- Artikel-API

## Offene Punkte

- In welcher Einheit soll die Lesezeit zurückgegeben werden — z. B. ganze Minuten als Zahl, Sekunden, oder ein vorformatierter String wie '3 min read'?
- Soll die Lesezeit bei allen Artikel-Endpunkten mitgeliefert werden (Liste und Einzel-Artikel) oder nur bei einem davon?
- Welche durchschnittliche Lesegeschwindigkeit (Wörter pro Minute) soll der Berechnung zugrunde liegen?
- Wie soll mit sehr kurzen Artikeln umgegangen werden, bei denen die Lesezeit unter einer Minute liegt?

## Geklärte Rückfragen

Keine.
