Zadanie 5

1. Healthcheck backendu

Do usługi backend dodano:

    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:5000/health')"]
      interval: 5s
      timeout: 3s
      retries: 5

a depends_on frontendu zmieniono na backend: condition: service_healthy.

Wynik docker compose up -d --build (od tworzenia kontenerów):

 Container breach-checker-db-1 Creating 
 Container breach-checker-db-1 Created 
 Container breach-checker-backend-1 Creating 
 Container breach-checker-backend-1 Created 
 Container breach-checker-frontend-1 Creating 
 Container breach-checker-frontend-1 Created 
 Container breach-checker-db-1 Starting 
 Container breach-checker-db-1 Started 
 Container breach-checker-db-1 Waiting 
 Container breach-checker-db-1 Healthy 
 Container breach-checker-backend-1 Starting 
 Container breach-checker-backend-1 Started 
 Container breach-checker-backend-1 Waiting 
 Container breach-checker-backend-1 Healthy 
 Container breach-checker-frontend-1 Starting 
 Container breach-checker-frontend-1 Started 

Wynik docker compose ps:

NAME                        IMAGE                     COMMAND                  SERVICE    CREATED          STATUS                    PORTS
breach-checker-backend-1    breach-checker-backend    "python app.py"          backend    18 seconds ago   Up 11 seconds (healthy)   5000/tcp
breach-checker-db-1         postgres:16-alpine        "docker-entrypoint.s…"   db         18 seconds ago   Up 17 seconds (healthy)   5432/tcp
breach-checker-frontend-1   breach-checker-frontend   "/docker-entrypoint.…"   frontend   17 seconds ago   Up 6 seconds              0.0.0.0:8080->80/tcp, [::]:8080->80/tcp

2. Trwałość bazy

$ docker compose exec db psql -U breach -d breachdb -c "INSERT INTO leaked_hashes VALUES ('bc45408273d7e1914b741c3407b0b559764f7876');"
INSERT 0 1

$ curl -X POST http://localhost:8080/api/check -H "Content-Type: application/json" -d '{"hash":"bc45408273d7e1914b741c3407b0b559764f7876"}'
{"found":true}

$ docker compose down
$ docker compose up -d
$ curl -X POST http://localhost:8080/api/check -H "Content-Type: application/json" -d '{"hash":"bc45408273d7e1914b741c3407b0b559764f7876"}'
{"found":true}

$ docker compose down -v
$ docker compose up -d
$ curl -X POST http://localhost:8080/api/check -H "Content-Type: application/json" -d '{"hash":"bc45408273d7e1914b741c3407b0b559764f7876"}'
{"found":false}

Skrót przetrwał docker compose down, bo to polecenie usuwa tylko kontenery i sieć, a dane PostgreSQL leżą w nazwanym wolumenie pgdata, który zostaje i jest ponownie podłączany do nowego kontenera db. docker compose down -v usuwa także wolumeny, więc cała zawartość bazy (razem z dopisanym skrótem) znika. Po down -v + up w bazie znów jest 89 skrótów z próbki, bo przy pustym wolumenie obraz postgres przy pierwszym starcie wykonuje skrypty z /docker-entrypoint-initdb.d/ - nasz init.sql tworzy tabelę i wczytuje skróty z pliku data/leaked-sha1.txt podmontowanego z hosta (plik jest w repozytorium, nie w wolumenie), a skrótu dodanego ręcznie INSERT-em w tym pliku nie ma.

3. Skalowanie

Wynik docker compose ps po docker compose up -d --scale backend=3:

