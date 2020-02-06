# Rendu DevOps - CINQUIN Benjamin & JUDE Lilian

# Part 01 - Docker

### Variable d'environnement
Afin d'éviter d'avoir le mot de passe écrit en clair dans le Dockerfile (ce qui serait une mauvaise pratique), il est possible de passer une variable d'environnement à la commande "Docker run".

`docker run -e "POSTGRES_PASSWORD=pwd" pgsql`

### Utilisation d'un volume de données
Il est possible (et conseillé) d'utiliser un volume afin de conserver les données mêmes dans le cas où l'image ou le container sont détruis.
Ainsi la base de données ne sera pas réinitialisée si l'on utilise le flag -v couplé à un dossier contenant des données.

`docker run -v /home/ben/Documents/Docker/tp1/pgsql/datas:/var/lib/postgresql/data pgsql`

### Build Multistage
Contruire une image à l'aide d'un multistage permet de séparer notre build en plusieurs étapes. On peut construire notre image en ciblant un certaine étape de notre image. Pour ce faire, il faut donner un alias à chaque étape. Les alias des étapes peuvent être réutilisés dans la suite du Dockerfile à l'aide de --from. Cela permet de repartir du répertoire courant de l'étape ciblée.

-> Dockerfile

    FROM openjdk:11 AS build
    COPY Main.java /usr/src/
    WORKDIR /usr/src/
    RUN javac Main.java
    
    FROM openjdk:11-jre AS exec
    COPY --from=build /usr/src/Main.class . # On part du répertoire courant de l'étape 'build'
    CMD ["java", "Main"]
    
    docker build --target build -t javac . # Cette commande execute uniquement la première étape du Dockerfile

### Génération de la SpringBoot application
Il est nécessaire d'ajouter en dépendence Spring web afin qu'un serveur Tomcat soit généré au lancement de notre application. C'est grace à ce serveur qu'il est possible de contacter notre API.

### Lancement de notre application SpringBoot
Après avoir adapter les chemins du Dockerfile donné dans le sujet, on peut build un image de notre application. On peut par la suite lancer un container partant de l'image précedemment buildée. Cette application va se lancer sur le port 8080 du conatainer. Pour pouvoir accéder à notre application en localhost, il faut binder le prt 8080 du container à notre port 8080.

**docker run -p 8080:8080 java**

### Récupération de la configuration Apache
En utilisant la commande **docker exec**, il est possible d'exécuter une commande sur un container. Cela va nous servir à récupérer la configuration apache notamment. On pourra ensuite appliquer le reverse proxy donné dans le sujet.

**docker exec apache cat /usr/local/apache2/conf/httpd.conf**

### Application du reverse proxy
Pour appliquer notre reverse proxy, nous allons créer notre propre image pour apache. Ainsi nous allons récupérer la conf httpd de base, la copier à côté de notre  puis la modifier pour ajouter notre reverse proxy. Ainsi dans notre image, nous allons copier notre nouvelle configuration apache à la place de celle présente initialement dans le container.

-> **Reverse Proxy**
ServerName localhost

<VirtualHost *:80>
    ProxyPreserveHost On
    ProxyPass / http ://java:8080/
    ProxyPassReverse / http://java:8080/
</VirtualHost>

-> **Dockerfile**
FROM httpd:2.4.41-alpine

COPY httpd.conf /usr/local/apache2/conf/httpd.conf

### Docker-compose et publication de nos images docker
La mis de en place de notre docker-compose va nous permettre d'installer nos trois images docker précedemment rédigées (postgreSQL, Java et Apache) et de run chacune d'elle dans un container. Cella permet de gagner en rapidité pour la mise en place de notre infrastructure. De plus, cela nous permet de vérifier le bon fonctionnement de nos 3 Dockerfile

-> **docker-compose.yml**

