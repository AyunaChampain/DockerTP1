# TP DevOps — Docker, CI/CD et Ansible

Application web 3-tiers (httpd → backend Spring Boot → PostgreSQL), plus un front Vue.js au TP3.

```
Client ──:80──▶ httpd (reverse proxy) ──/api/──▶ backend (:8080) ──▶ database (:5432)
                                       └──/────▶ front (nginx :80)
```

Seul `httpd` est exposé sur l'hôte (port 80). Un dossier par image : `database/`, `backend/`, `httpd/`, `front/`, plus `ansible/`, `sandbox/` (tests Java) et `.github/workflows/main.yml`.

---

## TP1 — Docker

### Base de données

```dockerfile
FROM postgres:17.2-alpine
COPY init/ /docker-entrypoint-initdb.d/
```

```bash
docker network create app-network
docker build -t my-database ./database
docker run -d --name my-database --network app-network \
  -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd \
  -v "$(pwd)/data:/var/lib/postgresql/data" my-database
docker run -d -p 8090:8080 --net=app-network --name=adminer adminer
```

**1-1 — Pourquoi `-e` plutôt que `ENV` dans le Dockerfile ?**
Les valeurs d'un Dockerfile sont figées dans l'image : quiconque l'obtient peut lire le mot de passe (`docker history`, `docker inspect`), et changer une valeur oblige à reconstruire. Avec `-e`, l'image reste générique et la configuration est injectée au lancement, différente selon l'environnement, sans être commitée.

**1-2 — Pourquoi un volume pour Postgres ?**
Le système de fichiers d'un conteneur disparaît avec lui. Le volume stocke les données hors du conteneur : on peut le détruire, le recréer ou le mettre à jour sans rien perdre.

**1-3 — Essentiels du conteneur base de données** : voir le Dockerfile et les commandes ci-dessus. Les scripts `01-CreateScheme.sql` et `02-InsertData.sql` sont exécutés au premier démarrage, dans l'ordre alphabétique.

### Backend

Dockerfile multistage (`backend/Dockerfile`) :

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS myapp-build
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
RUN apk add --no-cache maven
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

FROM eclipse-temurin:21-jre-alpine
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

**1-4 — Pourquoi un multistage ? Explication du Dockerfile**
Compiler demande un JDK, Maven et les sources ; exécuter ne demande qu'un JRE et le `.jar`. Le multistage compile dans Docker mais ne livre que le nécessaire : image plus légère, moins de paquets donc moins de failles, build reproductible sans rien installer en local.

