# TP1 — Discover Docker : application web 3-tiers


**Architecture finale**


 Client ──:80──▶ httpd (reverse proxy) ──▶ backend (Spring Boot :8080) ──▶ database (PostgreSQL :5432)
 

Seul httpd est exposé sur la machine hôte (port 80). Le backend et la base ne sont joignables que par le réseau Docker interne.

## 1. Structure du dépôt

Un dossier par image :

TP1/
├── docker-compose.yml
├── .env                    
├── .env.example            
├── .gitignore
├── README.md
├── database/
│   ├── Dockerfile
│   └── init/
│       ├── 01-CreateScheme.sql
│       └── 02-InsertData.sql
├── backend/               
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/...
├── httpd/
│   ├── Dockerfile
│   ├── httpd.conf
│   └── index.html
└── sandbox/                
    ├── hello-java/         
    └── hello-multistage/ 


## 2. Mise en place

Prérequis : Docker et Docker Compose v2 (`docker --version`, `docker compose version`).

Réseau créé pour les premières étapes (avant docker-compose). On utilise `--network` plutôt que `--link` (déprécié) :


docker network create app-network
Fichier `.env.example` (modèle, sans valeurs) :

env :
POSTGRES_DB=
POSTGRES_USER=
POSTGRES_PASSWORD=


Chacun copie ce fichier en `.env` et le remplit (`db` / `usr` / `pwd` dans ce TP).



## 3. Base de données (PostgreSQL)

### 3.1 Image de base et premier test

Image : `postgres:17.2-alpine`.

Première version du Dockerfile (variables en dur, uniquement pour tester) :

```dockerfile
FROM postgres:17.2-alpine

ENV POSTGRES_DB=db \
    POSTGRES_USER=usr \
    POSTGRES_PASSWORD=pwd
```

```bash
docker build -t my-database ./database
docker run -d --name my-database --network app-network my-database
docker ps
docker logs my-database       # "database system is ready to accept connections"
```

### 3.2 Adminer

```bash
docker run -d -p "8090:8080" --net=app-network --name=adminer adminer
```

Accès sur <http://localhost:8090> : système `PostgreSQL`, serveur `my-database`, utilisateur `usr`, mot de passe `pwd`, base `db`.

### 3.3 Variables d'environnement avec `-e`

Les `ENV` sont retirés du Dockerfile et passés au lancement :

```bash
docker run -d --name my-database --network app-network \
  -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd \
  my-database
```

**Question 1-1 — Pourquoi `-e` plutôt que les variables dans le Dockerfile ?**

Les valeurs écrites dans un Dockerfile sont figées dans l'image : quiconque récupère l'image ou le dépôt peut les lire (`docker history`, `docker inspect`), ce qui est une faille de sécurité pour un mot de passe. L'image devient aussi non réutilisable : changer une valeur oblige à reconstruire l'image. Avec `-e` (ou un fichier `.env`, ou des secrets en production), l'image reste générique et portable, et la configuration est injectée à l'exécution, différente selon l'environnement (dev, test, prod), sans jamais être commitée.

### 3.4 Initialisation du schéma et des données

Les scripts `.sql` placés dans `/docker-entrypoint-initdb.d` (dans le conteneur) sont exécutés au premier démarrage, dans l'ordre alphabétique : `database/init/01-CreateScheme.sql` crée les tables `departments` et `students`, puis `02-InsertData.sql` insère 3 départements et 4 étudiants.

Dockerfile final (`database/Dockerfile`) :

```dockerfile
FROM postgres:17.2-alpine
COPY init/ /docker-entrypoint-initdb.d/
```

Vérification :

```bash
docker build -t my-database ./database
docker run -d --name my-database --network app-network \
  -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd \
  my-database
docker exec -it my-database psql -U usr -d db -c "SELECT * FROM students;"
```

### 3.5 Persistance

```bash
docker run -d --name my-database --network app-network \
  -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd \
  -v "$(pwd)/data:/var/lib/postgresql/data" \
  my-database
```

Test : insérer une ligne, supprimer le conteneur (`docker rm -f my-database`), le relancer avec le même `-v`, puis vérifier que la ligne est toujours là.

**Question 1-2 — Pourquoi un volume pour le conteneur Postgres ?**

