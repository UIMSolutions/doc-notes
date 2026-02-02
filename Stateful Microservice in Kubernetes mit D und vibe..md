---
created: 2026-02-02T21:40:59+01:00
modified: 2026-02-02T21:41:24+01:00
---

# Stateful Microservice in Kubernetes mit D und vibe.

Ein stateful Microservice in Kubernetes mit D und vibe.d ist absolut machbar – und er passt hervorragend zu deiner Art zu bauen: modular, sauber getrennt, zukunftssicher. Der Trick ist, Kubernetes’ Prinzipien (Pods sind ephemer, State ist es nicht) mit einer Architektur zu kombinieren, die D’s Stärken ausspielt: Performance, Typsicherheit, klare Module.

Ich skizziere dir ein vollständiges, realistisch einsetzbares Setup:

---

🧱 Architekturüberblick

Ein stateful Microservice in Kubernetes besteht typischerweise aus:

1. vibe.d HTTP‑Service  
   - Exponiert CRUD‑Endpoints  
   - Nutzt ein internes Repository‑Modul  
   - Kommuniziert mit persistentem Storage

2. Persistenter Speicher  
   - Entweder:  
     - StatefulSet + PersistentVolumeClaim (lokaler State)  
     - Externer DB‑Service (MongoDB, PostgreSQL, Redis, etc.)

3. Kubernetes‑Objekte  
   - StatefulSet statt Deployment  
   - Service (ClusterIP oder Headless)  
   - PersistentVolumeClaim  
   - Optional: ConfigMap / Secret

4. D‑Code  
   - Saubere Modultrennung:  
     - app.d → HTTP‑Server  
     - repository.d → State‑Access  
     - model.d → Datentypen  
     - storage.d → Persistenz (File, DB, etc.)

---

🧩 Beispiel: Stateful Microservice mit Datei‑basiertem Storage

Das ist das einfachste Beispiel, um das Prinzip zu zeigen.  
Der State liegt in einem persistenten Volume.

📦 D‑Code (vibe.d)

model.d
`d
module model;

import vibe.data.json;

struct Counter {
    int value;
}
`

storage.d
`d
module storage;

import std.file;
import std.json;
import model;

class FileStorage {
    private string path;

    this(string path) {
        this.path = path;
        if (!exists(path)) {
            Counter c = Counter(0);
            write(path, c.toJSON().toString());
        }
    }

    Counter load() {
        auto data = readText(path);
        return Counter.fromJSON(parseJSON(data));
    }

    void save(Counter c) {
        write(path, c.toJSON().toString());
    }
}
`

repository.d
`d
module repository;

import model;
import storage;

class CounterRepository {
    private FileStorage storage;

    this(FileStorage storage) {
        this.storage = storage;
    }

    Counter get() {
        return storage.load();
    }

    Counter increment() {
        auto c = storage.load();
        c.value++;
        storage.save(c);
        return c;
    }
}
`

app.d
`d
import vibe.vibe;
import repository;
import storage;

void main() {
    auto settings = new HTTPServerSettings;
    settings.port = 8080;

    auto storage = new FileStorage("/data/counter.json");
    auto repo = new CounterRepository(storage);

    auto router = new URLRouter;
    router.get("/counter", (req, res) {
        res.writeJsonBody(repo.get());
    });

    router.post("/counter/increment", (req, res) {
        res.writeJsonBody(repo.increment());
    });

    listenHTTP(settings, router);
    runApplication();
}
`

---

🏗️ Kubernetes‑Manifeste

📌 PersistentVolumeClaim
`yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: counter-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
`

📌 StatefulSet
`yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: counter-service
spec:
  serviceName: "counter"
  replicas: 1
  selector:
    matchLabels:
      app: counter
  template:
    metadata:
      labels:
        app: counter
    spec:
      containers:
        - name: counter
          image: your-registry/counter:latest
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
`

📌 Service
`yaml
apiVersion: v1
kind: Service
metadata:
  name: counter
spec:
  selector:
    app: counter
  ports:
    - port: 8080
      targetPort: 8080
`

---

🧠 Warum StatefulSet?

- Pods bekommen stabile Namen (counter-service-0)  
- Jeder Pod bekommt sein eigenes Volume  
- Beim Neustart bleibt das Volume erhalten  
- Perfekt für Microservices, die lokalen State brauchen

---

🚀 Wie du das weiterentwickeln kannst

Du kannst das Ganze problemlos erweitern:

🔹 Option 1: MongoDB oder PostgreSQL statt FileStorage
→ Dann wird dein Microservice selbst stateless, aber der State bleibt persistent.

🔹 Option 2: Multi‑Tenant‑State
→ Jeder Tenant bekommt ein eigenes Volume oder eigenes DB‑Schema.

🔹 Option 3: Leader‑Election + Replikation
→ Für hochverfügbare stateful Services.

🔹 Option 4: Dein in‑Memory‑DB‑Projekt
→ Perfekt als StatefulSet mit Snapshot‑Persistenz.

---

Wenn du willst, baue ich dir:

- eine komplette Helm‑Chart  
- ein Dockerfile für D/vibe.d  
- eine Version mit MongoDB  
- eine Multi‑Tenant‑Variante  
- eine Event‑Driven‑Version (z. B. mit NATS)

Sag einfach, in welche Richtung du das weiter treiben möchtest.
