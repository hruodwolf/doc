# Teil 2: Die schlanke async‑Pipe – Daten anzeigen ohne Subscribe

Gerade in wachsenden Komponenten wird schnell sichtbar, dass der manuelle Umgang mit Subscriptions keinen guten Skalierungsweg darstellt.

Im ersten Teil dieser Serie haben wir uns das Ur‑Pattern für REST‑Calls in Angular angesehen: den manuellen `subscribe()`.

Wir haben die Hintergründe beleuchtet, anhand von Code‑Beispielen gezeigt, wie dieser Ansatz aussieht, und erklärt, warum er über viele Jahre hinweg so häufig eingesetzt wurde.

In diesem Beitrag betrachten wir nun die Kehrseite des manuellen Subscribe‑Patterns: die wachsende Komplexität, die dadurch in Komponenten entstehen kann. Außerdem zeigen wir, wie sich diese Komplexität durch den Einsatz der `async`‑Pipe deutlich reduzieren lässt – und wie dadurch übersichtlicher und wartbarer Angular‑Code entsteht.
``


Analyse Code 3
welche Probleme / Nachteile hat es


und jetzt kommt die async-Pipe ins spiel

Verbesserter Code mit async-Pipe





Empfehlungen & nächster Schritt
Was du sehr gut triffst:
	• historische Einordnung
	• realistische Projektperspektive
	• differenzierte Aussage („nicht falsch, aber …“)
Kleiner Verbesserungsvorschlag für Artikel 2:
	• Kontrast klar machen:
		• Artikel 1: Imperativ & Lifecycle‑Verantwortung beim Entwickler
Artikel 2: Deklarativ & Lifecycle‑Management durch Angular

Aussage aus: https://blog.angular-university.io/angular-reactive-templates/
Because now we have here the state stored on this variable at the level of the component, we might be tempted to further write code that mutates that state.
Here in this small example, it would not cause an issue, but in a larger application, this could potentially cause some maintainability problems.
=> D.h. Zustand in der Komponente ist nicht optimal, weil man diesen potenziel verändern kann



Die async‑Pipe existiert bereits seit Angular 2, wurde in vielen Projekten jedoch lange Zeit kaum konsequent eingesetzt.

Oder noch knackiger:

Obwohl die async‑Pipe seit Angular 2 verfügbar ist, dominiert in vielen bestehenden Codebases bis heute der manuelle subscribe().

Kaum propagiert
Schulungen zeigten nur manuells subscribe()
Viele Best-Practises kamen erst Jahre später, erst mit stärkerer Nutzung von RxJS
