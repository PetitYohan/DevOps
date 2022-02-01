### Par Yohan Petit - 4IRC
# TP01 - Docker

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

1-3 Document docker-compose most important commands
Pour lancer
>docker-compose up

Pour builder
>docker-compose build

1-4 Document your docker-compose file :
```yml
version: '3.3'
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

## Docker hub publish
1-5 Document your publication commands and published images in dockerhub:

Déclaration :
>docker tag tp1_httpd yosmall/tp1_httpd:1.0

Publication :
>docker push yosmall/tp1_httpd:1.0

La base de donnée, l'api java ansi que le serveur http ont été publiés sur docker hub à l'aide des commandes ci-dessus

Possibilité d'executer les commandes docker sur le docker-compose en spécifiant un service comme les logs, start, stop, restart, etc ...

# TP02 - CI/CD-GITHUB-ACTIONS
## First steps into the CI world
2-1 What are testcontainers?
>Testcontainer est une librairie java qui permet d'instancier dans des containers des systèmes.

2-2 Document your Github Actions configurations:
```yml
name: CI devops 2022 CPE
on:
  #to begin you want to launch this job in main and develop
  push:
    # Choit de la branch git
    branches: [master]
  pull_request:


jobs:
  test-backend:
    # Choit de l'OS
    runs-on: ubuntu-18.04
    steps:
      #checkout your github code using actions/checkout@v2.3.3
      - uses: actions/checkout@v2.3.3
      #do the same with another action (actions/setup-java@v2) that enable to setup jdk 11
      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          #Choit de la version de java
          java-version: 11
          #Choit de la distribution du djk
          distribution: 'adopt'
      #finally build your app with the latest command
      - name: Build and test with Maven
        #Exécution des tests sur le projet maven
        run: mvn clean verify --file ./TP1/Backend/simple-api/pom.xml
```