Le système de fichiers d'un conteneur est éphémère : il disparaît avec lui (suppression, recréation lors d'une mise à jour d'image, crash). Une base de données doit au contraire conserver ses données durablement. Un volume stocke les données en dehors du conteneur et les rend indépendantes de son cycle de vie : on peut détruire, recréer ou mettre à jour le conteneur sans rien perdre. Cela facilite aussi les sauvegardes et offre de meilleures performances d'I/O que la couche inscriptible du conteneur.

**Question 1-3 — Essentiels du conteneur base de données**

Dockerfile :

```dockerfile
FROM postgres:17.2-alpine
COPY init/ /docker-entrypoint-initdb.d/
```

Commandes :

```bash
docker network create app-network
docker build -t my-database ./database
docker run -d --name my-database --network app-network \
  -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd \
  -v "$(pwd)/data:/var/lib/postgresql/data" my-database
docker run -d -p 8090:8080 --net=app-network --name=adminer adminer
```

Points clés : image Alpine légère, scripts d'initialisation copiés dans `/docker-entrypoint-initdb.d`, secrets passés au runtime, données persistées par un volume sur `/var/lib/postgresql/data`, communication par réseau Docker dédié.

---

## 4. Backend API (Java / Spring Boot)

### 4.1 Hello World avec un JRE (`sandbox/hello-java`)

Un simple runtime suffit pour exécuter du bytecode : image `eclipse-temurin:21-jre-alpine`.

```java
public class Main {
    public static void main(String[] args) {
        System.out.println("Hello World!");
    }
}
```

Compilation locale (`javac Main.java`) puis :

```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY Main.class .
CMD ["java", "Main"]
```

```bash
docker build -t hello-java ./sandbox/hello-java
docker run --rm hello-java       # => Hello World!
```

### 4.2 Multistage (`sandbox/hello-multistage`)

Docker compile lui-même : JDK pour compiler, JRE pour exécuter.

```dockerfile
FROM eclipse-temurin:21-jdk-alpine
WORKDIR /usr/src
COPY Main.java .
RUN javac Main.java

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=0 /usr/src/Main.class .
CMD ["java", "Main"]
```

### 4.3 API Spring Boot simple


Dockerfile (build + exécution, avec cache des dépendances Maven) :

```dockerfile
# Build stage
FROM eclipse-temurin:21-jdk-alpine AS myapp-build
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME

RUN apk add --no-cache maven

COPY pom.xml .
RUN mvn dependency:go-offline

COPY src ./src
RUN mvn package -DskipTests

# Run stage
FROM eclipse-temurin:21-jre-alpine
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

```bash
docker build -t simpleapi-helloworld ./simpleapi-helloworld
docker run --rm -p 8080:8080 simpleapi-helloworld
curl http://localhost:8080/
```

**Question 1-4 — Pourquoi un multistage ? Explication du Dockerfile**

Pourquoi ?
 Compiler du Java demande un JDK, Maven et les sources, mais l'exécution ne demande qu'un JRE et le `.jar`. Le multistage permet de compiler dans Docker tout en ne livrant que le nécessaire : image beaucoup plus légère, surface d'attaque réduite (moins de paquets, donc moins de vulnérabilités), build reproductible (aucun JDK ni Maven à installer en local) et un seul Dockerfile pour tout le cycle.

Explication ligne par ligne :

| Instruction | Rôle |
|---|---|
| `FROM eclipse-temurin:21-jdk-alpine AS myapp-build` | Début de l'étape de build : JDK 21 sur Alpine, étape nommée `myapp-build`. |
| `ENV MYAPP_HOME=/opt/myapp` | Variable avec le répertoire de travail, pour ne pas répéter le chemin. |
| `WORKDIR $MYAPP_HOME` | Définit (et crée) le répertoire courant. |
| `RUN apk add --no-cache maven` | Installe Maven via Alpine, sans garder le cache. |
| `COPY pom.xml .` | Copie seulement le descripteur Maven, pour exploiter le cache de couches. |
| `RUN mvn dependency:go-offline` | Télécharge toutes les dépendances ; cette couche n'est reconstruite que si `pom.xml` change. |
| `COPY src ./src` | Copie le code source (qui change souvent, donc après les dépendances). |
| `RUN mvn package -DskipTests` | Compile et empaquette le `.jar` dans `target/`, sans lancer les tests. |
| `FROM eclipse-temurin:21-jre-alpine` | Étape finale : un JRE seulement, plus léger. |
| `ENV` / `WORKDIR` | Même répertoire de travail dans l'image finale. |
| `COPY --from=myapp-build ...` | Récupère uniquement le `.jar` de l'étape précédente et le renomme `myapp.jar`. Le reste de l'étape de build est abandonné. |
| `ENTRYPOINT ["java","-jar","myapp.jar"]` | Commande de démarrage du conteneur. |



### 4.4 Backend connecté à la base (`backend/`)

On a remplacé `src/` et `pom.xml` par ceux du dépôt `simple-api` fourni (Spring Boot 3.4.2, Java 21, Spring Data JPA, JDBC, PostgreSQL). Le backend accède à la base via le réseau Docker : le nom du service sert de nom d'hôte, donc `jdbc:postgresql://database:5432/db` (et pas `localhost`, qui désignerait le conteneur backend lui-même).

