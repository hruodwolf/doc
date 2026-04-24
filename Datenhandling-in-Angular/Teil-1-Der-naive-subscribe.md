# Teil 1 – Der manuelle Subscribe – der Klassiker

Schauen wir uns zuerst den **manuellen `subscribe()`** an.  
Dieses Pattern wird häufig in Schulungen als einfache Demonstration für einen REST-Call gezeigt – und findet sich auch in vielen produktiven Angular-Anwendungen wieder.

Aus heutiger Sicht ist der manuelle `subscribe()` für eine **reine Datenanzeige** eine mögliche Herangehensweise unter mehreren. Die Gründe für seinen Einsatz sind vielfältig:

*   **Einsteiger** nutzen dieses Pattern häufig, weil es leicht verständlich ist:  
    Alles passiert an einer Stelle, der Datenfluss ist direkt sichtbar.
*   **Fortgeschrittene Entwickler** greifen darauf zurück, weil es schnell geschrieben ist und für viele einfache Anwendungsfälle ausreicht.
*   **Zeitdruck** spielt – wie so oft – ebenfalls eine Rolle.

Für Entwickler, die eher mit deklarativen Ansätzen wie der `async`-Pipe oder Signals arbeiten, wirkt dieses Pattern teilweise ungewohnter, da das `subscribe()` dort meist nicht explizit sichtbar ist, sondern indirekt gehandhabt wird.

> Das folgende Beispiel ist bewusst vereinfacht – ohne Logging, Navigation oder Fehlerbehandlung.

---

## Code 1: Manueller Subscribe in der Komponente

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
export class App implements OnInit {
  products: Product[] = [];
  loading = true;

  constructor(readonly http: HttpClient) {}

  ngOnInit() {
    this.http
      .get<Product[]>('/api/products')
      .subscribe((value) => {
        this.products = value;
        this.loading = false;
      });
  }
}
````

---

## Erläuterung zu Code 1

### Datenanfrage in der Komponente

```ts
this.http.get<Product[]>('/api/products')
```

Diese Zeile liefert **ein Observable** zurück.
Zu diesem Zeitpunkt werden **noch keine Daten geladen** und **keine HTTP-Anfrage gestartet**.
Das Observable beschreibt lediglich, *was passieren soll*, sobald jemand es abonniert.

Erst mit dem Aufruf von `subscribe()` wird das Observable aktiviert – **jetzt entsteht eine Subscription und die Anfrage wird ausgeführt**.

```ts
.subscribe((value) => {
  this.products = value;
});
```

Im `subscribe()` wird definiert, was passieren soll, **wenn Daten eintreffen**.

In diesem Fall:

* werden die vom Server gelieferten Daten (`value`)
* der lokalen Komponenteneigenschaft `products` zugewiesen

Damit stehen die Daten der Komponente zur Verfügung.

> :point_up: Die Anfrage wird beim `subscribe()` ausgelöst, die Daten werden asynchron bereitgestellt.

---

### Darstellung der Daten im Template

Da `products` gesetzt wird, kann Angular die Daten über `*ngFor` darstellen.
Sobald sich der Wert ändert, wird die Anzeige automatisch aktualisiert.

---

### Zusammengefasst

* `http.get()` allein startet **keine Anfrage**
* `subscribe()` aktiviert das Observable
* die Datenverarbeitung erfolgt im Callback
* das Template rendert den aktuellen Zustand

---

## Einordnung des Patterns

Der manuelle `subscribe()` ist ein **imperativer Ansatz**:

* Die Komponente steuert aktiv, *wann* und *wie* Daten verarbeitet werden
* Zustand wird lokal gehalten und verändert

Das ist ein **valider und etablierter Stil**, bringt aber je nach Anwendung unterschiedliche Trade-offs mit sich:

* mehr Kontrolle über den Ablauf
* dafür potenziell mehr Boilerplate
* stärker verteilte Zustandslogik bei wachsender Komplexität

---

## Kleines Refactoring

### Trennung von Datenzugriff und Darstellung

Ein häufiger nächster Schritt ist, den Datenzugriff in einen Service auszulagern.

**Code 2: ProductService**

```ts
@Injectable({
  providedIn: 'root',
})
export class ProductService {
  constructor(readonly http: HttpClient) {}

  getProducts(): Observable<Product[]> {
    return this.http.get<Product[]>('/api/products');
  }
}
```

Das verbessert die Struktur und Wiederverwendbarkeit, unabhängig davon, ob später weiterhin manuell subscribt wird oder nicht.

---

## Muss man Subscriptions aufräumen?

Oft hört man: *„Du musst Subscriptions wegen Memory Leaks aufräumen.“*
Bei `HttpClient`-Requests ist diese Aussage differenziert zu betrachten.

`this.http.get()` liefert ein **cold Observable**:

1. Die Anfrage startet erst bei `subscribe()`
2. Ein `unsubscribe()` kann eine laufende Anfrage abbrechen
3. Nach der Antwort wird das Observable automatisch **completed**

Das bedeutet:

👉 Ein einzelner HTTP-Request führt in der Regel **nicht direkt zu einem Memory Leak**

---

### Wo kann dennoch ein Problem entstehen?

Das Risiko liegt eher im **Callback**:

Wenn eine Komponente während einer laufenden Anfrage zerstört wird, kann der Callback dennoch ausgeführt werden.
Je nach Zugriff auf den Zustand kann das zu unerwartetem Verhalten führen.

Daher ist es oft sinnvoll, Subscriptions bewusst zu beenden – insbesondere bei langlebigen oder mehrfachen Streams.

---

## Elegantes Aufräumen mit `takeUntil()`

Ein gängiges Pattern ist `takeUntil()`:

**Idee:**
Ein Stream läuft, bis ein zweiter Stream ein Stoppsignal sendet.

### Vorteile

* kein manuelles Verwalten einzelner Subscriptions
* gut kombinierbar mit mehreren Streams
* häufig in größeren Komponenten zu finden

---

## Code 3: Variante mit `takeUntil()`

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

> :point_up: Dieses Pattern bietet mehr Kontrolle über den Lifecycle, bringt aber auch zusätzlichen Code mit sich.

---

## Fazit

Ein manueller `subscribe()` beschreibt eine **imperative** Herangehensweise zur Verarbeitung asynchroner Daten in Angular.

Er kann besonders dann sinnvoll sein, wenn:

* explizite Kontrolle über den Ablauf benötigt wird  
* Seiteneffekte gezielt gesteuert werden sollen  

Für reine Datenanzeige kann dieser Ansatz jedoch mehr Boilerplate erzeugen als nötig.  
Deklarative Ansätze (z. B. mit der `async`-Pipe) stellen hier eine mögliche Alternative dar – bringen jedoch ebenfalls eigene Trade-offs mit sich.

👉 Es handelt sich dabei nicht um „richtig“ oder „falsch“, sondern um unterschiedliche Ansätze mit jeweils eigenen Stärken und Einschränkungen.

---

## Ausblick

Im nächsten Teil betrachten wir eine dieser Alternativen:

**Teil 2: Die schlanke async-Pipe – Daten anzeigen ohne Subscribe**

---

## Quellen

* Angular Documentation – Making HTTP requests
  [https://angular.dev/guide/http/making-requests](https://angular.dev/guide/http/making-requests)
* Angular Documentation – HTTP: Request data from a server
  [https://v17.angular.io/guide/http-request-data-from-server](https://v17.angular.io/guide/http-request-data-from-server)
