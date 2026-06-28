# DBMS_09 – From Container to Docker Compose

**Module:** Databases · THGA Bochum  
**Lecturer:** Stephan Bökelmann · <sboekelmann@ep1.rub.de>  
**Repository:** <https://github.com/MaxClerkwell/DBMS_09>  
**Prerequisites:** DBMS_01 – DBMS_07, Lecture 09  
**Duration:** 120 minutes

---

## Learning Objectives

After completing this exercise you will be able to:

- Explain the difference between a **container** and a **virtual machine**
- Start, inspect, and remove Docker **containers** and **images**
- Write a minimal **Dockerfile** and build a custom image from it
- Demonstrate the **ephemeral** nature of container storage
- Attach a **named volume** to a PostgreSQL container and verify data persists
  across container restarts
- Explain why running two services in one container is an anti-pattern
- Connect two containers via a **custom bridge network** using container names
  as hostnames
- Describe all key sections of a **docker-compose.yml** file
- Start a multi-service application with `docker compose up` and take it down
  cleanly
- Use a **`.env` file** to keep credentials out of `docker-compose.yml`
- Explain what a **Multi-Stage Build** is and why it reduces image size
- Apply the **Principle of Least Privilege** by running containers as a
  non-root user
- Use **init scripts** to initialise a PostgreSQL database on first start

**After completing this exercise you should be able to answer the following questions independently:**

- What is the difference between a Docker image and a running container?
- Why does deleting a container without a volume lose all data written to it?
- Why can `host="localhost"` inside one container not reach a second container?
- What does `docker compose down -v` do that `docker compose down` does not?
- Why should credentials never appear directly in `docker-compose.yml`?

---

## Prerequisites Check

You need Docker and Docker Compose installed.

```bash
docker --version
docker compose version
```

> Both commands should succeed and show version numbers.

