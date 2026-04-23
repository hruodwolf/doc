# Teil 2: Die schlanke async‑Pipe – Daten anzeigen ohne Subscribe

Gerade in wachsenden Komponenten wird schnell sichtbar, dass der manuelle Umgang mit Subscriptions keinen guten Skalierungsweg darstellt.

Im ersten Teil dieser Serie haben wir uns das Ur‑Pattern für REST‑Calls in Angular angesehen: den manuellen `subscribe()`.

Wir haben die Hintergründe beleuchtet, anhand von Code‑Beispielen gezeigt, wie dieser Ansatz aussieht, und erklärt, warum er über viele Jahre hinweg so häufig eingesetzt wurde.

In diesem Beitrag betrachten wir nun die Kehrseite des manuellen Subscribe‑Patterns: die wachsende Komplexität, die dadurch in Komponenten entstehen kann. Außerdem zeigen wir, wie sich diese Komplexität durch den Einsatz der `async`‑Pipe deutlich reduzieren lässt – und wie dadurch übersichtlicher und wartbarer Angular‑Code entsteht.

Im nächsten Schritt analysieren wir den manuellen `subscribe()` und die damit einhergehenden Herausforderungen.

**Code 1: Manueller Subscribe**
```ts
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [CommonModule],
  template: `
    <p *ngIf="loading">Load data...</p>

    <ul *ngIf="!loading && products.length > 0; else noData">
      <li *ngFor="let product of products">
        {{ product.id }} - {{ product.name }}
      </li>
    </ul>

    <ng-template #noData>
      <p>No data found</p>
    </ng-template>
  `,
})
export class App implements OnInit, OnDestroy {
  products: Product[] = [];
  loading = true;
  private destroy$ = new Subject<void>();

  constructor(readonly productService: ProductService) {}

  ngOnInit() {
    this.productService
      .getProducts()
      .pipe(takeUntil(this.destroy$))
      .subscribe((value) => {
        this.products = value;
        this.loading = false;
      });
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## Mögliche Nachteile und Problematiken

- **Manueller Umgang mit Subscriptions**  
  Die Komponente ist selbst dafür verantwortlich, Subscriptions zu verwalten und korrekt aufzuräumen.  
  Bei HTTP‑Requests ist das Risiko überschaubar, da diese nach der Antwort automatisch `complete()` werden.  
  Bei **langlebigen Observables** (z. B. `interval()`, WebSockets, `valueChanges`, Router‑Events) kann ein fehlender oder fehlerhafter Cleanup jedoch dazu führen, dass Callbacks weiterhin ausgeführt werden, obwohl die Komponente bereits zerstört wurde. Dies kann unerwartetes Verhalten, Fehler und schwer nachvollziehbare Bugs verursachen.

- **Vertikales Wachstum der Komponente**  
  Mit jeder weiteren Subscription wächst die Komponente um zusätzlichen Boilerplate‑Code:  
  weitere `loading`‑States, zusätzliche Subscriptions, zusätzliche `takeUntil()`‑Pipes oder eigene Cleanup‑Logik.  
  Die eigentliche Fachlogik rückt dabei zunehmend in den Hintergrund, und der Code wird schwieriger zu lesen und zu warten.

- **Zustandsverwaltung auf Komponentenebene**  
  Der State (z. B. `products`) wird als veränderbare Variable in der Komponente gehalten und direkt im `subscribe()` manipuliert.  
  Im weiteren Entwicklungsverlauf kann dieser Zustand an mehreren Stellen verändert werden, was die Nachvollziehbarkeit erschwert und schnell zu inkonsistentem oder unerwartetem Zustand führt.

- **Imperativer statt deklarativer Ansatz**  
  Die Komponente beschreibt nicht *was* angezeigt werden soll, sondern *wie* der Zustand Schritt für Schritt aufgebaut und geändert wird.  
  Dieser imperative Stil erfordert mehr Steuerlogik, erhöht die Komplexität und erschwert es, den Datenfluss auf einen Blick zu verstehen.

- **Kein reaktiver Stil**  
  Durch das manuelle Ablegen der Daten in lokalen Variablen wird der reaktive Datenfluss unterbrochen.  
  Das Template reagiert nicht direkt auf einen Stream, sondern auf explizit gesetzte Zwischenzustände in der Komponente.

## Was ist die async‑Pipe?

Die `async`‑Pipe ist eine integrierte Angular‑Pipe und Teil des `CommonModule`. Sie wurde speziell dafür entwickelt, asynchrone Datenquellen wie Observables oder Promises direkt im Template zu verarbeiten. Die asynchrone Verarbeitung findet dabei nicht mehr in der Komponente, sondern unmittelbar im Template statt.

Durch den Einsatz der `async`‑Pipe ist ein manueller `subscribe()` in der Komponente nicht mehr notwendig. Angular übernimmt das Abonnieren des Observables automatisch, sobald die Pipe im Template verwendet wird, und kümmert sich ebenso um das Abbestellen, wenn die Template‑Bindung entfernt wird – beispielsweise beim Zerstören der Komponente.

Dadurch reduziert sich der notwendige Boilerplate‑Code erheblich. Komponenten enthalten weniger Zustandsvariablen, da Daten nicht mehr explizit im `subscribe()` zwischengespeichert werden müssen. Der Code wird übersichtlicher, leichter lesbar und folgt einem stärker deklarativen Ansatz: Die Komponente beschreibt, *welche* Daten benötigt werden, während das Template definiert, *wie* diese dargestellt werden.

Insgesamt verlagert die `async`‑Pipe die Verantwortung für das Subscription‑Management aus der Komponente heraus und ermöglicht so klar strukturierte, wartbare Angular‑Anwendungen mit einem konsistenteren reaktiven Datenfluss.

Gerne – ich integriere deine Ergänzungen als **flüssigen Text**, korrigiere Sprache und Stil und halte die Aussage bewusst historisch‑einordnend, nicht wertend.

## Historische Einordnung

Die `async`‑Pipe existiert bereits seit Angular 2, wurde in vielen Projekten jedoch über einen langen Zeitraum hinweg kaum konsequent eingesetzt. In den Anfangsjahren von Angular lag der Fokus häufig auf einem schnellen Einstieg und auf grundlegendem Verständnis von Komponenten, Services und HTTP‑Requests. Entsprechend wurden in Schulungen und Tutorials meist manuelle `subscribe()`‑Aufrufe gezeigt, da sie leicht nachvollziehbar und gut sichtbar waren.

Zudem wurde die `async`‑Pipe lange Zeit nur wenig aktiv propagiert. Viele Best Practices rund um reaktive Programmierung, saubere Trennung von Zuständigkeiten und deklarative Template‑Strukturen haben sich erst im Laufe der Jahre etabliert. Erst mit der stärkeren und breiteren Nutzung von RxJS im Projektalltag rückte auch die `async`‑Pipe zunehmend in den Fokus.

In bestehenden Codebasen spiegelte sich diese Entwicklung häufig wider: Selbst wenn die `async`‑Pipe bereits verfügbar war, blieb der manuelle `subscribe()` lange der dominierende Ansatz. Die konsequente Nutzung deklarativer Patterns setzte sich oft erst deutlich später durch – meist dann, wenn Anwendungen größer wurden und die Nachteile manueller Subscription‑Verwaltung immer deutlicher zutage traten.






und jetzt kommt die async-Pipe ins spiel

Verbesserter Code mit async-Pipe

Artikel 2: Deklarativ & Lifecycle‑Management durch Angular

historische Einordnung
Die async‑Pipe existiert bereits seit Angular 2, wurde in vielen Projekten jedoch lange Zeit kaum konsequent eingesetzt.
Kaum propagiert
Schulungen zeigten nur manuells subscribe()
Viele Best-Practises kamen erst Jahre später, erst mit stärkerer Nutzung von RxJS

Empfehlungen & nächster Schritt
Was du sehr gut triffst:

	• realistische Projektperspektive
	• differenzierte Aussage („nicht falsch, aber …“)
Kleiner Verbesserungsvorschlag für Artikel 2:
	• Kontrast klar machen:
		• Artikel 1: Imperativ & Lifecycle‑Verantwortung beim Entwickler