`backend/src/main/resources/application.yml` :

```yaml
spring:
  datasource:
    url: jdbc:postgresql://database:5432/db
    username: usr
    password: pwd
    driver-class-name: org.postgresql.Driver
```

Avec docker-compose, ces valeurs sont surchargées par les variables d'environnement `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME` et `SPRING_DATASOURCE_PASSWORD` (voir §6), de sorte qu'on peut ne plus dépendre des valeurs écrites dans le fichier.

Test : `GET /departments/IRC/students` retourne :

```json
[{"id":1,"firstname":"Eli","lastname":"Copter","department":{"id":1,"name":"IRC"}}]
```


## 5. Serveur HTTP (httpd)

### 5.1 Bases

Image : `httpd:2.4-alpine`. Page d'accueil `httpd/index.html` :

```html
<!DOCTYPE html>
<html>
<head><title>Mon serveur</title></head>
<body><h1>Hello from httpd!</h1></body>
</html>
```

Commandes de vérification : `docker stats`, `docker inspect <conteneur>`, `docker logs <conteneur>`, `curl http://localhost/`.

### 5.2 Configuration

Récupération de la configuration par défaut depuis un conteneur en cours d'exécution :

```bash
docker exec my-httpd cat /usr/local/apache2/conf/httpd.conf > httpd/httpd.conf
# ou : docker cp my-httpd:/usr/local/apache2/conf/httpd.conf httpd/httpd.conf
```

Dockerfile actuel (`httpd/Dockerfile`) :

```dockerfile
FROM httpd:2.4-alpine
COPY httpd.conf /usr/local/apache2/conf/httpd.conf
```

### 5.3 Reverse proxy

Ajouts en fin de `httpd/httpd.conf` :

```apache
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so

<VirtualHost *:80>
    ProxyPreserveHost On
    ProxyPass / http://backend:8080/
    ProxyPassReverse / http://backend:8080/
</VirtualHost>
```

`backend` est le nom du service dans docker-compose (résolu par le DNS interne de Docker).

**Question 1-5 — Pourquoi un reverse proxy ?**

Un reverse proxy se place devant les applications et redirige les requêtes vers le bon service interne. Il apporte : de la sécurité et de l'isolation (le backend n'est pas exposé directement, un seul point d'entrée) ; la terminaison SSL/TLS en un seul endroit ; la répartition de charge entre plusieurs instances du backend ; la possibilité de servir un front-end statique et de mettre en cache ou compresser ; et un point d'entrée unique qui permet de router par chemin ou par domaine et de changer le backend sans impacter les clients.


## 6. docker-compose

`docker-compose.yml` actuel du dépôt (version fonctionnelle) :

```yaml
services:
  backend:
    build:
      context: ./backend
    networks:
      - app-network
    depends_on:
      - database
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://database:5432/db
      - SPRING_DATASOURCE_USERNAME=usr
      - SPRING_DATASOURCE_PASSWORD=pwd
    restart: unless-stopped

  database:
    build:
      context: ./database
    networks:
      - app-network
    environment:
      - POSTGRES_DB=db
      - POSTGRES_USER=usr
      - POSTGRES_PASSWORD=pwd
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped

  httpd:
    build:
      context: ./httpd
    ports:
      - "80:80"
    networks:
      - app-network
    depends_on:
      - backend
    restart: unless-stopped

networks:
  app-network:

volumes:
  pgdata:
```


