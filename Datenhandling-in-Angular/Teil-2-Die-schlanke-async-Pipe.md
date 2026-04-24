# Teil 2: Die schlanke async-Pipe – Daten anzeigen ohne Subscribe

Gerade in wachsenden Komponenten wird schnell sichtbar, dass der manuelle Umgang mit Subscriptions nicht immer ein guter Skalierungsweg darstellt.

Im ersten Teil dieser Serie haben wir uns das Ur-Pattern für REST-Calls in Angular angesehen: den manuellen `subscribe()`.

Wir haben die Hintergründe beleuchtet, anhand von Code-Beispielen gezeigt, wie dieser Ansatz aussieht, und erklärt, warum er über viele Jahre hinweg so häufig eingesetzt wurde.

In diesem Beitrag betrachten wir nun die Kehrseite des manuellen Subscribe-Patterns: die wachsende Komplexität, die dadurch in Komponenten entstehen kann. Außerdem zeigen wir eine alternative Herangehensweise mit der `async`-Pipe, die dabei helfen kann, diese Komplexität zu reduzieren – und wie dadurch übersichtlicher und wartbarer Angular-Code entstehen kann.

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
````

## Mögliche Nachteile und Problematiken

* **Manueller Umgang mit Subscriptions**
  Die Komponente ist selbst dafür verantwortlich, Subscriptions zu verwalten und korrekt aufzuräumen.
  Bei HTTP-Requests ist das Risiko überschaubar, da diese nach der Antwort automatisch `complete()` werden.
  Bei **langlebigen Observables** (z. B. `interval()`, WebSockets, `valueChanges`, Router-Events) kann ein fehlender oder fehlerhafter Cleanup jedoch dazu führen, dass Callbacks weiterhin ausgeführt werden, obwohl die Komponente bereits zerstört wurde.

* **Vertikales Wachstum der Komponente**
  Mit jeder weiteren Subscription wächst die Komponente um zusätzlichen Boilerplate-Code.
  Das kann die Lesbarkeit beeinträchtigen, insbesondere wenn viele parallele Datenströme verarbeitet werden.

* **Zustandsverwaltung auf Komponentenebene**
  Der State wird als veränderbare Variable gehalten und im `subscribe()` aktualisiert.
  Das ist ein gängiges und funktionierendes Muster, kann aber mit zunehmender Komplexität schwieriger nachzuvollziehen sein.

* **Imperativer statt deklarativer Ansatz**
  Die Komponente beschreibt stärker *wie* Daten verarbeitet werden, anstatt nur *welche* Daten benötigt werden.
  Das kann je nach Kontext mehr Steuerlogik erfordern.

* **Unterbrechung des reaktiven Datenflusses**
  Durch das Zwischenspeichern in lokalen Variablen wird der Stream nicht direkt im Template genutzt.
  Das ist nicht grundsätzlich falsch, aber ein anderer Stil als rein reaktive Ansätze.

## Was ist die async-Pipe?

Die `async`-Pipe ist eine integrierte Angular-Pipe und Teil des `CommonModule`. Sie wurde dafür entwickelt, asynchrone Datenquellen wie Observables oder Promises direkt im Template zu verarbeiten.

Durch den Einsatz der `async`-Pipe kann in vielen Fällen auf ein manuelles `subscribe()` in der Komponente verzichtet werden. Angular übernimmt das Abonnieren und auch das Aufräumen automatisch, sobald die Template-Bindung entsteht bzw. wieder entfernt wird.

Dadurch reduziert sich oft der notwendige Boilerplate-Code. Komponenten können schlanker werden, da Daten nicht mehr zwingend im `subscribe()` zwischengespeichert werden müssen. Der Code folgt stärker einem deklarativen Ansatz: Die Komponente beschreibt, *welche* Daten benötigt werden, während das Template definiert, *wie* diese dargestellt werden.

> :point_up: Die `async`-Pipe ist besonders hilfreich für einfache Datenströme.
> In komplexeren Szenarien (z. B. mehrfach genutzte Streams oder aufwendige Kombinationen) kann zusätzlicher Aufwand entstehen, etwa durch mehrfaches Subscriben oder notwendiges Sharing.

## Historische Einordnung

Die `async`-Pipe existiert bereits seit Angular 2, wurde jedoch nicht in allen Projekten konsequent eingesetzt.

