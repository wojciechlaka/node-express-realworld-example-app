---
ticket: CB-1
stage: plan
source: examples/tickets/cb-1-entwuerfe.md
---

# CB-1: Artikel als Entwurf speichern und veröffentlichen

## Problem

Autoren können Artikel derzeit nicht vorbereiten, ohne sie sofort zu veröffentlichen. Sobald ein Artikel angelegt wird, ist er unmittelbar für alle Nutzer sichtbar — im globalen Feed, im Feed der Follower und über den direkten Slug-Aufruf. Das `Article`-Modell kennt keinen Entwurfs- oder Veröffentlichungsstatus. Autoren, die einen Text überarbeiten oder zu einem geplanten Zeitpunkt veröffentlichen wollen, haben dadurch keine Möglichkeit, unfertige Inhalte zurückzuhalten.

## Gewünschtes Ergebnis

Autoren können einen Artikel anlegen, der zunächst nur für sie selbst sichtbar ist. Sie entscheiden selbst, wann der Artikel für alle Nutzer freigegeben wird. Erst nach der expliziten Veröffentlichung erscheint der Artikel in Feeds und Listings anderer Nutzer.

## Erfolgskriterien

- Ein Autor kann einen Artikel erstellen, der für andere Nutzer — einschließlich nicht angemeldeter Besucher — nicht sichtbar ist.
- Entwürfe erscheinen weder im globalen Artikel-Feed noch im Feed der Follower des Autors.
- Der Autor sieht seine eigenen Entwürfe in seiner persönlichen Artikelübersicht.
- Der Autor kann einen Entwurf veröffentlichen; danach ist der Artikel für alle Nutzer abrufbar und erscheint in den Feeds.
- Kein anderer Nutzer kann einen Entwurf über den direkten Slug oder über Listings aufrufen.

## Betroffene

- Autoren
- Leser
- Artikel-API

## Offene Punkte

- Kann ein bereits veröffentlichter Artikel wieder in den Entwurfsstatus zurückversetzt werden, oder ist die Veröffentlichung eine Einbahnstraße?
- Sollen Entwürfe für den Autor in denselben Listen wie veröffentlichte Artikel auftauchen (z. B. bei Filterung nach Tag oder Favorit), oder gibt es eine separate Entwurfs-Ansicht?
- Ist es erlaubt, einen Entwurf zu favorisieren oder zu kommentieren — oder sind diese Aktionen erst nach der Veröffentlichung möglich?

## Geklärte Rückfragen

Keine.
