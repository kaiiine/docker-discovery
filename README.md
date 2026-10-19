# TP 01 - Docker

Application 3-tiers : **httpd** (reverse proxy) → **backend** (Spring Boot) → **database** (PostgreSQL), orchestrée avec Docker Compose.

## Structure

```
.
├── db/              # Dockerfile Postgres + scripts SQL d'init
├── simpleapi/       # Backend Spring Boot + Dockerfile multistage
├── http-server/     # Dockerfile httpd + httpd.conf + index.html
├── docker-compose.yaml
├── .env             # secrets (non versionné)
└── .env.example     # modèle
```

---

## Database

### 1-1 Pourquoi `-e` plutôt que mettre les variables dans le Dockerfile ?

Les variables d'un `ENV` du Dockerfile sont écrites **en clair dans l'image** : toute personne qui a l'image (ou le dépôt Git) peut les lire avec `docker history` ou `docker inspect`. Avec `-e`, elles sont injectées **au lancement** du conteneur : l'image reste générique, réutilisable, et on peut changer les identifiants selon l'environnement (dev, prod) sans reconstruire.

### 1-2 Pourquoi un volume pour Postgres ?

Le système de fichiers d'un conteneur est éphémère : une fois le conteneur supprimé, ses données disparaissent. Le volume stocke `/var/lib/postgresql/data` **en dehors du conteneur**, donc les données survivent à sa suppression ou à sa recréation (mise à jour d'image, crash). Il peut aussi être sauvegardé et réattaché à un nouveau conteneur.

### 1-3 Essentiels du conteneur database

`db/Dockerfile` :

```dockerfile
FROM postgres:17.2-alpine

COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d/
COPY 02-InsertData.sql /docker-entrypoint-initdb.d/
```

Les scripts de `/docker-entrypoint-initdb.d/` sont exécutés par ordre alphabétique **au premier démarrage** (base vide uniquement).

Commandes :

```bash
docker network create app-network

docker build -t my-database ./db

docker run -d --name database --network app-network \
  -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd \
  -v pgdata:/var/lib/postgresql/data \
  my-database

docker run -d --name adminer --network app-network -p 8090:8080 adminer
```

Vérification : `docker logs database` (scripts exécutés), puis Adminer sur `http://localhost:8090` (serveur `database`, user `usr`, base `db`).

---

## Backend API

### 1-4 Pourquoi un multistage build ? Explication du Dockerfile

Le build nécessite un JDK + Maven (lourds), mais l'exécution n'a besoin que d'un JRE et du `.jar`. Le multistage permet de **compiler dans une étape et de ne garder que le résultat dans l'image finale** : image plus légère, surface d'attaque réduite (pas d'outils de build ni de code source), et build reproductible sans rien installer sur la machine.

```dockerfile
# Build stage
FROM eclipse-temurin:21-jdk-alpine AS myapp-build   # image avec JDK, étape nommée "myapp-build"
ENV MYAPP_HOME=/opt/myapp                           # variable pour le dossier de travail
WORKDIR $MYAPP_HOME                                 # dossier courant pour les commandes suivantes

RUN apk add --no-cache maven                        # installe Maven dans l'image de build

COPY pom.xml .                                      # copie le pom seul (permet de profiter du cache Docker)
COPY src ./src                                      # copie le code source
RUN mvn package -DskipTests                         # compile et produit target/*.jar sans lancer les tests

# Run stage
FROM eclipse-temurin:21-jre-alpine                  # nouvelle image, JRE seul (plus léger)
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar   # récupère uniquement le jar de l'étape de build

ENTRYPOINT ["java", "-jar", "myapp.jar"]            # commande de démarrage de l'application
```

Connexion à la base : l'URL JDBC utilise le **nom du service** comme hôte (`jdbc:postgresql://database:5432/db`), car `localhost` désigne le conteneur lui-même. Elle est surchargée par les variables `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME` et `SPRING_DATASOURCE_PASSWORD`.

---

## Http server

### 1-5 Pourquoi un reverse proxy ?

C'est un serveur placé devant le backend : les clients ne parlent qu'à lui et il relaie vers l'application. Il permet :

- **Sécurité** : le backend n'est pas exposé (aucun port publié), un seul point d'entrée.
- **Point d'entrée unique** : un seul domaine/port pour plusieurs services.
- **SSL/TLS** géré à un seul endroit.
- **Répartition de charge** entre plusieurs instances, cache, compression.
- Possibilité de changer le backend sans impact pour le client.

`http-server/Dockerfile` :

```dockerfile
FROM httpd:2.4
COPY index.html /usr/local/apache2/htdocs/index.html
COPY httpd.conf /usr/local/apache2/conf/httpd.conf
```

La config par défaut a été récupérée avec `docker cp web:/usr/local/apache2/conf/httpd.conf ./httpd.conf`, puis complétée :

