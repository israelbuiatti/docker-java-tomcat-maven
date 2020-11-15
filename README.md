
A ideia é que eu possa usar esta solução em qualquer projeto que utilize "Maven + Tomcat" colocando os seguintes arquivos na raiz do projeto seguindo os passos abaixo.

No exemplo utilizei o projeto que segue disponível no github: https://github.com/rogeriofonseca/docker-java-tomcat-maven

1 - Estrutura de arquivos para configuração do Docker

docker-compose.yml
dockerfiles/
├── Dockerfile-apache
├── Dockerfile-maven
├── Dockerfile-mysql
└── tomcat-users.xml
scripts/
└── entrypointscript.sh

2 - docker-compose.yml
```
version: '2'
services:
  db:
    image: mysql:5.7
    environment:
      - MYSQL_ROOT_PASSWORD=root
      - MYSQL_DATABASE=projeto_jpa
    ports:
      - "3306:3306"
  maven:
      build:
        context: dockerfiles
        dockerfile: Dockerfile-maven
      volumes:
        - ~/.m2:/root/.m2
        - $PWD:/usr/src/mymaven
      volumes_from:
        - tomcat
  tomcat:
    build:
      context: dockerfiles
      dockerfile: Dockerfile-apache
    ports:
      - "8888:8080"
```

2.1 Dockerfile-apache
```
FROM tomcat:8.0

ADD . /code
WORKDIR /code
COPY tomcat-users.xml  $CATALINA_HOME/conf/
VOLUME $CATALINA_HOME/webapps
```
2.2 Dockerfile-maven
```
FROM openjdk:7-jdk

ARG MAVEN_VERSION=3.3.9
ARG USER_HOME_DIR="/root"

RUN mkdir -p /usr/share/maven /usr/share/maven/ref \
  && curl -fsSL http://apache.osuosl.org/maven/maven-3/$MAVEN_VERSION/binaries/apache-maven-$MAVEN_VERSION-bin.tar.gz \
    | tar -xzC /usr/share/maven --strip-components=1 \
  && ln -s /usr/share/maven/bin/mvn /usr/bin/mvn

ENV MAVEN_HOME /usr/share/maven
ENV MAVEN_CONFIG "$USER_HOME_DIR/.m2"

VOLUME "$USER_HOME_DIR/.m2"
WORKDIR /usr/src/mymaven
ENTRYPOINT ["/usr/src/mymaven/scripts/entrypointscript.sh"]
```
2.3 Dockerfile-mysql
```
FROM mysql:5.7

ENV MYSQL_ROOT_PASSWORD=root
ENV MYSQL_DATABASE=projeto_jpa

expose 3306:3306
```
2.4 Um ponto de atenção para este aqui. Ele é responsável por compilar e copiar todos artefatos para o tomcat.
No meu caso tenho um projeto que tem vários módulos, então o resultado final terá mais de um .war, por este motivo o "find" buscando em todo diretório e sub-diretórios.

Arquivo: scripts/entrypoint.sh
```
#!/bin/bash
mvn clean install -e -f  /usr/src/mymaven  &&
find /usr/src/mymaven/target -name "*.war" -exec cp '{}' /usr/local/tomcat/webapps \;
```
2.5 tomcat-users.xml
```
<?xml version='1.0' encoding='utf-8'?>
<tomcat-users>
  <role rolename="manager"/>
  <role rolename="admin"/>
  <user username="tomcat" password="tomcat" roles="admin,manager, manager-gui"/>
</tomcat-users>
```
# Comandos úteis:

1 - Para subir os containers basta executar na raiz do projeto o comando:

$ docker-compose build --no-cache && docker-compose up -d

2 - Para recompilar após efetuar alguma alteração no código Java basta executar apenas o container específico do maven. Ele fará o papel de recompilar o código e copiar os artefatos para o volume compartilhado entre os dois containers.

$ docker-compose run maven

Ps.: O próximo passo será adequar este fluxo no meu ambiente de desenvolvimento integrando o fluxo com a minha IDE. Mas isto é pauta para outro tópico.
