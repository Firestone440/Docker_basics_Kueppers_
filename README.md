Hier alle im Video erwähnten Befehle:
Docker installieren:
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh ./get-docker.sh
Falls kein curl installiert: apt install curl

Wenn man nicht mit dem Nutzer Root arbeitet, sollte man den aktuellen Benutzer berechtigen:

sudo usermod -aG docker $USER
Mit Docker arbeiten
Laufende Container auflisten: docker ps
Alle Container auflisten (auch gestoppte): docker ps -a
Einen Container anhalten: docker stop <Containername> (den Namen findet man mit docker ps heraus)
Einen gestoppten Container endgültig löschen: docker rm <Containername>
Einen simplen Webserver starten
Der Container aus dem Image nginx fährt mit folgendem Befehl hoch:

docker run -p 80:80 nginx
Die eigene IP-Adresse erhält man mit ip a

Arbeiten mit Docker-Compose
Legt euch am besten einen eigenen Ordner für das Docker-Projekt an, um Ordnung zu halten. Die Datei docker-compose.yml enthält die Definition der Container.

Bearbeitet wird die Datei mit:

nano docker-compose.yml
Die Inhalte findet ihr unten in diesem GitHub-Gist. Den Texteditor Nano beendet man mit: Strg+X, dann Y

Die Compose-Zusammenstellung hochfahren:

docker compose up -d
Will man die Container updaten, lädt man die neuen Images mit

docker compose pull
Eine oder mehrere Docker-Compose-Dateien?
Das ist definitiv Geschmachssache und hängt von der Umgebung ab. Wenn man mehr als ein Projekt (zum Beispiel einen Blog und ein Pihole) auf einem Server betreibt, sollte man für jedes einen Ordner anlegen und darin eine Docker-Compose-Datei ablegen. Die nützlichen Helfer wie Portainer und Watchtower kommen zusammen in eine weitere Datei. Dann kann man mit docker compose downgezielt Teile der Umgebung herunterfahren.


## Pi-hole

Pi-hole ist ein netzwerkweiter DNS-Sinkhole-Dienst, der Werbe- und Tracking-Domains bereits auf DNS-Ebene blockiert, bevor sie im Browser geladen werden. Er läuft hier als Container mit veröffentlichten Ports für DNS (53/tcp+udp), DHCP (67/udp) und die Weboberfläche (80/tcp), erreichbar unter `http://localhost/admin`.

## Portainer

Portainer ist eine Web-Oberfläche zur Verwaltung von Docker-Umgebungen. Darüber lassen sich Container, Images, Volumes und Netzwerke grafisch starten, stoppen und überwachen, statt jeden Befehl über die Kommandozeile abzusetzen. Erreichbar unter `http://localhost:9000`.

## nginx

nginx ist ein Webserver und Reverse Proxy, der HTTP-Anfragen entgegennimmt und z. B. an dahinterliegende Dienste weiterleitet oder statische Inhalte ausliefert.
