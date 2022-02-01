# TP01 - Docker
### Par Yohan Petit - 4IRC

_newgrp docker => pour lancer docker sur session CPE_

## Database
### Port 5432:5432 ADMINER 8081:8080

1-1 Document your database container essentials: commands and
Dockerfile

Build : 
>docker build -t yohanp/database .

Network : 
>docker network create app-network

Run database : 
>docker run -d -v /temp/data:/var/lib/postgresql/data --network=app-network --name=database yohanp/database

Run adminer : 
>docker run -d -p 8081:8080 --network=app-network --name=adminer adminer

Dockerfile :
```docker
FROM postgres:11.6-alpine
COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d
COPY 02-InsertData.sql /docker-entrypoint-initdb.d
ENV POSTGRES_DB=db \
    POSTGRES_USER=usr \
    POSTGRES_PASSWORD=pwd
```

## Backend
### Port 8080:8080

Build :
>docker build -t yohanp/backend .

Run :
>docker run -d --network=app-network --name=backend yohanp/backend

Dockerfile :
```dockerfile
FROM openjdk:16-alpine3.13
COPY Main.class /prog/
CMD cd /prog ; java Main
```

1-2 Why do we need a multistage build ? And explain each steps of
this dockerfile
>Pour d'abord build le package maven (jdk) et ensuite l'exécuter le package .jar avec (jre qui est plus léger)
```dockerfile
# Build
# Séléction de la version maven
FROM maven:3.6.3-jdk-11 AS myapp-build
# Récupération et assigne le chemin de ma variable d'environnement
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
# Copie les fichiers de configuration du projet
COPY backend-api/pom.xml .
COPY backend-api/src ./src
# Packaging du projet sans les tests
CMD mvn dependency:go-offline
RUN mvn package -DskipTests

# Run
# Séléction de la version de java
FROM openjdk:11-jre
# Récupération et assigne le chemin de ma variable d'environnement
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
# Copie du package au bon endroit
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
# Exécution du package .jar pour lancer le backend
ENTRYPOINT java -jar myapp.jar
```

## HttpServer
### Port 80:80
Build :
>docker build -t yohanp/httpserver .

Run :
>docker run -d -p 80:80 --network=app-network --name=httpserver yohanp/httpserver

Dockerfile :
```dockerfile
FROM httpd:2.4
COPY index.html /usr/local/apache2/htdocs/
COPY ./my-httpd.conf /usr/local/apache2/conf/httpd.conf
```

modifier la configuration : 
>docker run --rm httpd:2.4 cat /usr/local/apache2/conf/httpd.conf > my-httpd.conf

## Docker-compose

Commande : 
>docker-compose up

docker-compose.yml :
```yml
version: '3.7'
services:
  backend:
    build: ./Backend
    networks: 
      - app-network
    depends_on:
      - database
  database:
    build: ./Database
    networks:
      - app-network
  httpd:
    build: ./HttpServer
    ports: 
      - "80:80"
    networks:
      - app-network
    depends_on: 
      - database
      - backend
networks:
  app-network:
```