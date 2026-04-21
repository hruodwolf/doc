# Teil 1 – Der naive Subscribe – ein Klassiker aus der Angular‑Vergangenheit

Schauen wir uns zuerst den **manuellen `subscribe()`** an.  
Oft wird dieses Pattern in Schulungen nur als einfache Demonstration für einen REST-Call gezeigt. In der Praxis begegnet es einem jedoch erstaunlich häufig in produktiven Angular-Anwendungen.

Aus heutiger Sicht gehört der manuelle `subscribe()` für eine **reine Datenanzeige** eher zu den älteren Praktiken. Die Gründe für seinen Einsatz sind vielfältig:

*   **Einsteiger** nutzten dieses Pattern häufig, weil es leicht verständlich ist:  
    Alles passiert an einer Stelle, der Datenfluss ist direkt sichtbar.
*   **Fortgeschrittene Entwickler** griffen darauf zurück, weil es schnell geschrieben ist, für einfache Anwendungsfälle ausreicht und in Projekten meist akzeptiert wurde.
*   **Zeitdruck** spielte – wie so oft – ebenfalls eine Rolle.

Für Entwickler, die heute eher mit neueren Konzepten wie der `async`-Pipe oder Signals arbeiten, stellt dieses Pattern eine zusätzliche Hürde dar. Das `subscribe()` findet in modernen Ansätzen meist „unter der Haube“ statt und ist nicht mehr explizit sichtbar.

> Das folgende Beispiel ist bewusst stark vereinfacht – ohne Logging, Navigation oder Fehlerbehandlung.

**Code 1: Manueller Subscribe in der Komponente**

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
```

## Erläuterung zu Code 1
### Datenanfrage in der Komponente
```ts
this.http.get<Product[]>('/api/products')
```

Diese Zeile liefert **ein Observable** zurück.\
Zu diesem Zeitpunkt werden **noch keine Daten geladen** und **keine HTTP‑Anfrage gestartet**.\
Das Observable beschreibt lediglich, *was passieren soll*, sobald jemand es abonniert.

Erst mit dem Aufruf von `subscribe()` wird das Observable tatsächlich aktiviert – **jetzt entsteht eine Subscription und die Anfrage wird an den Server gesendet**.

```ts
.subscribe((value) => {
  this.products = value;
});
```

Im `subscribe()` wird die **Callback‑Funktion** definiert, also das Verhalten, das ausgeführt wird, **wenn der Server die Anfrage erfolgreich beantwortet**.

In diesem Fall:

*   werden die vom Server gelieferten Daten (`value`)
*   der lokalen Komponenteneigenschaft `products` zugewiesen.

Dadurch stehen die Daten der Komponente zur Verfügung.

### Darstellung der Daten im Template

Da `products` nun initialisiert ist, kann Angular die Daten über `*ngFor` iterieren und im Template darstellen.\
Sobald sich der Wert von `products` ändert, aktualisiert Angular automatisch die Anzeige.



**Zusammengefasst**:

*   `http.get()` allein startet **keine Anfrage**
*   `subscribe()` aktiviert das Observable und definiert den Callback
*   die Logik zur Datenverarbeitung liegt vollständig im `subscribe()`
*   das Template rendert den aktuellen Zustand der Komponentendaten

## Kleines Refactoring
### Trennung von Datenzugriff und Darstellung
Der erste sinnvolle Schritt zur Verbesserung ist, den `http.get()`-Aufruf in eine **injectable, wiederverwendbare Service-Klasse** auszulagern. Dadurch trennen wir Datenzugriff und Darstellung und erhalten eine wartbarere Struktur.

**Code 2: Ausgelagerter Datenzugriff im ProductService**
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

### Muss man Subscriptions aufräumen?

Oft hört man: *„Du musst Subscriptions wegen Memory Leaks aufräumen.“*  
Bei `HttpClient`-Requests ist diese Aussage jedoch nur teilweise richtig.

`this.http.get()` liefert ein **cold Observable** mit folgenden Eigenschaften:

1.  Die Anfrage startet erst, wenn jemand `subscribe()` aufruft.
2.  Ein `unsubscribe()` bricht eine laufende Anfrage ab.
3.  Sobald der Server antwortet, wird das Observable automatisch **completed**.

Aus Punkt 3 folgt:  
**Ein `HttpClient`-Request verursacht in der Regel kein Memory Leak**, auch wenn man ihn nicht explizit abbestellt.

#### Wo liegt dann das Problem?

Das Risiko liegt **nicht im Observable**, sondern **im Subscribe-Callback**.

Wird während einer laufenden Anfrage z. B. auf eine andere Seite navigiert und die Komponente zerstört, kann der Callback dennoch ausgeführt werden. Greift dieser dann auf eine bereits zerstörte Komponente zu, kann das zu Fehlern oder unerwartetem Verhalten führen.

Deshalb gilt die Empfehlung:

> **Subscriptions sollten immer bereinigt werden, wenn eine Komponente zerstört wird – auch bei HTTP-Requests.**


### Elegantes Aufräumen mit `takeUntil()`

Es gibt verschiedene Möglichkeiten, Subscriptions sauber zu beenden.  
Ein besonders bewährtes Pattern ist der Einsatz von `takeUntil()`.

**Idee:**  
Ein Observable läuft so lange, bis ein anderes Observable signalisiert: *„Stopp!“*

#### Vorteile von `takeUntil()`:

*   Kein manuelles Tracken einzelner `Subscription`-Objekte
*   Funktioniert mit mehreren Streams gleichzeitig
*   Sehr gut skalierbar in komplexeren Komponenten

**Code 3: Verbesserte Version des Manuellen Subscribes**

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



## Fazit

Ein manueller `subscribe()` ist **nicht per se falsch**. Entscheidend ist der **bewusste Einsatz**.

Für einfache REST-Calls, bei denen Daten ausschließlich angezeigt werden, ist dieses Pattern heute jedoch meist unnötig. Moderne Angular-Anwendungen profitieren deutlich von deklarativen Ansätzen, bei denen Subscription-Logik automatisch gehandhabt wird.

Genau darum geht es im nächsten Artikel dieser Serie:

**Teil 2: Die schlanke async-Pipe – Daten anzeigen ohne Subscribe**