- **Étape 1 (build)** : JDK 21 + Maven. `COPY pom.xml` puis `mvn dependency:go-offline` télécharge les dépendances dans une couche mise en cache (refaite seulement si le `pom.xml` change). `COPY src` puis `mvn package` produit le `.jar`.
- **Étape 2 (run)** : JRE seul. `COPY --from=myapp-build` ne récupère que le `.jar` (le reste de l'étape 1 est abandonné). `ENTRYPOINT` lance l'application.

Le backend joint la base par le nom du service (`jdbc:postgresql://database:5432/db`), pas par `localhost`. Les identifiants sont surchargés par `SPRING_DATASOURCE_URL/USERNAME/PASSWORD`.

### httpd (reverse proxy)

```dockerfile
FROM httpd:2.4-alpine
COPY httpd.conf /usr/local/apache2/conf/httpd.conf
COPY index.html /usr/local/apache2/htdocs/
```

La configuration par défaut a été récupérée avec `docker exec` / `docker cp`, puis complétée avec les modules `proxy` et `proxy_http` et :

```apache
<VirtualHost *:80>
    ProxyPreserveHost On
    ProxyPass        /api/ http://backend:8080/
    ProxyPassReverse /api/ http://backend:8080/
    ProxyPass        /     http://front:80/
    ProxyPassReverse /     http://front:80/
</VirtualHost>
```

**1-5 — Pourquoi un reverse proxy ?**
Il offre un point d'entrée unique devant les services internes, qui ne sont plus exposés directement (sécurité). Il permet aussi la terminaison SSL en un seul endroit, la répartition de charge, la mise en cache/compression, et le routage par chemin ou domaine (ici `/api/` vers le backend, le reste vers le front) sans impact sur les clients.

### docker-compose

Quatre services (`front`, `backend`, `database`, `httpd`) sur le réseau `app-network`, volume `pgdata`, `restart: unless-stopped`.

```bash
docker compose up -d --build
curl http://localhost/api/departments/IRC/students
```

**1-6 — Pourquoi docker-compose ?**
Il décrit toute l'application multi-conteneurs dans un fichier versionnable et la démarre, l'arrête ou la reconstruit en une commande, au lieu d'enchaîner à la main des `docker build`/`run` avec leurs réseaux, volumes, variables et ports. L'environnement devient reproductible et partageable.

**1-7 — Commandes principales**

```bash
docker compose up -d --build     # construire et démarrer
docker compose ps                # état des services
docker compose logs -f backend   # logs d'un service
docker compose restart backend   # redémarrer un service
docker compose down              # arrêter et supprimer conteneurs et réseau
docker compose down -v           # idem + volumes (perte des données)
```

**1-8 — Documentation du fichier docker-compose**

| Élément | Explication |
|---|---|
| `build.context` | Dossier du Dockerfile de chaque service. |
| `environment` | `POSTGRES_*` (base) et `SPRING_DATASOURCE_*` (backend, surcharge `application.yml`). |
| `networks` | Réseau `app-network` : les services se joignent par leur nom. |
| `ports` | Seul `httpd` publie `80:80` ; backend, front et base restent internes. |
| `depends_on` | Ordre de démarrage `database` → `backend` → `httpd`. |
| `volumes` | `pgdata` sur `/var/lib/postgresql/data` : les données survivent à `down`. |
| `restart` | `unless-stopped` : redémarrage après crash ou reboot. |

### Publication

**1-9 — Commandes et images publiées**

```bash
docker login
docker tag my-database champaina/my-database:1.0
docker tag my-backend  champaina/my-backend:1.0
docker tag my-httpd    champaina/my-httpd:1.0
docker push champaina/my-database:1.0
docker push champaina/my-backend:1.0
docker push champaina/my-httpd:1.0
```

Images : <https://hub.docker.com/r/champaina/my-database>, <https://hub.docker.com/r/champaina/my-backend>, <https://hub.docker.com/r/champaina/my-httpd>. Au TP2, la CI publie `champaina/tp-devops-{backend,database,httpd,front}:latest`.

**1-10 — Pourquoi un dépôt en ligne ?**
Pour partager les images et les récupérer sur n'importe quelle machine (`docker pull`) sans reconstruire, les déployer sur des serveurs (indispensable à la CI/CD), les versionner (`1.0`, `1.1`…) pour mettre à jour ou revenir en arrière, et garantir que tout le monde exécute le même artefact.

---

## TP2 — CI/CD (GitHub Actions)

`.github/workflows/main.yml`, déclenché sur `push` (`master`, `develop`) et `pull_request`. Jobs actifs : `test-backend` (build, tests, Sonar), puis `build-and-push-backend/database/httpd` (publication sur `master`, `needs: test-backend`). Les jobs du TP3 (`build-and-push-front`, `deploy`, `deploy-backend-ansible`) sont conservés en commentaire.

```yaml
- name: Build, test and analyze with Sonar
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
  run: >
    mvn -B clean verify
    org.sonarsource.scanner.maven:sonar-maven-plugin:sonar
    -Dsonar.projectKey=AyunaChampain_DockerTP1
    -Dsonar.organization=ayunachampain
    -Dsonar.host.url=https://sonarcloud.io
    --file backend/pom.xml
```

**2-1 — Que sont les testcontainers ?**
Des bibliothèques Java qui lancent de vrais conteneurs Docker pendant les tests, puis les détruisent. Ici, une base PostgreSQL jetable (`jdbc:tc:postgresql://...` dans `src/test/resources/application.yml`) : les tests d'intégration tournent sur une vraie base, isolée et propre à chaque exécution, sans rien installer.

**2-2 — Pourquoi des variables sécurisées (secrets) ?**
Un identifiant écrit dans le dépôt est lisible par tous et reste dans l'historique git. Les secrets sont stockés chiffrés, injectés à l'exécution, masqués dans les logs, et révocables sans toucher au code. Utilisés : `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN` (access token en écriture), `SONAR_TOKEN`.

**2-3 — Pourquoi `needs: test-backend` ?**
Sans `needs`, les jobs s'exécutent en parallèle : une image pourrait être publiée alors que les tests échouent. Avec `needs`, la publication n'a lieu que si les tests réussissent.

**2-4 — Pourquoi pousser les images ?**
Le runner est détruit à la fin du workflow : une image non poussée est perdue. Le registre la conserve, versionnée, et permet au serveur de la récupérer par un simple `docker pull` pour le déploiement.

**Quality gate (SonarCloud).** Sonar analyse le code sans l'exécuter et signale bugs, vulnérabilités et « code smells » ; le quality gate est un ensemble de seuils, « Passed » ou « Failed », qui empêche de laisser passer du code de mauvaise qualité. Le plugin est appelé par son nom complet (`sonar:sonar` seul échoue, le plugin n'étant pas déclaré dans le `pom.xml`), et `fetch-depth: 0` fournit l'historique git complet.

