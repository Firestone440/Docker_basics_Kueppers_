# Netzwerke mit Docker Compose

Diese Datei beantwortet die Fragen aus Teil 1 der Detailanleitung auf Basis des Artikels
[Networking in Docker Compose](https://docs.docker.com/compose/how-tos/networking/) aus der offiziellen Docker-Dokumentation.

## 1. Standardnetzwerk von Docker Compose

Wenn in der `compose.yml` kein eigenes Netzwerk definiert ist, legt Docker Compose beim Start (`docker compose up`) automatisch ein eigenes Bridge-Netzwerk an. Es trägt standardmäßig den Namen `<projektname>_default`, wobei der Projektname meist aus dem Ordnernamen abgeleitet wird. Alle in der Compose-Datei definierten Services werden ohne weiteres Zutun mit diesem Netzwerk verbunden.

## 2. Kommunikation über Servicenamen

Innerhalb des (Standard- oder eines benutzerdefinierten) Netzwerks betreibt Docker Compose einen internen DNS-Dienst. Jeder Service ist darüber unter seinem in der Compose-Datei vergebenen Namen erreichbar, zum Beispiel kann ein `web`-Service den Datenbank-Container einfach über `db:5432` statt über eine IP-Adresse ansprechen.

Der Servicename sollte anstelle der IP-Adresse verwendet werden, weil Docker jedem Container beim Start eine IP-Adresse dynamisch aus dem Subnetz des Netzwerks zuweist. Diese Adresse wird nicht dauerhaft gespeichert und kann sich bei jedem Neustart oder bei jeder Neuerstellung des Containers ändern. Der Servicename bleibt dagegen stabil, sodass die Kommunikation auch nach einem Neustart oder einem Update des Containers zuverlässig funktioniert.

## 3. Benutzerdefinierte Netzwerke

Eigene Netzwerke werden im Abschnitt `networks:` auf oberster Ebene der Compose-Datei deklariert. Anschließend wird jedem Service über den Schlüssel `networks:` mitgeteilt, mit welchem/welchen dieser Netzwerke er verbunden werden soll. Nur Services, die mindestens ein gemeinsames Netzwerk teilen, können miteinander kommunizieren.

```yaml
services:
  proxy:
    build: ./proxy
    networks:
      - frontend
  app:
    build: ./app
    networks:
      - frontend
      - backend
  db:
    image: postgres:latest
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
```

In diesem Beispiel kann `app` sowohl mit `proxy` als auch mit `db` sprechen, `proxy` und `db` können sich jedoch nicht direkt erreichen, da sie kein gemeinsames Netzwerk besitzen. Zusätzlich lässt sich ein Netzwerk mit `internal: true` so konfigurieren, dass es keinen Zugriff auf externe/öffentliche Netzwerke (Internet) hat.

## 4. Netzwerkmodi (`network_mode`)

Über die Service-Option `network_mode` lässt sich das Standardverhalten überschreiben. Mögliche Werte sind unter anderem:

- **`bridge`** – das Standardverhalten, der Container erhält ein eigenes isoliertes Netzwerk-Interface.
- **`host`** – der Container nutzt direkt den Netzwerk-Stack des Host-Systems, ohne eigene Netzwerk-Isolation.
- **`none`** – der Container erhält keinerlei Netzwerkanbindung.
- **`service:<name>`** – der Container teilt sich den Netzwerk-Stack eines anderen Services.
- **`container:<name>`** – der Container teilt sich den Netzwerk-Stack eines konkreten, über die Container-ID referenzierten Containers.

Der Modus `host` eignet sich vor allem für Anwendungen, die direkten Zugriff auf die Netzwerkschnittstellen des Hosts benötigen oder sehr performante Netzwerkverbindungen ohne den Overhead der Bridge-Netzwerkschicht erfordern, zum Beispiel Monitoring-Tools. Die Einschränkungen dabei sind: Es ist keine Portweiterleitung über `ports:` mehr nötig oder möglich, da der Container die Ports des Hosts direkt nutzt; die Auflösung von Servicenamen über den internen DNS-Dienst von Compose funktioniert in diesem Modus nicht mehr; außerdem steht dieser Modus in vollem Umfang nur unter Linux zur Verfügung, unter Docker Desktop (macOS/Windows) ist er nur eingeschränkt nutzbar.

## 5. Unterschied zwischen `docker compose stop` und `docker compose down`

```bash
docker compose stop
docker compose down
```

`docker compose stop` beendet lediglich die laufenden Container der Anwendung, entfernt sie aber nicht. Container, Netzwerke und Volumes bleiben bestehen, sodass die Anwendung anschließend mit `docker compose start` wieder fortgesetzt werden kann, ohne dass etwas neu erstellt werden muss.

`docker compose down` geht einen Schritt weiter: Es stoppt die Container und entfernt sie anschließend vollständig, zusammen mit den von Compose selbst erstellten Netzwerken (also dem Standardnetzwerk sowie allen nicht als extern markierten benutzerdefinierten Netzwerken). Volumes und Images bleiben dabei standardmäßig erhalten und werden nur entfernt, wenn zusätzlich die Optionen `--volumes` bzw. `--rmi` angegeben werden.

Ein Netzwerk, das in der Compose-Datei mit `external: true` gekennzeichnet ist, wurde nicht von Compose erstellt, sondern muss bereits vorher existieren (z. B. über `docker network create`). Da Compose ein solches Netzwerk nicht besitzt, sondern nur nutzt, wird es auch durch `docker compose down` nicht entfernt. Es müsste bei Bedarf manuell mit `docker network rm` gelöscht werden.

## 6. Kommunikation zwischen unterschiedlichen Compose-Projekten

Da jedes Compose-Projekt standardmäßig sein eigenes, isoliertes Netzwerk erhält, können Container aus verschiedenen Projekten sich normalerweise nicht sehen. Um das zu ermöglichen, wird zunächst manuell ein gemeinsames Netzwerk angelegt:

```bash
docker network create inter-project
```

Dieses Netzwerk wird anschließend in beiden Compose-Dateien als `external` referenziert und den gewünschten Services zugeordnet:

```yaml
services:
  api:
    image: myapi:latest
    networks:
      - shared
      - default

networks:
  shared:
    external: true
    name: inter-project
```

Sobald Services aus unterschiedlichen Projekten mit demselben externen Netzwerk verbunden sind, können sie einander genau wie innerhalb eines einzelnen Projekts über ihren Servicenamen erreichen.

## 7. Netzwerk-Aliase

Ein Netzwerk-Alias ist ein zusätzlicher Hostname, unter dem ein Service innerhalb eines Netzwerks – neben seinem eigentlichen Servicenamen – über den internen DNS-Dienst erreichbar ist. Das ist zum Beispiel praktisch, wenn eine Anwendung einen bestimmten, fest erwarteten Hostnamen voraussetzt, der vom eigentlichen Servicenamen abweicht, oder wenn mehrere Services unter einem gemeinsamen Namen ansprechbar sein sollen.

Definiert werden Aliase pro Service und pro Netzwerk, indem statt einer einfachen Liste von Netzwerknamen ein Objekt mit dem Schlüssel `aliases` angegeben wird:

```yaml
services:
  db:
    image: postgres:latest
    networks:
      backend:
        aliases:
          - database
          - sql-server

networks:
  backend:
    driver: bridge
```

Der `db`-Service ist im Netzwerk `backend` damit sowohl unter `db` als auch unter `database` und `sql-server` erreichbar.

## 8. Dynamische und statische IP-Adressen

Standardmäßig vergibt Docker die IP-Adressen der Container **dynamisch** aus dem Subnetz des jeweiligen Netzwerks. Diese Zuweisung erfolgt bei jedem Start neu und bleibt nicht über Neustarts hinweg erhalten – genau deshalb wird für die Kommunikation zwischen Containern der Servicename statt der IP-Adresse empfohlen (siehe Frage 2).

Falls eine feste (statische) IP-Adresse dennoch benötigt wird, lässt sich das über zwei zusammenhängende Einstellungen konfigurieren: Zunächst wird beim Netzwerk selbst über den `ipam`-Block (IP Address Management) mit einer `config`-Liste ein eigenes `subnet` festgelegt. Anschließend kann jedem Service für dieses Netzwerk über `ipv4_address` eine konkrete, innerhalb dieses Subnetzes liegende Adresse zugewiesen werden:

```yaml
networks:
  backend:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.20.0.0/24

services:
  db:
    image: postgres:latest
    networks:
      backend:
        ipv4_address: 172.20.0.10
```

## 9. Unterschied zwischen Host-Port und Container-Port

Bei der Portangabe eines Services, z. B. `ports: ["8080:80"]`, stehen sich zwei unterschiedliche Ports gegenüber. Der **Container-Port** (hier `80`) ist der Port, auf dem der Prozess innerhalb des Containers tatsächlich lauscht. Der **Host-Port** (hier `8080`) ist der Port auf dem Host-System, über den dieser Dienst von außen – also z. B. über den Browser auf dem Host-Rechner oder aus dem lokalen Netzwerk – erreichbar gemacht wird.

Für die Kommunikation zwischen Containern innerhalb desselben Docker-Netzwerks wird immer der **Container-Port** verwendet. Container sprechen sich direkt über das interne Netzwerk und den dort tatsächlich geöffneten Port an; die `ports`-Zuordnung zum Host-Port spielt dabei keine Rolle und ist nur für den Zugriff von außerhalb des Docker-Netzwerks relevant.

---

**Quelle:** [Networking in Docker Compose – docs.docker.com](https://docs.docker.com/compose/how-tos/networking/)
