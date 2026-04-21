In Angular gibt es zahlreiche Ansätze und Möglichkeiten, Daten über REST-API-Aufrufe abzurufen und den Zustand dieser Daten innerhalb der Anwendung zu verwalten. Dieses Thema hat mich dazu motiviert, eine Artikelserie zu schreiben, in der ich einige dieser Ansätze näher erläutere und ihre Hintergründe beleuchte.

Angular-Schulungen vermitteln nur grundlegende Konzepte z.B. einen rudimentären Rest-Call ausführen, Vermittlung von Konzepten für Datenverwaltung in einer komplexen Anwendungsumgebung ist meistens nicht teil dieser Art von Schulungen. Zurück im Projektleben ist der Entwickler mit vielen Herausforderungen konfrontiert.

In Projekten auf grüner Wiese hat man die Schwierigkeit sich für ein Vorgehen zu entscheiden. Fragestellungen wie nimmt man ein bereits etabliertes Vorgehen oder setzt man auf neue Konzepte.

Bei den laufenden Projekten ist das Vorgehen im Besten Fall bereits vorgegeben. Etablierte Ansätze werden verwendet. Bereits in der späten Entwicklungsphase  werden häufig Patterns vermischt, da hohe Fluktuation im Projekt herscht und neue Entwickler ihren favorisierten Ansatz umsetzen  statt das definierte Vorgehen zu verfolgen weil sie damit bisher keine Berührungspunkte hatten.

In die Jahre gekommene Projekten die in Wartung sind, trifft er auf ältere Praktiken beispielsweise einen manuellen subscribe()  weil die async-pipe zur damalgen Zeit nicht gab.

In Lagacy-Projekten trifft man auf den Einsatz von verschiedenen konkurierenden Pattern zur Datenermittlung und Zustandsverwaltung in einer Anwendung.

Der Entwickler muss evolutionsbeding eine ganze Breite an Praktiken beherschen.

Diese Artikelserie verschafft einen Überblick, stellt in kurzen Teil-Artikeln jeweilige Ansätze vor mit Code-Beispielen, zeigt auf was man beachten muss und Gibt Empfehlungen.