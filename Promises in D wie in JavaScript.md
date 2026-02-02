---
created: 2026-02-02T21:33:05+01:00
modified: 2026-02-02T21:34:33+01:00
---

# Promises in D wie in JavaScript

In D gibt es kein eingebautes Promise‑System wie in JavaScript, aber du kannst es auf zwei Wegen lösen:

1. Ein fertiges Promise‑Package nutzen → dpromise  
2. Mit std.concurrency oder std.async eigene Promise‑ähnliche Strukturen bauen

Ich zeige dir beide Wege — erst den schnellen, dann den systematischen.

---

1️⃣ Fertige Lösung: dpromise (ES2015‑like Promises)

Es gibt ein kleines, aber funktionierendes Package namens dpromise, das explizit „ES2015‑like Promises for D“ implementiert.

Installation (dub)
`json
"dependencies": {
    "dpromise": "~>0.0.1"
}
`

Beispiel wie in JavaScript
`d
import dpromise;

void main() {
    auto p = new Promise!int((resolve, reject) {
        // async work simulieren
        resolve(42);
    });

    p.then!((value) {
        writeln("Result: ", value);
    }).catch!((err) {
        writeln("Error: ", err);
    });
}
`

Das fühlt sich schon sehr nach JS an.

---

2️⃣ Eigene Promises bauen (D‑idiomatisch)

Wenn du lieber vollen Kontrolle willst (und ich weiß, das liegt dir), kannst du Promises mit:

- std.concurrency (Message Passing)
- std.parallelism
- std.async (Phobos)

selbst modellieren.

Minimaler Promise‑Typ in D
Hier ein kleines, aber funktionales Promise‑Gerüst:

`d
import core.thread;
import std.concurrency;
import std.stdio;

struct Promise(T) {
    Tid worker;
    this(T delegate() work) {
        worker = spawn({
            auto result = work();
            ownerTid.send(result);
        });
    }

    void then(void delegate(T) onSuccess) {
        receive((T value) {
            onSuccess(value);
        });
    }
}

void main() {
    auto p = Promise!int({
        Thread.sleep(1.seconds);
        return 42;
    });

    p.then((value) {
        writeln("Result: ", value);
    });
}
`

Das ist natürlich nur ein Startpunkt — du kannst:

- Fehlerbehandlung ergänzen  
- catch und finally hinzufügen  
- Chaining implementieren  
- mehrere Promises kombinieren (all, race, …)

Wenn du willst, bauen wir das gemeinsam zu einem vollwertigen Promise‑Framework in D, das JS‑Promises 1:1 nachbildet.

---

3️⃣ Wann sollte man Promises in D überhaupt nutzen?

D ist traditionell eher:

- futures (std.concurrency)
- tasks (std.parallelism)
- async I/O (vibe.d, std.async)

Promises sind sinnvoll, wenn du:

- eine JS‑ähnliche API willst  
- asynchrone Pipelines baust  
- ein DSL oder Framework entwickelst  
- D in Web‑ähnlichen Umgebungen nutzt (z. B. WASM)

---

Wenn du willst, bauen wir jetzt:

🔧 Ein vollständiges Promise‑System in D
mit:

- Promise.then
- Promise.catch
- Promise.finally
- Promise.all
- Promise.race
- Promise.resolve
- Promise.reject

Sag mir einfach, welchen Stil du bevorzugst:

A) JS‑kompatibel  
B) D‑idiomatisch  
C) Hybrid (JS‑API, aber D‑Performance)
