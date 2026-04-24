# Teil 3 – async/await: der lineare Pragmatiker
Kurz gesagt: **Ja – `async/await` ist auch heute ein valider Ansatz**, aber **nicht immer die beste Wahl im Angular-Kontext**. Es ist eher eine **Alternative mit anderen Trade-offs**.

---

## 🧠 Einordnung in Angular

Angulars HTTP-Client basiert auf **Observables (RxJS)**.
Das bedeutet:

* Der „native“ Ansatz in Angular ist **reaktiv (Observable-basiert)**
* `async/await` funktioniert über **Promises**

👉 Du bewegst dich also zwischen zwei Paradigmen:

* **Observable → Stream / reaktiv**
* **Promise → einmaliger Wert / imperativ**

---

## ✅ Wann `async/await` gut passt

`async/await` ist völlig sinnvoll, wenn:

* du **einen einzelnen HTTP-Call** hast
* du das Ergebnis **einmal brauchst**
* du **keine weitere Stream-Verarbeitung** brauchst

### Beispiel

```ts
async ngOnInit() {
  this.products = await firstValueFrom(
    this.productService.getProducts()
  );
}
```

👉 Vorteile:

* sehr lesbar
* linearer Codefluss
* leicht verständlich (auch ohne RxJS-Wissen)

---

## ⚠️ Einschränkungen / Trade-offs

### 1. Verlust von Reaktivität

* kein Stream mehr
* keine automatische Updates bei neuen Werten
* keine einfache Kombination mehr (`combineLatest`, `switchMap`, etc.)

---

### 2. Kein Lazy Verhalten mehr

Observables:

* starten bei `subscribe()`

Promises (`await`):

* laufen sofort los

👉 subtiler, aber wichtiger Unterschied

---

### 3. Kein eingebautes Cancellation-Konzept

* Observables → `unsubscribe()`
* Promises → ❌ kein echtes Canceln

👉 bei HTTP oft egal, aber nicht immer

---

### 4. Schlechter skalierbar bei mehreren Requests

Beispiel:

```ts
const a = await getA();
const b = await getB();
```

vs.

```ts
combineLatest([a$, b$])
```

👉 RxJS ist hier deutlich mächtiger

---

## 🆚 Vergleich der Ansätze

| Ansatz        | Stärke                 | Schwäche          |
| ------------- | ---------------------- | ----------------- |
| `subscribe()` | volle Kontrolle        | viel Boilerplate  |
| `async` pipe  | deklarativ, wenig Code | weniger Kontrolle |
| `async/await` | einfach, linear        | nicht reaktiv     |

---

## 🎯 Fazit

* ✅ `async/await` ist **kein Anti-Pattern**
* ✅ gut für **einfache, einmalige Requests**
* ⚠️ weniger geeignet für:

  * komplexe Datenflüsse
  * UI-getriebene Reaktivität
  * Kombination mehrerer Streams

👉 Es ist also eine **bewusste Entscheidung**, kein „besser oder schlechter“.

---

## 💡 Gute Formulierung für deine Artikel

Wenn du das einbauen willst:

> Neben Observables kann auch `async/await` für HTTP-Requests verwendet werden.
> Dieser Ansatz ist besonders für einfache, einmalige Datenabfragen gut geeignet, verzichtet jedoch auf die reaktiven Möglichkeiten von RxJS und ist daher nicht in allen Szenarien gleich gut skalierbar.

---

Wenn du willst, kann ich dir noch eine **Mini-Sektion für Teil 1 oder 2 schreiben**, die `async/await` sauber einordnet – das rundet die Serie fachlich stark ab.