> **Screenshot 1:** Take a screenshot showing both version outputs.
>
>  ![Screenshot1](https://github.com/Oryanick/DBMS_08/blob/master/Screenshot%201.png)


---

## 1 – Hello Container

### Step 1 – Run the hello-world Image

```bash
docker run hello-world
```

> Docker pulls the image from Docker Hub (first run only), starts a container,
> prints a message, and exits.

List all containers, including stopped ones:

```bash
docker ps -a
docker images
```

> **Screenshot 2:** Take a screenshot showing `docker ps -a` and
> `docker images` output.
>
> ![Screenshot2](https://github.com/Oryanick/DBMS_08/blob/master/Screenshot%202.png)

### Step 2 – Run an nginx Webserver

```bash
docker run -d -p 8080:80 --name webserver nginx
curl http://localhost:8080
docker logs webserver
docker exec -it webserver bash
```

Inside the container shell, inspect the running process and exit:

```bash
ps aux
exit
```

Stop and remove the container:

```bash
docker stop webserver && docker rm webserver
docker system df
```

### Questions for Section 1

**Question 1.1:** The flag `-d` starts the container in detached mode.
What happens without `-d`, and why is detached mode useful for a web server?

> *Your answer:* Ohne das -d Flag läuft der Container im Vordergrund, das heißt das Terminal ist direkt an den laufenden Prozess gebunden und ich kann keine weiteren Befehle eingeben, solange der Container aktiv ist. Der gesamte Output des Containers wird direkt im Terminal angezeigt.
Der Detached Mode startet den Container dagegen im Hintergrund. Das ist besonders sinnvoll für Webserver, weil sie dauerhaft laufen sollen und ich parallel weiterarbeiten kann, ZB: andere Befehle ausführen oder Logs prüfen.

**Question 1.2:** `-p 8080:80` maps host port 8080 to container port 80.
Which port is the application actually listening on inside the container?
What would `-p 9000:80` change?

> *Your answer:* Der Webserver läuft intern im Container auf Port 80. Mit -p 8080:80 wird dieser intern Port auf den Host Port 8080 weitergeleitet. Das bedeutet also, dass man innerhalb des Containers Port 80 haben und auf dem Rechner http://localhost:8080. Wenn ich stattdessen -p 9000:80 verwende, würde die Anwendung dann über http://localhost:9000 erreichbar sein. Der Container selbst bleibt dabei unverändert und hört weiterhin auf Port 80, nur die externe Weiterleitung ändert sich.


---

## 2 – Writing a Dockerfile

### Step 1 – Create the Project Directory

```bash
mkdir ~/dbms09_dockerfile
cd ~/dbms09_dockerfile
git init
git remote add origin git@github.com:<your-username>/dbms09_dockerfile.git
```

### Step 2 – Write the Dockerfile

```bash
vim Dockerfile
```

```dockerfile
FROM debian:13
RUN apt-get update && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
CMD ["bash"]
```

### Step 3 – Build and Run

```bash
docker build -t mein-debian .
docker run -it mein-debian
```

Inside the container:

```bash
curl --version
whoami
exit
```

> **Screenshot 3:** Take a screenshot showing the `docker build` output and
> the commands run inside the container.
>
> ![Screenshot3](https://github.com/Oryanick/DBMS_08/blob/master/Screenshot%203.png)

### Step 4 – Commit

```bash
git add Dockerfile
git commit -m "feat: minimal Debian image with curl"
git push -u origin main
```

### Questions for Section 2

**Question 2.1:** Why does the `RUN` instruction combine `apt-get update`,
`apt-get install`, and `rm -rf /var/lib/apt/lists/*` in a single line?
What would happen to the image size if these were three separate `RUN` lines?

> *Your answer:*  Die drei Befehle werden in einer einzigen RUN Zeile kombiniert, weil Docker jede RUN Anweisung als eigene Image Schicht speichert. Wenn man also apt-get update, apt-get install und das Aufräumen in getrennte RUN Zeilen schreibt, würden die temporären Paketlisten aus apt-get update in einer eigenen Schicht gespeichert bleiben. Diese bleiben dann im Image erhalten und erhöhen die Größe unnötig. Durch das Zusammenfassen passiert alles in einer einzigen Schicht: installieren und direkt danach den Cache löschen, bevor die Schicht eingefroren wird. Dadurch bleibt das Image deutlich kleiner und sauberer.

**Question 2.2:** `EXPOSE 80` in a Dockerfile does **not** actually open port
80. What does it do, and what is required at `docker run` time to actually
forward a port?

> *Your answer:* EXPOSE 80 öffnet keinen Port wirklich, sondern ist nur eine Art Dokumentation im Image. Es sagt nur, dass die Anwendung im Container Port 80 nutzt. Damit der Dienst wirklich von außen erreichbar ist, muss man beim Start des Containers den Port explizit weiterleiten, ZB: docker run -p 8080:80. Erst dieses -p verbindet den Host Port mit dem Container Port. Ohne -p bleibt der Dienst im Container isoliert und ist von außen nicht erreichbar.

---

## 3 – The Persistence Problem

### Step 1 – Start a PostgreSQL Container Without a Volume

```bash
docker run --name pg \
    -e POSTGRES_PASSWORD=geheim \
    -d postgres:16
```

Connect and create a table:

```bash
docker exec -it pg psql -U postgres
```

Inside `psql`:

```sql
CREATE TABLE test (id SERIAL PRIMARY KEY, wert TEXT);
INSERT INTO test (wert) VALUES ('Hallo Docker');
SELECT * FROM test;
\q
```

### Step 2 – Destroy and Recreate the Container

```bash
docker stop pg && docker rm pg
docker run --name pg -e POSTGRES_PASSWORD=geheim -d postgres:16
docker exec -it pg psql -U postgres -c "SELECT * FROM test;"
```

> Expected output: `ERROR: relation "test" does not exist`

> **Screenshot 4:** Take a screenshot showing the error message.
>
> ![Screenshot4](https://github.com/Oryanick/DBMS_08/blob/master/Screenshot%204.png)

### Questions for Section 3

**Question 3.1:** You stopped and removed the container but the image
`postgres:16` still exists on your machine. Why does recreating a container
from the same image not restore the data?

> *Your answer:* Die Daten sind nicht im Docker Image gespeichert, sondern im Container Dateisystem zur Laufzeit. Dieses Dateisystem ist jedoch ephemer, das heißt, sobald der Container gelöscht wird, gehen alle darin gespeicherten Daten verloren. Das Image postgres:16 enthält nur die Software und Grundkonfiguration aber keine Laufzeitdatenbank. Deshalb wird beim Neustart eines neuen Containers aus dem gleichen Image eine komplett frische Datenbank ohne vorherige Tabellen erstellt.

**Question 3.2:** `docker stop` sends SIGTERM and waits for the process to
exit cleanly. `docker kill` sends SIGKILL immediately. Why is `docker stop`
preferred for a database container?

> *Your answer:* docker stop wird bevorzugt, weil es dem Prozess erlaubt, sich kontrolliert zu beenden. Dabei wird zuerst ein SIGTERM Signal gesendet, wodurch PostgreSQL noch offene Verbindungen schließen, Transaktionen beenden und Daten sauber auf die Festplatte schreiben kann. docker kill hingegen beendet den Prozess sofort mit SIGKILL, ohne Cleanup. Das kann bei Datenbanken zu Datenverlust oder beschädigten Daten führen. Deshalb ist docker stop für Stateful Services wie Datenbanken die sichere Variante.

---

## 4 – Named Volumes

### Step 1 – Create a Volume and Attach It

```bash
docker volume create pg_data
docker run --name pg \
    -e POSTGRES_PASSWORD=geheim \
    -v pg_data:/var/lib/postgresql/data \
    -d postgres:16
```

Insert data:

```bash
docker exec -it pg psql -U postgres
```

```sql
CREATE TABLE test (id SERIAL PRIMARY KEY, wert TEXT);
INSERT INTO test (wert) VALUES ('Daten überleben');
SELECT * FROM test;
\q
```

### Step 2 – Destroy and Recreate With the Same Volume

```bash
docker stop pg && docker rm pg
docker run --name pg \
    -e POSTGRES_PASSWORD=geheim \
    -v pg_data:/var/lib/postgresql/data \
    -d postgres:16
docker exec -it pg psql -U postgres -c "SELECT * FROM test;"
```

> The data should still be there.

```bash
docker volume ls
docker volume inspect pg_data
```

> **Screenshot 5:** Take a screenshot showing the `SELECT` result after
> container recreation, and the `docker volume inspect` output.
>
> ![Screenshot5](https://github.com/Oryanick/DBMS_08/blob/master/Screenshot%205.png)

### Step 3 – Clean Up

```bash
docker stop pg && docker rm pg
docker volume rm pg_data
```

### Questions for Section 4

**Question 4.1:** `docker volume inspect pg_data` shows a `Mountpoint` on
the host filesystem. Why is it still recommended to use named volumes instead
of bind-mounting that path directly with `-v /var/lib/docker/volumes/...`?

> *Your answer:* Auch wenn docker volume inspect pg_data den konkreten Mountpoint auf dem Host Dateisystem anzeigt, sollte man trotzdem benannte Volumes verwenden, anstatt direkt den Pfad unter /var/lib/docker/volumes/... als Bind-Mount einzubinden. Der wichtigste Grund ist, dass dieser Pfad intern von Docker verwaltet wird und sich je nach System, Docker Version oder Storage Driver ändern kann. Außerdem ist die direkte Arbeit in diesem Verzeichnis nicht empfohlen, da Docker dort selbst Daten organisiert und man durch manuelle Änderungen leicht Inkonsistenzen oder Datenverlust verursachen kann. Benannte Volumes sind dagegen abstrahiert, sicherer und plattformunabhängig, da Docker die Speicherorte automatisch verwaltet und bei Bedarf korrekt an Container bindet.

**Question 4.2:** You want to back up the database. Which `docker` command
lets you copy files out of a running container, and how would you copy the
volume contents to a `.tar.gz` archive on the host?

> *Your answer:* Um Daten aus einem laufenden Container zu sichern, kann der Befehl docker cp verwendet werden. Damit lassen sich Dateien oder Verzeichnisse direkt zwischen Container und Host System kopieren. Für ein PostgreSQL Volume würde ich den Inhalt des Datenverzeichnisses aus dem Container bzw Volume zunächst auf das Host System kopieren und anschließend daraus ein Archiv erstellen. Konkret könnte man beispielsweise mit docker cp pg:/var/lib/postgresql/data ./backup die Daten aus dem Container in ein lokales Backup Verzeichnis kopieren. Danach kann dieses Verzeichnis mit einem Befehl wie tar -czvf backup.tar.gz backup/ auf dem Host in ein komprimiertes Archiv umgewandelt werden. Dadurch entsteht eine vollständige Sicherung der Datenbank, die unabhängig vom Container gespeichert werden kann.

---

## 5 – Two Containers and the Network Problem

### Step 1 – Reproduce the Connectivity Failure

Start PostgreSQL:

```bash
docker run --name pg \
    -e POSTGRES_PASSWORD=geheim \
    -v pg_data:/var/lib/postgresql/data \
    -d postgres:16
```

Start a second container and try to reach the first via `localhost`:

```bash
docker run --rm -it postgres:16 \
    psql -h localhost -U postgres -c "SELECT 1;"
```

> This should fail with a connection refused error.

> **Screenshot 6:** Take a screenshot showing the connection error.
>
> ![Screenshot6](https://github.com/Oryanick/DBMS_08/blob/master/Screenshot%206.png)

### Step 2 – Fix It With a Custom Bridge Network

```bash
docker network create mein-netz

docker stop pg && docker rm pg
docker run --name pg \
    -e POSTGRES_PASSWORD=geheim \
    -v pg_data:/var/lib/postgresql/data \
    --network mein-netz \
    -d postgres:16

docker run --rm -it \
    --network mein-netz \
    postgres:16 \
    psql -h pg -U postgres -c "SELECT 1;"
```

> Notice that `-h pg` uses the **container name** as the hostname — Docker's
> internal DNS resolves it automatically.

```bash
docker network inspect mein-netz
```

### Step 3 – Clean Up

```bash
docker stop pg && docker rm pg
docker network rm mein-netz
docker volume rm pg_data
```

### Questions for Section 5

**Question 5.1:** Without a custom bridge network, containers are placed on
the default bridge. Why can containers on the default bridge **not** resolve
each other by name, while containers on a user-defined bridge can?

> *Your answer:* Auf der Standard-Bridge von Docker gibt es keine automatische DNS Auflösung für Containernamen, das heißt Container kennen sich dort nicht über ihre Namen, sondern nur über IP-Adressen. Dadurch kann ein Container den anderen nicht einfach über einen Hostnamen wie pg erreichen. Bei einem benutzerdefinierten Bridge-Netzwerk ist das anders, weil Docker dort automatisch einen internen DNS Server aktiviert, der die Containernamen auf die jeweiligen IP-Adressen auflöst. Deshalb funktioniert dort die Kommunikation über Namen direkt und ohne feste IPs.

**Question 5.2:** You could find the IP address of the `pg` container with
`docker inspect` and hard-code it. Why is using the container name as a
hostname strongly preferable?

> *Your answer:* Die Verwendung der IP-Adresse ist zwar technisch möglich, aber nicht sinnvoll, weil sich die IP-Adresse eines Containers bei jedem Neustart ändern kann. Wenn ich diese IP fest im Code oder in Konfigurationen eintrage, würde die Verbindung schnell kaputtgehen, sobald der Container neu erstellt wird. Der Containernamen ist dagegen stabil und bleibt gleich, solange der Container existiert. In einem benutzerdefinierten Netzwerk übernimmt Docker die Auflösung automatisch, sodass ich einfach den Namen pg verwenden kann und keine festen IPs brauche. Das ist deutlich flexibler und entspricht der typischen Docker Arbeitsweise.

---

## 6 – Docker Compose

Managing multiple `docker run` commands by hand is error-prone. Docker Compose
describes an entire multi-service application in a single YAML file.

### Step 1 – Create the Project

```bash
mkdir ~/dbms09_compose
cd ~/dbms09_compose
git init
git remote add origin git@github.com:<your-username>/dbms09_compose.git
```

### Step 2 – Write docker-compose.yml

```bash
vim docker-compose.yml
```

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: vorlesung
      POSTGRES_PASSWORD: geheim
      POSTGRES_DB: vorlesung
    volumes:
      - pg_data:/var/lib/postgresql/data
    networks:
      - backend

  api:
    image: python:3.12-slim
    working_dir: /app
    volumes:
      - ./api:/app
    command: >
      bash -c "pip install fastapi uvicorn psycopg2-binary --quiet
               && uvicorn main:app --host 0.0.0.0 --port 8000"
    ports:
      - "8000:8000"
    depends_on:
      - postgres
    networks:
      - backend

volumes:
  pg_data:

networks:
  backend:
```

### Step 3 – Create the API Code

```bash
mkdir api
vim api/main.py
```

```python
import psycopg2
import psycopg2.extras
from fastapi import FastAPI

app = FastAPI(title="Studenten-API")

DB_CONFIG = {
    "dbname": "vorlesung",
    "user": "vorlesung",
    "password": "geheim",
    "host": "postgres",   # container name as hostname
    "port": 5432,
}

@app.get("/")
def root():
    return {"status": "API läuft"}

@app.get("/studenten")
def alle_studenten():
    conn = psycopg2.connect(**DB_CONFIG)
    cur = conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor)
    cur.execute("SELECT id, matrikel, nachname, vorname FROM student ORDER BY nachname")
    rows = cur.fetchall()
    cur.close()
    conn.close()
    return rows
```

### Step 4 – Start and Test

```bash
docker compose up -d
docker compose ps
docker compose logs api
```

Wait for the API to start, then test:

```bash
curl http://localhost:8000/
curl http://localhost:8000/studenten
```

> The `/studenten` endpoint will return an empty list for now — that is
> expected. You will add the schema in Section 7.

> **Screenshot 7:** Take a screenshot showing `docker compose ps` and the
> `curl /` response.
>
> ![Screenshot7](https://github.com/Oryanick/DBMS_08/blob/master/Screenshot%207.png)

### Step 5 – Observe Compose Networking

```bash
docker network ls
docker network inspect dbms09_compose_backend
```

> Compose automatically prefixes network names with the project directory name.

### Step 6 – Take It Down

```bash
docker compose down
docker compose down -v   # also removes the named volume
```

> **Question:** What is the difference between `down` and `down -v`?
> When would you use each?

> Der Unterschied zwischen docker compose down und docker compose down -v liegt darin, was beim Herunterfahren der Anwendung gelöscht wird. docker compose down stoppt und entfernt nur die Container sowie das zugehörige Netzwerk, die durch Compose erstellt wurden, die Volumes bleiben dabei aber erhalten. Dadurch bleiben persistente Daten ( ZB:Datenbanken) erhalten und können beim nächsten Start wieder verwendet werden. docker compose down -v geht einen Schritt weiter und löscht zusätzlich alle zugehörigen Volumes, also auch die gespeicherten Daten der Datenbank. Dadurch wird die Umgebung komplett sauber rückgesetzt. Man verwendet docker compose down, wenn man die Anwendung nur stoppen oder neu starten möchte, ohne Daten zu verlieren. docker compose down -v nutzt man dagegen, wenn man wirklich alles zurücksetzen will.


### Step 7 – Commit

```bash
git add docker-compose.yml api/main.py
git commit -m "feat: initial docker-compose setup with postgres and api"
git push -u origin main
```

### Questions for Section 6

**Question 6.1:** `depends_on: postgres` ensures the `postgres` service
starts before `api`. Does it guarantee that PostgreSQL is **ready to accept
connections** when the API starts? What is the correct way to handle this?

> *Your answer:* depends_on stellt sicher, dass der PostgreSQL Container zuerst gestartet wird, aber es garantiert nicht, dass der Dienst innerhalb des Containers bereits vollständig bereit ist und Verbindungen annimmt. Die API kann also starten, während PostgreSQL noch initialisiert wird, was zu Verbindungsfehlern führen kann. Korrekt handhabt man das, indem man zusätzlich eine Healthcheck Konfiguration im Compose File verwendet oder in der API Retry Mechanismen einbaut, sodass erst verbunden wird, wenn die Datenbank wirklich erreichbar ist.

**Question 6.2:** The `api` service uses `volumes: - ./api:/app` (a bind
mount). What is the advantage of this during development compared to
`COPY`-ing the code into an image at build time?

> *Your answer:* Der Vorteil eines Bind-Mounts ist, dass Änderungen am lokalen Quellcode sofort im Container verfügbar sind, ohne dass das Image neu gebaut werden muss. Dadurch kann ich während der Entwicklung viel schneller arbeiten und Änderungen direkt testen. Beim COPY müsste ich jedes Mal das Image neu bauen, was deutlich langsamer und umständlicher wäre. Deshalb ist Bind-Mount ideal für die Entwicklung, während COPY eher für produktive Builds geeignet ist. 


---

## 7 – Init Script for PostgreSQL

The official `postgres` image runs all scripts placed in
`/docker-entrypoint-initdb.d/` on first start. This lets you initialise the
schema automatically.

### Step 1 – Write init.sql

```bash
vim init.sql
```

```sql
CREATE TABLE IF NOT EXISTS student (
    id        INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    matrikel  CHAR(8)      NOT NULL UNIQUE,
    nachname  VARCHAR(100) NOT NULL,
    vorname   VARCHAR(100) NOT NULL,
    email     VARCHAR(200)
);

INSERT INTO student (matrikel, nachname, vorname, email) VALUES
    ('12345678', 'Meier',   'Anna',  'a.meier@stud.thga.de'),
    ('23456789', 'Schmidt', 'Ben',   'b.schmidt@stud.thga.de'),
    ('34567890', 'Yilmaz',  'Ceren', 'c.yilmaz@stud.thga.de'),
    ('45678901', 'Nguyen',  'David', 'd.nguyen@stud.thga.de');
```

### Step 2 – Mount the Script in docker-compose.yml

Add a bind mount to the `postgres` service so the script lands in the init
directory:

```yaml
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: vorlesung
      POSTGRES_PASSWORD: geheim
      POSTGRES_DB: vorlesung
    volumes:
      - pg_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - backend
```

### Step 3 – Reinitialise and Test

The init script only runs when the data directory is empty. Remove the
existing volume first:

```bash
docker compose down -v
docker compose up -d
```

Wait a moment, then query the data:

```bash
curl http://localhost:8000/studenten
```

> You should now see all four students in the JSON response.

> **Screenshot 8:** Take a screenshot showing the `curl /studenten` response
> with all four rows.
>
> ![Screenshot8](https://github.com/Oryanick/DBMS_08/blob/master/Screenshot%208.png)

### Step 4 – Commit

```bash
git add init.sql docker-compose.yml
git commit -m "feat: add init.sql for automatic schema and seed data"
git push
```

### Questions for Section 7

**Question 7.1:** You run `docker compose down` (without `-v`), change
`init.sql`, and run `docker compose up -d` again. The schema change does
**not** appear in the database. Why not, and how do you force re-initialisation?

> *Your answer:* Wenn ich docker compose down ohne -v ausführe, bleibt das Docker Volume für PostgreSQL erhalten. Das bedeutet, dass die Datenbank weiter mit dem bestehenden Datenverzeichnis arbeitet. Das init.sql wird vom Postgres Container nur beim allerersten Initialisieren einer leeren Datenbank ausgeführt. Da das Volume noch existiert, wird das Skript nicht erneut ausgeführt und Änderungen am Schema werden ignoriert. Um die Initialisierung erneut zu erzwingen, muss das Volume gelöscht werden, also docker compose down -v, damit die Datenbank komplett neu erstellt wird und init.sql erneut ausgeführt wird.

**Question 7.2:** `GENERATED ALWAYS AS IDENTITY` is used instead of
`SERIAL`. What is the practical difference? Which one is the modern
SQL-standard approach?

> *Your answer:* SERIAL ist ein älterer PostgreSQL spezifischer Mechanismus, der intern eine Sequenz erstellt und automatisch hochzählt, aber technisch kein echter SQL Standard ist. GENERATED ALWAYS AS IDENTITY ist die moderne und standardkonforme Lösung nach SQL:2011. Der Vorteil ist, dass die Spalte klar als automatisch generierte Identität definiert ist und besser mit anderen Datenbanksystemen kompatibel ist. Außerdem bietet IDENTITY mehr Kontrolle über das Verhalten der Generierung und ist der empfohlene moderne Ansatz gegenüber SERIAL.

---

## 8 – Secrets via .env

Passwords must not appear in `docker-compose.yml` because that file is
committed to version control.

### Step 1 – Create .env

```bash
vim .env
```

```
POSTGRES_USER=vorlesung
POSTGRES_PASSWORD=geheim
POSTGRES_DB=vorlesung
```

### Step 2 – Reference Variables in docker-compose.yml

Replace the hard-coded values in the `postgres` service:

```yaml
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
```

Also update `api/main.py` to read the password from the environment:

```python
import os

DB_CONFIG = {
    "dbname": os.environ.get("POSTGRES_DB", "vorlesung"),
    "user": os.environ.get("POSTGRES_USER", "vorlesung"),
    "password": os.environ.get("POSTGRES_PASSWORD", ""),
    "host": "postgres",
    "port": 5432,
}
```

Add `env_file` to the `api` service in `docker-compose.yml`:

```yaml
  api:
    ...
    env_file: .env
```

### Step 3 – Gitignore .env

```bash
echo ".env" >> .gitignore
```

### Step 4 – Restart and Verify

```bash
docker compose down -v
docker compose up -d
curl http://localhost:8000/studenten
```

> The response should be identical to before.

### Step 5 – Commit

```bash
git add docker-compose.yml api/main.py .gitignore
git commit -m "feat: move credentials to .env file"
git push
```

> **Do not add `.env` to the commit.** Confirm with `git status` that it
> is untracked.

> **Screenshot 9:** Take a screenshot showing `git status` confirming
> `.env` is not staged, and the working `curl` response.
>
> ![Screenshot9](https://github.com/Oryanick/DBMS_08/blob/master/Screenshot%209.png)

### Questions for Section 8

**Question 8.1:** A teammate clones your repository and runs
`docker compose up -d`. The application fails because `.env` is missing.
What is the standard practice to document which variables are required
without committing the actual secrets?

> *Your answer:* Üblicherweise dokumentiert man die benötigten Umgebungsvariablen über eine separate Beispieldatei wie .env.example oder .env.template, in der nur die Variablennamen enthalten sind, aber keine echten Geheimnisse stehen. Dadurch kann jeder Entwickler sehen, welche Variablen benötigt werden und eine eigene .env Datei lokal erstellen, ohne dass sensible Daten im Repository gespeichert werden.

**Question 8.2:** Even with `.env` excluded from git, the password is still
stored in plain text on disk. Name one mechanism Docker provides for
production-grade secret management that avoids plain-text env files entirely.

> *Your answer:* Docker bietet für Produktionsumgebungen sogenannte Docker Secrets, die speziell für die sichere Verwaltung sensibler Daten wie Passwörter entwickelt wurden. Secrets werden verschlüsselt gespeichert und nur zur Laufzeit in den Container eingebunden sodass sie nicht als Umgebungsvariable im Klartext sichtbar sind. Dieser Mechanismus wird insbesondere in Docker Swarm oder Kubernetes ähnlichen Setups verwendet und ist deutlich sicherer als die Verwendung von .env Dateien.

---

## 9 – Multi-Stage Build

A Python image that includes `pip`, build tools, and cache is larger than
necessary. A Multi-Stage Build separates dependency installation from the
final runtime image.

### Step 1 – Create an api/Dockerfile

```bash
vim api/Dockerfile
```

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY pyproject.toml .
RUN pip install uv && uv sync --no-dev

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app/.venv .venv
COPY . .
ENV PATH="/app/.venv/bin:$PATH"
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Step 2 – Add pyproject.toml

```bash
vim api/pyproject.toml
```

```toml
[project]
name = "studenten-api"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "fastapi",
    "uvicorn[standard]",
    "psycopg2-binary",
]
```

### Step 3 – Switch the api Service to Use the Build

Replace the `image:` key with `build:` in the `api` service:

```yaml
  api:
    build: ./api
    ports:
      - "8000:8000"
    depends_on:
      - postgres
    env_file: .env
    networks:
      - backend
```

Remove the `volumes: - ./api:/app` bind mount and the `command:` key — they
were only needed for the quick-start approach.

### Step 4 – Build and Test

```bash
docker compose down -v
docker compose build
docker images    # compare sizes
docker compose up -d
curl http://localhost:8000/studenten
```

> **Screenshot 10:** Take a screenshot showing `docker images` with the
> final image size and the working `curl` response.
>
> ![Screenshot10](https://github.com/Oryanick/DBMS_08/blob/master/Screenshot%2010.png)

### Step 5 – Commit

```bash
git add api/Dockerfile api/pyproject.toml docker-compose.yml
git commit -m "feat: multi-stage Dockerfile for slim production image"
git push
```

### Questions for Section 9

**Question 9.1:** `COPY --from=builder /app/.venv .venv` copies the virtual
environment from the builder stage. The final image does not contain `pip` or
`uv`. What security advantage does this provide?

> *Your answer:* Wenn die virtuelle Umgebung aus der Builder Phase ins finale Image kopiert wird, enthält das Runtime Image keine Build Tools wie pip oder uv. Dadurch wird die Angriffsfläche deutlich reduziert, weil potenzielle Werkzeuge zum Nachinstallieren oder Manipulieren von Paketen im laufenden Container nicht mehr vorhanden sind. Das Image enthält nur die tatsächlich benötigten Laufzeit Abhängigkeiten, was es sicherer und kontrollierter macht, da weniger Komponenten für Missbrauch oder Supply-Chain-Angriffe zur Verfügung stehen.

**Question 9.2:** The builder stage installs dependencies from `pyproject.toml`
before copying the application code. Why does this ordering improve build
cache efficiency when you frequently change only `main.py`?

> *Your answer:* Die Reihenfolge verbessert den Docker-Build-Cache, weil pyproject.toml zuerst kopiert und die Abhängigkeiten installiert werden, bevor der eigentliche Anwendungscode ( main.py usw..) ins Image kommt. Wenn sich später nur der Code ändert aber die Dependencies gleich bleiben, kann Docker den Schritt mit uv sync aus dem Cache wiederverwenden. Dadurch muss nicht jedes Mal alles neu installiert werden, was den Build deutlich schneller und effizienter macht, besonders bei häufigen Codeänderungen.

---

## 10 – Non-Root User

Containers run as `root` by default. If an attacker escapes the container,
they have root on the host.

### Step 1 – Add a Non-Root User to the Dockerfile

Open `api/Dockerfile` and add the user before the `CMD`:

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY pyproject.toml .
RUN pip install uv && uv sync --no-dev

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app/.venv .venv
COPY . .
ENV PATH="/app/.venv/bin:$PATH"
RUN adduser --disabled-password --gecos "" appuser
USER appuser
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Step 2 – Rebuild and Verify

```bash
docker compose build
docker compose up -d
docker compose exec api whoami
```

> Expected output: `appuser`

> **Screenshot 11:** Take a screenshot showing `docker compose exec api whoami`
> returning `appuser`.
>
> ![Screenshot11](https://github.com/Oryanick/DBMS_08/blob/master/Screenshot%2011.png)

### Step 3 – Commit

```bash
git add api/Dockerfile
git commit -m "feat: run api container as non-root appuser"
git push
```

### Questions for Section 10

**Question 10.1:** The `USER appuser` instruction is placed after
`COPY . .`. Why would placing it *before* `COPY` cause a permission problem?

> *Your answer:* Wenn USER appuser vor COPY . . . stehen würde, hätte der Benutzer appuser noch keine Schreibrechte oder Besitzrechte an den Zielverzeichnissen im Container. Da der COPY Befehl dann mit diesem nicht root Benutzer ausgeführt wird, könnten Dateien nicht korrekt kopiert oder erstellt werden. Dadurch entstehen Permission Errors oder unvollständige Builds.

**Question 10.2:** State the **Principle of Least Privilege** in one
sentence, and name one other place in a typical web application stack
(outside of containers) where this principle is applied.

> *Your answer:* Das Prinzip der minimalen Rechtevergabe besagt, dass ein Benutzer oder Dienst nur die Rechte erhalten soll, die er unbedingt für seine Aufgabe benötigt. Dadurch wird die Angriffsfläche reduziert und die Sicherheit erhöht. Ein Beispiel außerhalb von Containern ist ein Datenbankbenutzer, der nur SELECT oder INSERT Rechte auf bestimmte Tabellen besitzt und keine administrativen Rechte auf das gesamte System hat.

---

## 11 – Reflection

**Question A – The Monolith Anti-Pattern:**  
Section 6 of the lecture shows a Dockerfile that runs both PostgreSQL and
FastAPI in a single container. Describe two concrete operational problems
this causes in a production environment.

> *Your answer:* Wenn PostgreSQL und FastAPI im selben Container laufen, entstehen zwei konkrete Probleme: Erstens ist die Skalierung nicht möglich, da Datenbank und API nur gemeinsam gestartet oder gestoppt werden können. Zweitens wird das Monitoring und die Fehlerdiagnose schwieriger weil Logs und Prozesse nicht sauber getrennt sind und ein Fehler beide Dienste gleichzeitig beeinträchtigen kann.

**Question B – Volume vs. Bind Mount:**  
Compare named volumes and bind mounts. When is each type appropriate?

> *Your answer:* Benannte Volumes werden vom Docker Engine verwaltet und eignen sich gut für persistente Daten wie Datenbanken in Produktionsumgebungen, da sie unabhängig vom Host Dateisystem sind. Bind Mounts hingegen verbinden ein Verzeichnis vom Host direkt mit dem Container und sind besonders für die Entwicklung geeignet weil Änderungen am Code sofort sichtbar sind.

**Question C – Compose and Reproducibility:**  
A colleague says: "I can just write the `docker run` commands in a shell
script — why do I need `docker-compose.yml`?" Give two specific advantages
of Compose over a shell script of `docker run` commands.

> *Your answer:* Docker Compose bietet eine deklarative Struktur in der alle Dienste, Netzwerke und Volumes zentral in einer YAML Datei definiert werden, wodurch die Konfiguration einfacher wartbar und reproduzierbar wird. Außerdem verwaltet Compose automatisch Abhängigkeiten, Netzwerke und Startreihenfolgen, was mit reinen docker run Skripten deutlich fehleranfälliger ist.

**Question D – The Complete Chain:**  
You have now built and containerised the full stack: PostgreSQL in a
container with a named volume and init script → FastAPI in a slim
non-root image → both orchestrated by Docker Compose with credentials
in `.env`. Describe in two sentences what each layer contributes to
**portability** and **security**.

> *Your answer:* PostgreSQL wird in einem isolierten Container mit persistentem Volume betrieben und durch ein Init Skript automatisch initialisiert, was Datenpersistenz und Reproduzierbarkeit gewährleistet. FastAPI läuft in einem schlanken, nicht root Container und wird über Docker Compose zusammen mit der Datenbank orchestriert, wobei .env Dateien die sichere und portable Konfiguration der Zugangsdaten ermöglichen.

---

## Further Reading

- [Docker – Get started](https://docs.docker.com/get-started/)
- [Docker – Volumes](https://docs.docker.com/storage/volumes/)
- [Docker – Networking overview](https://docs.docker.com/network/)
- [Docker Compose – Reference](https://docs.docker.com/compose/compose-file/)
- [Docker – Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [postgres Docker Hub – Environment variables](https://hub.docker.com/_/postgres)
- Lecture 09 handout