```yaml
version : '3.7'
services :
  java :
    build :
      ./java/spring/simple-api/
    networks :
      - my-net
    depends_on :
      - pgsql

  pgsql :
    build :
      ./pgsql/
    networks :
      - my-net

  apache :
    build :
      ./apache/
    ports :
      - "80:80"
    networks :
      - my-net
    depends_on :
      - pgsql
      - java

networks :
  my-net :
    name: my-net
```

-> **Dockerfile PostgreSQL**

    FROM postgres:11.6-alpine
    
    ENV POSTGRES_DB=db \
        POSTGRES_USER=admin \
        POSTGRES_PASSWORD=admin
    
    COPY *.sql /docker-entrypoint-initdb.d/

-> **Dockerfile Java**

    # Build
    FROM maven:3.6.3-jdk-11 AS myapp-build
    ENV MYAPP_HOME /opt/myapp
    WORKDIR $MYAPP_HOME
    COPY pom.xml .
    RUN mvn dependency:go-offline
    
    COPY src ./src
    RUN mvn package -DskipTests
    
    # Run
    FROM openjdk:11-jre
    ENV MYAPP_HOME /opt/myapp
    WORKDIR $MYAPP_HOME
    COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
    
    EXPOSE 8080
    
    ENTRYPOINT java -jar myapp.jar

-> **Dockerfile Apache**

    FROM httpd:2.4.41-alpine
    
    COPY httpd.conf /usr/local/apache2/conf/httpd.conf

# Part 02 - CI/CD

- Technologies utiles pour ce TP :
	- GitHub : logiciel de gestion de versions
	- Travis CI :  logiciel d'intégration continue (compiler, tester et déployer)
	- Sonarcloud :  logiciel d'analyse et performance de code

- La commande `mvn clean verify` permet d'effectuer des vérifications sur les résultats des tests d'intégration pour s'assurer que les critères de qualité soient respectés.

- Notre premier CI avec notre .travis.yml :

```yaml
language: java

cache:
  directories:
  - "$HOME/.m2/repository"

script: mvn clean verify

```

Le projet est reconnu comme `PASSED`, il est validé.

- Pour la suite, nous créons une nouvelle branche "develop" afin de faire nos tests et vérifications afin de ne pas "gâcher" la branche master.

- Dans le fichier travis.yml il est important de protéger nos variables de connexions car elles peuvent être visibles par d'autres utilisateurs. Afin de les protéger, il est nécessaire de créer des variables d'environnement sécurisés :
` travis encrypt MY_SECRET_ENV=super_secret --add env.global`

- On renseigne les 2 Dockerfiles :

```
FROM openjdk:11-jre
ENTRYPOINT java -jar sample-application-http-api-server.jar
```
```
FROM openjdk:11-jre
ENTRYPOINT java -jar sample-application-db-changelog-job.jar
```
- Publication des images docker dans le DockerHub :

```yaml
language: java

cache:
  directories:
  - "$HOME/.m2/repository"

services:
  - docker

script: mvn clean verify
	- echo "$PASSWORD_DOCKER" | docker login -u "$USERNAME_DOCKER" --password-stdin
	- docker build -t $USERNAME_DOCKER/db-changelog-job ./sample-application-db-changelog-job
	- docker tag $USERNAME_DOCKER/db-changelog-job  $USERNAME_DOCKER/sample-application-db-changelog-job
	- docker push $USERNAME_DOCKER/sample-application-db-changelog-job

	- docker build -t $USERNAME_DOCKER/http-api-server ./sample-application-http-api-server
	- docker tag $USERNAME_DOCKER/http-api-server  $USERNAME_DOCKER/sample-application-http-api-server
	- docker push $USERNAME_DOCKER/sample-application-http-api-server
```

- Mise à jour du pipeline pour utiliser SonarCloud, on ajoute cet élément dans le travis.yml :
```yaml
addons:
  sonarcloud:
    organization: "lilianjude"
    token:
      secure: "$TOKEN_SONARCLOUD" # encrypted value of your token
```
- On modifie aussi le script :
```yaml
script: mvn clean org.jacoco:jacoco-maven-plugin:prepare-agent verify sonar:sonar -Dsonar.projectKey=LilianJude_sample-application-students
```