```apache
ServerName localhost

LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so

<VirtualHost *:80>
    ProxyPreserveHost On
    ProxyPass / http://${BACKEND_HOST}:${BACKEND_PORT}/
    ProxyPassReverse / http://${BACKEND_HOST}:${BACKEND_PORT}/
</VirtualHost>
```

`BACKEND_HOST` et `BACKEND_PORT` sont fournies par la section `environment` du service dans le compose.

---

## Docker Compose

### 1-6 Pourquoi docker-compose est important ?

Il décrit toute l'application (services, réseaux, volumes, variables) dans **un seul fichier versionné** et la pilote avec une commande. Cela évite de lancer et enchaîner à la main plusieurs `docker run` avec leurs options, garantit un environnement **reproductible** pour toute l'équipe, gère l'ordre de démarrage et les dépendances, et simplifie le build, l'arrêt et le nettoyage.

### 1-7 Commandes principales

```bash
docker compose up -d --build     # construit et démarre en arrière-plan
docker compose ps                # état des services
docker compose logs -f backend   # logs d'un service en direct
docker compose exec database psql -U usr -d db   # commande dans un service
docker compose restart backend   # redémarre un service
docker compose stop              # arrête sans supprimer
docker compose down              # arrête et supprime conteneurs + réseaux
docker compose down -v           # idem + volumes (données perdues)
docker compose config            # affiche le compose avec les variables résolues
```

### 1-8 Fichier docker-compose

```yaml
services:
  backend:
    build: ./simpleapi
    container_name: backend
    environment:
      SPRING_DATASOURCE_URL: ${SPRING_DATASOURCE_URL}
      SPRING_DATASOURCE_USERNAME: ${POSTGRES_USER}
      SPRING_DATASOURCE_PASSWORD: ${POSTGRES_PASSWORD}
    networks:
      - front-net
      - back-net
    depends_on:
      database:
        condition: service_healthy
    restart: unless-stopped

  database:
    build: ./db
    container_name: database
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - back-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 10
    restart: unless-stopped

  httpd:
    build: ./http-server
    container_name: httpd
    environment:
      BACKEND_HOST: ${BACKEND_HOST}
      BACKEND_PORT: ${BACKEND_PORT}
    ports:
      - "80:80"
    networks:
      - front-net
    depends_on:
      - backend
    restart: unless-stopped

  adminer:
    image: adminer
    container_name: adminer
    ports:
      - "8090:8080"
    networks:
      - back-net
    depends_on:
      - database
    restart: unless-stopped

networks:
  front-net:
  back-net:

volumes:
  pgdata:
```

Choix de conception :

- **Variables** dans un `.env` non versionné (modèle fourni dans `.env.example`) : aucun mot de passe dans les fichiers commités.
- **Ports** : seul httpd est publié sur l'hôte. Backend et database ne sont joignables que depuis le réseau Docker.
- **Deux réseaux** (ségrégation, moindre privilège) : `front-net` relie httpd et backend, `back-net` relie backend, database et adminer. httpd, le service exposé, ne peut pas atteindre la base. Le backend fait le pont.
- **Healthcheck + `service_healthy`** : le backend ne démarre qu'une fois Postgres prêt à accepter des connexions.
- **Volume nommé `pgdata`** : persistance des données.
- **`restart: unless-stopped`** : redémarrage automatique après un crash ou un reboot.
- **Adminer** est un outil de développement : à ne pas exposer en production.

Test : `curl http://localhost/departments/IRC/students` répond via httpd, alors que `localhost:8080` est inaccessible (backend non publié).

---

## Publish

### 1-9 Commandes de publication

```bash
docker login

docker tag my-database kaine/my-database:1.0
docker tag my-backend  kaine/simpleapi:1.0
docker tag my-httpd    kaine/my-httpd:1.0

docker push kaine/my-database:1.0
docker push kaine/simpleapi:1.0
docker push kaine/my-httpd:1.0
```

Images publiées sur Docker Hub (compte `kaine`) : `kaine/my-database:1.0`, `kaine/simpleapi:1.0`, `kaine/my-httpd:1.0`. Le tag de version (`1.0`) remplace `latest` pour savoir précisément quelle version est déployée et pouvoir revenir en arrière.

### 1-10 Pourquoi mettre les images dans un dépôt en ligne ?

- **Partage** : les autres membres de l'équipe et les serveurs peuvent récupérer l'image avec `docker pull`, sans la reconstruire.
- **Déploiement** : les environnements (CI/CD, serveurs de production, cloud) tirent les images depuis un registre central.
- **Versionnage** : chaque version est taguée, ce qui permet les rollbacks et la traçabilité.
- **Reproductibilité** : tout le monde exécute exactement la même image.
- **Sauvegarde** : l'image existe en dehors de la machine locale.

En entreprise, on utilise généralement un registre privé auto-hébergé plutôt que Docker Hub public.
