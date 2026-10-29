---
ticket: CB-1
stage: plan
source: examples/tickets/cb-1-entwuerfe.md
---

# CB-1: Artikel als Entwurf speichern

## Problem

Autoren können Artikel heute nur sofort veröffentlichen – ein Zwischenspeichern ohne Sichtbarkeit für andere Nutzer ist nicht möglich. Dadurch müssen Autoren einen Artikel in einem Zug fertigstellen und können ihn nicht in Ruhe vorbereiten oder überarbeiten, bevor er öffentlich sichtbar wird.

## Gewünschtes Ergebnis

Ein Autor kann einen Artikel erstellen oder speichern, ohne ihn zu veröffentlichen. Solange ein Artikel im Entwurfs-Zustand ist, erscheint er weder im globalen Feed noch in anderen öffentlichen Artikel-Listen und ist für andere Benutzer nicht abrufbar. Nur der Autor selbst sieht seine eigenen Entwürfe.

## Betroffene

- Autoren
- Leser (öffentlicher Feed)
- Artikel-API (findAll, create, update)

## Offene Punkte

- Wie veröffentlicht ein Autor einen Entwurf — über eine explizite Publish-Aktion oder durch ein Update-Feld im bestehenden PUT-Endpunkt?
- Gibt es eine dedizierte Ansicht/Liste, in der der Autor seine eigenen Entwürfe sehen kann, oder reicht es, dass Entwürfe im eigenen Profil-Tab sichtbar sind?
- Können Entwürfe über ihren Slug direkt aufgerufen werden — z. B. wenn ein Autor einen Link teilt — oder sind sie auch per direktem URL-Zugriff für alle anderen gesperrt?
- Bleibt ein einmal veröffentlichter Artikel dauerhaft veröffentlicht, oder soll es auch möglich sein, einen Artikel wieder in den Entwurfs-Zustand zurückzusetzen?

## Geklärte Rückfragen

Keine.