NAME                        IMAGE                     COMMAND                  SERVICE    CREATED          STATUS                    PORTS
breach-checker-backend-1    breach-checker-backend    "python app.py"          backend    2 minutes ago    Up 2 minutes (healthy)    5000/tcp
breach-checker-backend-2    breach-checker-backend    "python app.py"          backend    16 seconds ago   Up 14 seconds (healthy)   5000/tcp
breach-checker-backend-3    breach-checker-backend    "python app.py"          backend    16 seconds ago   Up 15 seconds (healthy)   5000/tcp
breach-checker-db-1         postgres:16-alpine        "docker-entrypoint.s…"   db         2 minutes ago    Up 2 minutes (healthy)    5432/tcp
breach-checker-frontend-1   breach-checker-frontend   "/docker-entrypoint.…"   frontend   2 minutes ago    Up 2 minutes              0.0.0.0:8080->80/tcp, [::]:8080->80/tcp

Wynik docker compose exec frontend nslookup backend:

Server:         127.0.0.11
Address:        127.0.0.11:53

Non-authoritative answer:
Name:   backend
Address: 172.18.0.5
Name:   backend
Address: 172.18.0.6
Name:   backend
Address: 172.18.0.3

Non-authoritative answer:


Logi docker compose logs backend (POST /check) - 20 zapytań przed restartem frontendu, wszystkie obsłużyła replika backend-1:

backend-1  | 172.18.0.4 - - [20/Nov/2026 09:58:02] "POST /check HTTP/1.1" 200 -
backend-1  | 172.18.0.4 - - [20/Nov/2026 09:58:02] "POST /check HTTP/1.1" 200 -
backend-1  | 172.18.0.4 - - [20/Nov/2026 09:58:02] "POST /check HTTP/1.1" 200 -
backend-1  | 172.18.0.4 - - [20/Nov/2026 09:58:02] "POST /check HTTP/1.1" 200 -
backend-1  | 172.18.0.4 - - [20/Nov/2026 09:58:02] "POST /check HTTP/1.1" 200 -
backend-1  | 172.18.0.4 - - [20/Nov/2026 09:58:02] "POST /check HTTP/1.1" 200 -

Po docker compose restart frontend i ponownych 20 zapytaniach (z tej serii: backend-1 - 2, backend-2 - 17, backend-3 - 1):

backend-1  | 172.18.0.4 - - [20/Nov/2026 09:58:02] "POST /check HTTP/1.1" 200 -
backend-1  | 172.18.0.4 - - [20/Nov/2026 09:58:06] "POST /check HTTP/1.1" 200 -
backend-2  | 172.18.0.4 - - [20/Nov/2026 09:58:05] "POST /check HTTP/1.1" 200 -
backend-2  | 172.18.0.4 - - [20/Nov/2026 09:58:06] "POST /check HTTP/1.1" 200 -
backend-2  | 172.18.0.4 - - [20/Nov/2026 09:58:06] "POST /check HTTP/1.1" 200 -
backend-2  | 172.18.0.4 - - [20/Nov/2026 09:58:06] "POST /check HTTP/1.1" 200 -
backend-2  | 172.18.0.4 - - [20/Nov/2026 09:58:06] "POST /check HTTP/1.1" 200 -
backend-3  | 172.18.0.4 - - [20/Nov/2026 09:58:06] "POST /check HTTP/1.1" 200 -

Przed restartem cały ruch trafiał do jednej repliki, bo nginx tłumaczy nazwę backend z proxy_pass na adresy tylko raz - przy starcie (wczytaniu konfiguracji). Frontend wystartował, gdy istniała tylko jedna replika, więc nginx znał wyłącznie jej adres; nowe repliki pojawiły się w DNS Dockera (widać je w nslookup), ale nginx o nie nie pytał. Po restarcie nginx ponownie rozwiązał nazwę, dostał trzy adresy i zaczął rozdzielać zapytania między repliki. Gdyby backend miał ports: "5000:5000", każda replika próbowałaby opublikować ten sam port 5000 na hoście - druga i trzecia nie mogłyby wystartować (port already allocated), więc skalowanie by się nie udało; bez publikacji portu repliki są dostępne tylko w sieci Compose pod wspólną nazwą backend.

Na koniec powrót do jednej kopii: docker compose up -d --scale backend=1.
