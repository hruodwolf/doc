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

## Historische Einordnung

Die `async`‑Pipe existiert bereits seit Angular 2, wurde in vielen Projekten jedoch über einen langen Zeitraum hinweg kaum konsequent eingesetzt. In den Anfangsjahren von Angular lag der Fokus häufig auf einem schnellen Einstieg und auf grundlegendem Verständnis von Komponenten, Services und HTTP‑Requests. Entsprechend wurden in Schulungen und Tutorials meist manuelle `subscribe()`‑Aufrufe gezeigt, da sie leicht nachvollziehbar und gut sichtbar waren.

Zudem wurde die `async`‑Pipe lange Zeit nur wenig aktiv propagiert. Viele Best Practices rund um reaktive Programmierung, saubere Trennung von Zuständigkeiten und deklarative Template‑Strukturen haben sich erst im Laufe der Jahre etabliert. Erst mit der stärkeren und breiteren Nutzung von RxJS im Projektalltag rückte auch die `async`‑Pipe zunehmend in den Fokus.

In bestehenden Codebasen spiegelte sich diese Entwicklung häufig wider: Selbst wenn die `async`‑Pipe bereits verfügbar war, blieb der manuelle `subscribe()` lange der dominierende Ansatz. Die konsequente Nutzung deklarativer Patterns setzte sich oft erst deutlich später durch – meist dann, wenn Anwendungen größer wurden und die Nachteile manueller Subscription‑Verwaltung immer deutlicher zutage traten.

Das Beispiel aus Code 1 wird nun umgeschrieben, um den Einsatz der `async`‑Pipe zu zeigen.

**Code 2: Von `subscribe()` zur async‑Pipe**

```ts
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [CommonModule],
  template: `
    <ng-container *ngIf="products$ | async as products; else loading">
      <ul *ngIf="products.length > 0; else noData">
        <li *ngFor="let product of products">
          {{ product.id }} - {{ product.name }}
        </li>
      </ul>
    </ng-container>

    <ng-template #loading>
      <p>Load data...</p>
    </ng-template>

    <ng-template #noData>
      <p>No data found</p>
    </ng-template>
  `,
})
export class App {
  products$: Observable<Product[]>;

  constructor(readonly productService: ProductService) {
    this.products$ = this.productService.getProducts();
  }
}
```
## Erklärung des Codes

Die Komponente benötigt keine Zwischenvariable für den Zustand der Daten mehr, sondern stellt ausschließlich ein `Observable` bereit. Dieses Observable wird im Konstruktor initialisiert, was in diesem Fall völlig valide ist, da kein manueller `subscribe()` mehr notwendig ist und somit auch keine Seiteneffekte ausgelöst werden.

Durch den Einsatz von `products$ | async` im Template erfolgt das Abonnieren des Observables implizit über die `async`‑Pipe. Erst an dieser Stelle wird die Anfrage tatsächlich ausgeführt. Die Komponente selbst definiert lediglich den Datenstrom, ohne dessen Lifecycle manuell zu steuern.

Mit `as products` wird ein Alias für die durch die `async`‑Pipe gelieferten Daten definiert. Dadurch können die Werte im weiteren Template komfortabel weiterverwendet werden, ohne erneut auf die Pipe zugreifen zu müssen.

Da es sich um eine asynchrone Verarbeitung handelt, wird über `else loading` zunächst das entsprechende Loading‑Template gerendert, solange noch keine Daten verfügbar sind. Sobald die Anfrage abgeschlossen ist, wird überprüft, ob Daten vorhanden sind. Ist die Anfrage erfolgreich, liefert jedoch keine Einträge zurück, wird stattdessen über `else noData` das entsprechende Template angezeigt.

> :point_up: Mit der `async`‑Pipe beschreibt die Komponente nur noch den benötigten Datenstrom.
Das Abonnieren, Aktualisieren und Aufräumen übernimmt Angular automatisch im Template.

## Moderne Angular‑Schreibweise: Weniger Boilerplate, mehr Klarheit