> 📷 À ajouter : capture du rapport SonarCloud et du pipeline vert.

---

## TP3 — Ansible

Inventaire `ansible/inventories/setup.yml` :

```yaml
all:
 vars:
   ansible_user: admin
   ansible_ssh_private_key_file: ~/.ssh/formation_key
 children:
   prod:
     hosts: ayuna.champain.takima.school
```

**3-1 — Inventaire et commandes de base**

```bash
ansible all -i inventories/setup.yml -m ping
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=absent" --become
```

L'inventaire liste les hôtes et les regroupe (`prod`) ; `vars` donne l'utilisateur et la clé SSH. `setup` renvoie les *facts* (infos découvertes sur l'hôte, préfixées `ansible_`). Ansible est déclaratif et idempotent : relancer la suppression d'Apache2 ne change plus rien.

**3-2 — Documentation du playbook**

```yaml
- hosts: all
  gather_facts: true
  become: true
  roles:
    - { role: docker,   tags: docker }
    - { role: network,  tags: network }
    - { role: database, tags: database }
    - { role: app,      tags: app }
    - { role: front,    tags: front }
    - { role: proxy,    tags: proxy }
```

```bash
ansible-playbook -i inventories/setup.yml playbook.yml --syntax-check
ansible-playbook -i inventories/setup.yml playbook.yml
```

Le playbook s'exécute en root sur tous les hôtes et appelle six rôles dans l'ordre. Le rôle `docker` suit la documentation Docker pour Debian : paquets prérequis, clé GPG et dépôt APT, installation de `docker-ce`, Python, création d'un venv `/opt/docker_venv` avec le SDK `docker`, puis démarrage du service.

**3-3 — Tâches `docker_container`**

```yaml
- name: Run backend API
  community.docker.docker_container:
    name: backend
    image: "{{ dockerhub_user }}/tp-devops-backend:latest"
    pull: always
    state: started
    restart_policy: unless-stopped
    networks:
      - name: "{{ docker_network_name }}"
    env:
      SPRING_DATASOURCE_URL: "jdbc:postgresql://database:5432/{{ db_name }}"
      SPRING_DATASOURCE_USERNAME: "{{ db_user }}"
      SPRING_DATASOURCE_PASSWORD: "{{ db_password }}"
  vars:
    ansible_python_interpreter: /opt/docker_venv/bin/python
```

- `name` sert de nom DNS dans le réseau ; `pull: always` récupère la dernière image ; `state: started` est idempotent ; `restart_policy` relance après crash.
- `ansible_python_interpreter` pointe vers le venv qui contient le SDK Docker.
- `database` ajoute `volumes: pgdata:/var/lib/postgresql/data` ; seul `proxy` publie `80:80` ; `network` crée `app-network` avec `docker_network`.
- Les variables viennent de `group_vars/all/vars.yml` ; le mot de passe (`vault_db_password`) est stocké chiffré dans `vault.yml` avec **Ansible Vault** (`.vault_pass` est ignoré par git).

**3-4 — Est-il sûr de déployer automatiquement chaque nouvelle image ?**
Pas totalement. `latest` est mutable : toute image poussée (erreur, token Docker Hub compromis) est déployée telle quelle, sans version identifiable ni retour arrière facile. Peu de contrôles (seul le backend est testé), aucune validation humaine, aucun scan de vulnérabilités. Pour sécuriser : tags immuables (version, SHA du commit ou digest), tests obligatoires avant déploiement et protection de la branche `master`, approbation manuelle via un *environment* GitHub, environnement de préproduction avec contrôle de santé après déploiement, scan des images, et droits minimaux pour le token et l'utilisateur SSH.

**Front et déploiement continu.** Le front (Vue.js) est construit par un Dockerfile multistage (Node puis nginx) ; httpd route `/` vers lui et `/api/` vers le backend. Le déploiement continu repose sur les jobs `build-and-push-front`, `deploy` (playbook complet) et `deploy-backend-ansible` (image Docker « ansible-runner » + vault, tag `app` uniquement), actuellement en commentaire dans `main.yml`.