In den Anfangsjahren lag der Fokus häufig auf einem schnellen Einstieg, weshalb manuelle `subscribe()`-Aufrufe oft bevorzugt wurden, da sie leicht nachvollziehbar sind.

Erst mit der stärkeren Nutzung von RxJS und wachsender Projektgröße gewannen deklarative Patterns – und damit auch die `async`-Pipe – zunehmend an Bedeutung.

In bestehenden Codebasen finden sich daher häufig beide Ansätze nebeneinander.

Das Beispiel aus Code 1 wird nun umgeschrieben, um eine mögliche Nutzung der `async`-Pipe zu zeigen.

**Code 2: Von `subscribe()` zur async-Pipe**

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

Die Komponente stellt ein `Observable` bereit, das im Konstruktor initialisiert wird.
Dieses Vorgehen ist hier unproblematisch, da kein manueller `subscribe()` erfolgt.

Durch `products$ | async` im Template wird das Observable automatisch abonniert.
Die Ausführung der Anfrage erfolgt erst an dieser Stelle.

Mit `as products` wird ein Alias definiert, um den Wert im Template weiterzuverwenden.

Der Loading- und Fallback-Zustand wird weiterhin im Template abgebildet.

> :point_up: Die Verantwortung für das Subscription-Handling wird hier ins Template verlagert.
> Das reduziert Code in der Komponente, bedeutet aber auch, dass ein Teil des Datenflusses weniger explizit sichtbar ist.

## Moderne Angular-Schreibweise: Weniger Boilerplate, mehr Klarheit

### Functional Inject und Built-in Control Flow

Mit Angular 14 wurde mit `inject()` eine Alternative zur Constructor-Injection eingeführt.
Damit können Abhängigkeiten direkt als Klassenfelder definiert werden.

Seit Angular 17 stehen zusätzlich neue Template-Kontrollstrukturen wie `@if` und `@for` zur Verfügung, die eine klarere und kompaktere Schreibweise ermöglichen.

**Code 3: Functional Inject & Built-in Control Flow**

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

### Was hier passiert – und warum das sinnvoll sein kann

#### 1. Field-Injection statt Constructor-Injection

Mit `inject(ProductService)` verwenden wir Field-Injection.
Ein möglicher Vorteil ist, dass `products$` direkt bei der Deklaration initialisiert werden kann.

Das spart etwas Boilerplate und kann die Lesbarkeit erhöhen.
Gleichzeitig ist dies vor allem eine Frage des Stils – Constructor-Injection bleibt weiterhin eine etablierte und valide Alternative.

#### 2. Deklarative Templates mit Built-in Control Flow

Die neuen Kontrollstrukturen können die Lesbarkeit verbessern und reduzieren zusätzliche Template-Konstrukte.

Allerdings handelt es sich auch hier primär um eine alternative, modernere Schreibweise.
Die bisherigen Structural Directives wie `*ngIf` und `*ngFor` sind weiterhin stabil und werden unterstützt,
auch wenn die neue Control-Flow-Syntax perspektivisch stärker in den Fokus rückt.

## Fazit

* Die **Async Pipe** kann den Umgang mit asynchronen Daten vereinfachen und Boilerplate reduzieren.
  In vielen Fällen entfällt ein manueller Umgang mit Subscriptions.
* Der Code kann dadurch deklarativer werden, da Datenströme direkt im Template genutzt werden.
* Gleichzeitig bringt dieser Ansatz auch Trade-offs mit sich, z. B. weniger explizite Kontrolle oder mögliche Mehrfach-Subscriptions im Template.
* Klassische Muster mit `subscribe()` sind weiterhin gültig und können je nach Anwendungsfall sinnvoll sein.
* Insgesamt handelt es sich bei der `async`-Pipe um eine von mehreren möglichen Herangehensweisen, die je nach Kontext ihre Stärken und Schwächen hat.

## Quellen

* Angular Documentation – AsyncPipe
  [https://angular.dev/api/common/AsyncPipe](https://angular.dev/api/common/AsyncPipe)
* Angular University – Reactive Angular Templates
  [https://blog.angular-university.io/angular-reactive-templates/](https://blog.angular-university.io/angular-reactive-templates/)
* Angular Documentation – Dependency Injection
  [https://angular.dev/guide/di](https://angular.dev/guide/di)
* Angular Documentation (v17) – Built-in Control Flow
  [https://v17.angular.io/guide/control_flow](https://v17.angular.io/guide/control_flow)