- Enfin pour séparer le tests des builds, on structure le fichier avec des jobs et stages. Notre fichier va donc ressembler à cela :

```yaml
language: java

cache:
  directories:
  - "$HOME/.m2/repository"

services:
  - docker

addons:
  sonarcloud:
    organization: "lilianjude"
    token:
      secure: "$TOKEN_SONARCLOUD" # encrypted value of your token

jobs:
  include:
    - stage: test
      script: mvn clean org.jacoco:jacoco-maven-plugin:prepare-agent verify sonar:sonar -Dsonar.projectKey=LilianJude_sample-application-students
    - stage: build
      script:
        - echo "$PASSWORD_DOCKER" | docker login -u "$USERNAME_DOCKER" --password-stdin

        - docker build -t $USERNAME_DOCKER/db-changelog-job ./sample-application-db-changelog-job
        - docker tag $USERNAME_DOCKER/db-changelog-job  $USERNAME_DOCKER/sample-application-db-changelog-job
        - docker push $USERNAME_DOCKER/sample-application-db-changelog-job

        - docker build -t $USERNAME_DOCKER/http-api-server ./sample-application-http-api-server
        - docker tag $USERNAME_DOCKER/http-api-server  $USERNAME_DOCKER/sample-application-http-api-server
        - docker push $USERNAME_DOCKER/sample-application-http-api-server
```

# Part 03 - Ansible

### Inventories
Les inventories permettent de définir des paramètres pour certains hôtes seulement

### Main Server Informations
A l'aide de la commande suivante `ansible all -i ansible/inventories/setup.yml -m setup` il est possible de voir toutes les différentes configurations du seveur. Nous avons ainsi accès à notre adresse IP, notre distribution et sa version ou encore les différentes versions des applicatifs installés sur le serveur.

### Exécution d'un playbook
Un playbook exécute toutes les tasks qu'il décrit à tous les hosts qu'il déclare

-> **playbook.yml**

```yaml
- hosts : all # Tous les hosts déclarés dans le fichier /etc/ansible/hosts sont selectionnés
  gather_facts : false
  become : yes
 
  tasks :
    - name : Test connection
      ping : # Exécute ping sur tous les hosts
```

### Playbook avancé
$basearch représente notre architecture de base (pour nous 'x86_64'). Cette variable est utilisable après installation de yum-utils.

Notre playbook avancé va nous permettre d'installer docker sur notre serveur distant.
Si on détaille un peu plus notre playbook :
 1 - On installe yum-utils afin d'utiliser la variable $basearch
 2 - On installe device-mapper-persistent-data et lvm2 (sûrement 2 modules nécessaires pour Docker)
 3 - On créé un repository Docker stable
 4 - On installe Docker
 5 - On s'assure du bon lancement de docker

-> **playbook.ylm**
- hosts : all
  gather_facts : false
  become : yes

  tasks :
  # Install Docker

```yaml
    # Etape 1
    - name : Install yum-utils
      yum :
        name : yum-utils
        state : latest
    # Etape 2
    - name : Install device-mapper-persistent-data
      yum :
        name : device-mapper-persistent-data
        state : latest

    - name : Install lvm2
      yum :
        name : lvm2
        state : latest
    # Etape 3
    - name : Add Docker stable repository
      yum_repository :
        name : docker-ce
        description : Docker CE Stable - $basearch
        baseurl : https://download.docker.com/linux/centos/7/$basearch/stable
        state : present
        enabled : yes
        gpgcheck : yes
        gpgkey : https://download.docker.com/linux/centos/gpg
    # Etape 4
    - name : Install Docker
      yum :
        name : docker-ce
        state : present
    # Etape 5
    - name : Make sure Docker is running
      service : name=docker state=started
      tags : docker
```

