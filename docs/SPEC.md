# deal-o-mat – Spezifikation

deal-o-mat hilft einem Nutzer beim Verkaufen gebrauchter Sachen. Beispiel: zehn Lego-Sets sollen auf [Kleinanzeigen](https://www.kleinanzeigen.de) verkauft werden. Im Handy-Browser werden die Sets fotografiert und kurz beschrieben, deal-o-mat stellt ein paar Rückfragen, recherchiert auf Wunsch den Wert mit Quellen, schreibt den Anzeigentext im gewünschten Stil, bereitet die Fotos auf und legt alles zum Übernehmen in die Anzeige bereit. Danach erinnert er an den Verkauf und meldet sich über Onkel Bummel.

- **Server:** C# / .NET 10 (ASP.NET Core) in Docker im Heimnetz, per HTTPS im Tailnet erreichbar
- **Client:** Web-App fürs Handy (Vue 3, TypeScript, Tailwind, Pinia), vom Server ausgeliefert, auf dem Home-Bildschirm installierbar
- **Modelle:** lokal auf oi-o-mat über die LLM-Schranke ([Marcel-B/scripts → `docs/llm-gate.md`](https://github.com/Marcel-B/scripts/blob/main/docs/llm-gate.md))
- **Benachrichtigungen:** [Onkel Bummel](https://github.com/Marcel-B/onkel-bummel), Topic `dealomat`
- **Anmeldung:** über den eigenen Identity-Server, wie bei Onkel Bummel

Nicht im Umfang: mehrere Nutzer, ein eigener Marktplatz, automatisches Einstellen oder Auslesen bei Kleinanzeigen, automatische Antworten an Käufer, native iOS-App (kann später folgen).

## Was bei Kleinanzeigen geht und was nicht

Geprüft am 15.11.2026.

| Frage | Ergebnis | Quelle |
|---|---|---|
| Gibt es eine API, über die ein privater Verkäufer Anzeigen einstellen kann? | Nein. Die Schnittstelle ist intern und für Partner mit Vertrag, ein Entwicklerprogramm gibt es nicht. Die Nutzungsbedingungen kennen keine Schnittstelle für Nutzer. | [Stack Overflow (Antwort eines Mitarbeiters, älter)](https://stackoverflow.com/questions/43802316/ebay-kleinanzeigen-api-access-ebay-classifieds-germany), [Nutzungsbedingungen](https://themen.kleinanzeigen.de/nutzungsbedingungen/) |
| Darf deal-o-mat die Seite automatisch bedienen oder auslesen? | Nein. § 5 Nr. 1 untersagt, „ohne die ausdrückliche schriftliche Zustimmung von Kleinanzeigen Crawler, Spider, Scraper oder andere automatisierte Mechanismen zu nutzen, um auf die Kleinanzeigen-Dienste zuzugreifen und Inhalte zu sammeln“. | [Nutzungsbedingungen, Stand 17.02.2024](https://themen.kleinanzeigen.de/nutzungsbedingungen/) |
| Darf deal-o-mat sich mit Marcels Konto anmelden? | Nein. Zugangsdaten sind geheim zu halten (§ 2 Nr. 4), und es wäre wieder ein automatisierter Zugriff. | wie oben |
| Sperrt die `robots.txt` Konto und Nachrichten? | Ja, unter anderem `/api`, `/messages/` und die Kontoseiten. | [robots.txt](https://www.kleinanzeigen.de/robots.txt) |

Daraus folgt:

- **Einstellen bleibt Handarbeit**, deal-o-mat macht sie so kurz wie möglich (siehe [Übergabe](#übergabe)).
- **Die Preisrecherche liest Kleinanzeigen nicht aus.** Sie nutzt andere Quellen.
- **Überwachen geht nur über das, was Kleinanzeigen von sich aus schickt**, also Mails (siehe [Überwachung](#überwachung)). Wie oft eine Anzeige gemerkt wurde, sieht man nur im eigenen Konto; das bekommt deal-o-mat nicht mit.

Kleinanzeigen hat seit Ende 2025 selbst einen KI-Anzeigenassistenten ([onlinemarktplatz.de, 11.07.2026](https://onlinemarktplatz.de/268756/kleinanzeigen-ki/)). deal-o-mat unterscheidet sich durch die Recherche mit Quellen, mehrere Artikel in einem Rutsch, die Fotobearbeitung, eigene Schreibstile und dadurch, dass alles lokal läuft.

## Konzepte

- **Verkauf:** eine Sammlung von Artikeln, die zusammen erfasst werden („Lego aus dem Keller“). Trägt die Vorgaben für alle Artikel: Plattform, Schreibstil, Abholung oder Versand, Ort.
- **Artikel:** eine Sache, die eine eigene Anzeige bekommt. Hat Fotos, die Kurzbeschreibung des Nutzers, Antworten auf Rückfragen, eine Recherche, einen Anzeigenentwurf und einen Status. Mehrere Artikel lassen sich zu einem Paket zusammenlegen, das dann eine gemeinsame Anzeige bekommt.
- **Foto:** das Original und eine bearbeitete Fassung. Das Original bleibt immer erhalten.
- **Recherche:** eine Preisspanne mit Empfehlung und einer Liste von **Quellen**. Jede Quelle hat URL, Titel, Abrufzeit, den gefundenen Preis und den Zustand, auf den er sich bezieht.
- **Anzeigenentwurf:** Titel, Text, Preis (Festpreis oder VB), Vorschlag für die Kategorie, Reihenfolge der Fotos.
- **Schreibstil:** eine benannte Vorgabe für den Text, etwa „sachlich“, „freundlich“, „knapp“. Eigene Stile lassen sich mit einem Beispieltext anlegen.
- **Status eines Artikels:** `erfasst` → `in Arbeit` → `bereit` → `eingestellt` → `verkauft` (oder `zurückgezogen`).

## Ablauf

1. **Erfassen.** Neuer Verkauf, je Artikel Fotos aufnehmen oder aus der Mediathek wählen und eine Kurzbeschreibung eintippen oder diktieren („Lego 75192 Millennium Falcon, vollständig, mit Karton“).
2. **Rückfragen.** Das Modell sieht Fotos und Beschreibung und stellt höchstens fünf Fragen, die für Preis und Text wichtig sind (Zustand, Vollständigkeit, Originalverpackung, Nichtraucherhaushalt, Versand). Jede lässt sich überspringen.
3. **Bestätigen.** Der Nutzer sieht die Angaben, wählt, ob recherchiert und ob die Fotos bearbeitet werden sollen, und startet.
4. **Arbeiten.** Recherche, Text und Fotos laufen als Auftrag im Hintergrund. Die Schranke kann Anfragen lange zurückhalten, deshalb wartet niemand vor dem Bildschirm: Ist ein Verkauf fertig, kommt eine Nachricht über Onkel Bummel mit Link.
5. **Prüfen.** Alles ist änderbar: Preis, Titel, Text (auch „noch einmal, kürzer“), Zuschnitt, Reihenfolge der Fotos.
6. **Übergeben** und danach **begleiten**, siehe unten.

## Preisrecherche

Quellen sind der Kern. Regeln:

- **Kein Preis ohne Quelle.** Das Modell bekommt nur die abgerufenen Treffer und darf nur daraus zitieren. Der Server prüft, dass jeder genannte Preis in einem Treffer steht; sonst fällt er weg.
- Jede Quelle wird mit Link, Abrufzeit und Zustand (neu, gebraucht, verkauft, angeboten) gezeigt. Angebotspreise und tatsächlich erzielte Preise stehen getrennt, weil Angebote meist zu hoch liegen.
- Das Ergebnis ist eine Spanne, eine Empfehlung und ein Satz zur Begründung. Findet sich zu wenig, sagt deal-o-mat das, statt zu schätzen.
- Kleinanzeigen wird nicht ausgelesen. Infrage kommen eine Websuche, Preisvergleiche und Fachseiten je Warengruppe (für Lego etwa Preisdatenbanken zu Set-Nummern) sowie eBay über dessen offizielle Schnittstelle. Welche Quellen sich sauber abrufen lassen, klärt Stufe 1 an den Lego-Sets.
- Die Recherche ist abwählbar. Dann trägt der Nutzer den Preis selbst ein.

## Anzeigentext

Titel (Kleinanzeigen begrenzt die Länge, der Entwurf hält sie ein) und Text aus den Angaben, im gewählten Stil. Der Text nennt nur, was der Nutzer angegeben oder bestätigt hat; was das Modell nur auf den Fotos zu sehen meint, erscheint als Rückfrage und nicht als Behauptung. Am Ende stehen die üblichen Hinweise (Privatverkauf, Abholung oder Versand) aus den Vorgaben des Verkaufs.

## Fotos

- Beim Hochladen: richtig drehen, Standort und andere Metadaten entfernen, auf eine vernünftige Größe verkleinern.
- Auf Wunsch: Zuschnitt auf den Artikel (Vorschlag vom Modell, von Hand verschiebbar), Helligkeit und Kontrast angleichen.
- Kein Verändern des Artikels selbst, kein Wegretuschieren von Mängeln. Die Anzeige soll zeigen, was verkauft wird.
- Das Modell schlägt das Titelbild vor und meldet unscharfe oder doppelte Fotos.

## Übergabe

Je Artikel eine Seite „Anzeige“, gebaut für das Handy:

- Titel, Preis und Text mit je einem Knopf zum Kopieren.
- Fotos über das Teilen-Menü des Handys (Web Share) in die Mediathek oder direkt an eine App; als Rückfall ein Download.
- Ein Knopf öffnet Kleinanzeigen bei „Anzeige aufgeben“.
- Danach trägt der Nutzer den Link der Anzeige ein, der Artikel ist `eingestellt`.

Ob die Kleinanzeigen-App geteilte Fotos annimmt, zeigt der Versuch in Stufe 2. Die Übergabe steckt hinter einer Schnittstelle je Plattform, damit später eine mit echter API (eBay) dazukommen kann.

## Überwachung

In zwei Teilen:

1. **Begleiten, geht sicher:** deal-o-mat kennt den Status jedes Artikels und meldet sich über Onkel Bummel, etwa „seit 10 Tagen eingestellt, Preis senken?“ mit Actions („5 € runter“, „verkauft“, „später“). Bei „verkauft“ fragt er nach dem erzielten Preis; daraus entsteht die Übersicht, was ein Verkauf gebracht hat.
2. **Nachrichten von Interessenten, zu prüfen:** Kleinanzeigen verschickt Mails zu Anfragen. Leitet das Postfach sie an eine Adresse weiter, die deal-o-mat liest, kann er sie dem Artikel zuordnen und über Onkel Bummel melden. Ob die Mails genug enthalten (Anzeige, Text der Nachricht) und sich abonnieren lassen, wenn die App Pushes bekommt, zeigt erst eine echte Anzeige. Geantwortet wird immer von Hand in der Kleinanzeigen-App.

Beobachter und Aufrufe einer Anzeige stehen nur im eigenen Konto und bleiben außen vor.

## Modelle

deal-o-mat meldet sich bei der LLM-Schranke als `deal-o-mat` an, Priorität 20. Vorgesehen sind die Modelle, die dort schon für andere Apps laufen: ein Modell mit Bildverständnis für Fotos und Rückfragen, ein kleineres für Text. Die App verträgt lange Wartezeiten, abgebrochene Antworten und die feste Kontextlänge, wie in `llm-gate.md` beschrieben. Jeder Schritt eines Auftrags wird einzeln gespeichert und bei Abbruch wiederholt.

Für die Websuche braucht der Server einen Suchdienst, den er selbst abfragt; das Modell wertet nur die Treffer aus.

## Stufen

1. **Kern:** Erfassen, Rückfragen, Recherche mit Quellen, Anzeigentext, Meldung über Onkel Bummel. Getestet mit den Lego-Sets.
2. **Fotos und Übergabe:** Aufbereitung, Zuschnitt, Seite „Anzeige“ mit Kopieren und Teilen.
3. **Überwachung:** Begleiten mit Erinnerungen und Actions, danach der Versuch mit den Mails.

## Offene Punkte

- Welcher Suchdienst liefert die Treffer für die Recherche, und welche Preisquellen lassen sich je Warengruppe erlaubt abrufen?
- Reichen die vorhandenen Modelle für brauchbare Rückfragen und Texte, oder braucht es ein anderes Textmodell?
- Nimmt die Kleinanzeigen-App geteilte Fotos für eine neue Anzeige an?
- Was steht in den Mails von Kleinanzeigen, und lassen sie sich zuverlässig einer Anzeige zuordnen?
- Topic und Publish-Token in Onkel Bummel, Client im Identity-Server, Zeile in `oi-o-mat.md`.