Démarrage et test :

```bash
cp .env.example .env     # puis remplir les valeurs
docker compose up -d --build
docker compose ps
curl http://localhost/departments/IRC/students
```


**Question 1-6 — Pourquoi docker-compose est-il important ?**

Il décrit toute l'application multi-conteneurs de façon déclarative dans un seul fichier versionnable, et la démarre, l'arrête ou la reconstruit avec une seule commande, au lieu d'enchaîner à la main des docker build / docker run avec leurs options (réseaux, volumes, variables, ports). Il gère les réseaux et la résolution DNS entre services, l'ordre de démarrage, les redémarrages, les volumes et les variables d'environnement. L'environnement devient reproductible et partageable (un docker compose up suffit à obtenir la même pile), ce qui réduit les erreurs humaines et facilite le travail en équipe, les tests et le déploiement.

**Question 1-7 — Commandes docker-compose importantes**

```bash
docker compose up -d --build      # construire et démarrer en arrière-plan
docker compose ps                 # état des services
docker compose logs -f backend    # logs en continu d'un service
docker compose stop / start       # arrêter / redémarrer sans supprimer
docker compose restart backend    # redémarrer un service
docker compose build              # reconstruire les images
docker compose exec database psql -U usr -d db   # commande dans un service
docker compose down               # arrêter et supprimer conteneurs et réseaux
docker compose down -v            # idem + volumes (perte des données)
```

**Question 1-8 — Documentation du fichier docker-compose**

| Élément | Explication |
|---|---|
| `services` | Trois conteneurs : `backend`, `database`, `httpd`. |
| `build.context` | Dossier contenant le Dockerfile de chaque image (un dossier par image). |
| `environment` | Variables injectées à l'exécution. Dans la version recommandée, elles viennent du fichier `.env` (non commité) : aucun secret en dur. |
| `networks` | Version actuelle : un seul réseau `app-network`. Version recommandée : `front-net` (httpd ↔ backend) et `back-net` (backend ↔ database), si bien que httpd ne peut pas atteindre la base (moindre privilège). |
| `ports` | Seul `httpd` publie un port (`80:80`). Backend et base ne sont pas exposés à l'hôte, comme demandé. |
| `depends_on` | Ordre de démarrage base → backend → httpd. Avec `condition: service_healthy`, le backend attend que Postgres accepte réellement des connexions. |
| `healthcheck` | `pg_isready` vérifie que PostgreSQL est prêt. |
| `volumes` | Volume nommé `pgdata` monté sur `/var/lib/postgresql/data` : les données survivent à `docker compose down`. |
| `restart: unless-stopped` | Redémarrage automatique après un crash ou un reboot, sauf arrêt manuel. |



## 7. Publication sur Docker Hub


**Question 1-9 — Commandes de publication et images publiées**

docker login
docker tag my-database champaina/my-database:1.0
docker tag my-backend champaina/my-backend:1.0
docker tag my-httpd champaina/my-httpd:1.0

docker push champaina/my-database:1.0
docker push champaina/my-backend:1.0
docker push champaina/my-httpd:1.0
https://hub.docker.com/r/champaina/my-httpd
https://hub.docker.com/r/champaina/my-backend
https://hub.docker.com/r/champaina/my-database


**Question 1-10 — Pourquoi mettre nos images dans un dépôt en ligne ?**

Les images construites ne sont stockées que sur la machine locale. Un registre permet de les partager avec l'équipe et de les récupérer sur n'importe quelle machine (`docker pull`) sans reconstruire depuis les sources ; de les déployer sur des serveurs de test ou de production, indispensable pour la CI/CD ; de les versionner (tags `1.0`, `1.1`…) pour mettre à jour ou revenir en arrière ; d'en garder une copie centralisée, indépendante du poste d'un développeur ; et de garantir que tout le monde exécute exactement le même artefact. En entreprise, on privilégie souvent un registre privé ou auto-hébergé (confidentialité, contrôle d'accès).