Schauen wir uns einige Stellen an, an denen sich Angular in den letzten Versionen deutlich weiterentwickelt hat – und wie wir diese Entwicklungen nutzen können, um Code **klarer, deklarativer und wartbarer** zu schreiben.

### Functional Inject und Built‑in Control Flow

Mit **Angular 14 (2022)** wurde mit **Functional Inject (`inject()`)** eine moderne Alternative zur klassischen Constructor‑Injection eingeführt. Diese erlaubt es, Abhängigkeiten direkt als Klassenfelder zu deklarieren – ohne zusätzlichen Konstruktor‑Boilerplate.

Ein weiterer großer Schritt folgte mit **Angular 17 (2023)**: die Einführung von **Built‑in Control Flow**. Damit wurden die bisherigen **Structural Directives** (`*ngIf`, `*ngFor`, `*ngSwitch`) durch neue, fest in die Template‑Sprache integrierte Konstrukte wie `@if` und `@for` ergänzt.  
Das Ergebnis: weniger `ng-template`, weniger Microsyntax, bessere Lesbarkeit.

**Code 3: Functional Inject & Built‑in Control Flow in der Praxis**
```ts
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [CommonModule],
  template: `
  @if (products$ | async; as products) {
    @if (products.length > 0) {
      <ul>
        @for (product of products; track product.id) {
          <li>
            {{ product.id }} - {{ product.name }}
          </li>
        }
      </ul>
    } @else {
      <p>No data found</p>
    }
  } @else {
    <p>Load data...</p>
  }
  `,
})
export class App {
  productService = inject(ProductService);
  products$: Observable<Product[]> = this.productService.getProducts();
}
```

### Was hier passiert – und warum das sinnvoll ist

#### 1. Field‑Injection statt Constructor‑Injection

Mit `inject(ProductService)` verwenden wir **Field‑Injection**. Der entscheidende Vorteil:  
Die Observable‑Variable `products$` kann **direkt bei der Deklaration** über den Service initialisiert werden.

Das ist mit klassischer Constructor‑Injection nicht möglich, da der Konstruktor erst nach der Feldinitialisierung ausgeführt wird.  
Das Ergebnis ist weniger Code, weniger Umwege – und eine klarere Intention.

#### 2. Deklarative Templates mit Built‑in Control Flow

Im Template nutzen wir `@if` und `@for` statt `*ngIf`, `*ngFor` und `ng-template`.  
Der Kontrollfluss liest sich dadurch näher an JavaScript und beschreibt **klar und linear**, was dargestellt wird:

*   Daten vorhanden?
*   Liste nicht leer?
*   Fallback bei leerem Ergebnis
*   Loading‑Zustand

All das ohne zusätzliche Template‑Referenzen oder Hilfskonstrukte.

## Fazit

*   Die **Async Pipe** vereinfacht den Umgang mit asynchronen Daten erheblich.  
    Manuelles `subscribe()` sowie explizites `unsubscribe()` entfallen – Angular kümmert sich automatisch um das Subscription‑Lifecycle‑Management.
*   Der Code wird insgesamt **deutlich deklarativer**:  
    Komponente und Template beschreiben *was* dargestellt wird, nicht *wie* der Zustand intern verwaltet wird.
*   Klassische Zwischenvariablen und explizite Loading‑State‑Flags werden überflüssig.
*   Das Ergebnis ist **besser lesbarer, schlankerer und wartbarer Code**, der näher an der fachlichen Intention bleibt.

## Quellen

*   Angular Documentation – AsyncPipe
    *   <https://angular.dev/api/common/AsyncPipe>
*   Angular University – Reactive Angular Templates
    *   <https://blog.angular-university.io/angular-reactive-templates/>
*   Angular Documentation – Dependency Injection
    *   <https://angular.dev/guide/di>
*   Angular Documentation (v17) – Built‑in Control Flow
    *   <https://v17.angular.io/guide/control_flow>
