# Docker compose: orquestación y escalado

1. [Crear una orquestación compleja con imágenes independientes](#new-orquestations)
2. [Ejecutar, administrar y probar la orquestación](#test-orquestations)
3. [Escalar servicios con docker-compose (múltiples instancias)](#scale)
4. [Redes virtuales en docker y múltiples entornos](#vpn)

<hr>

<a name="new-orquestation"></a>

## 1. Crear una orquestación compleja con imágenes independientes

Descargamos el repositorio con los servicios diferenciados. En este caso tendremos un dockerfile por cada servicio:

- Para la aplicación ángular:

~~~dockerfile
FROM nginx:alpine
 # use a volume is more efficient and speed that filesystem
VOLUME /tmp
RUN rm -rf /usr/share/nginx/html/*
COPY nginx.conf /etc/nginx/nginx.conf
COPY billingApp /usr/share/nginx/html
#expose app and 80 for nginx app
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
~~~
**RUN** se usa básicamente para correr un comando, ya sea instalar un servicio, eliminar ficheros, crear directorios. Se puede ejecutar las veces que lo necesitemos dentro del dockerfile. **CMD** se utiliza para ejecutar un comando por defecto, cuando se crea el contenedor

- Para el microservicio java:

~~~dockerfile
FROM openjdk:8-jdk-alpine
RUN addgroup -S devopsc && adduser -S javams -G devopsc
USER javams:devopsc
ENV JAVA_OPTS=""
ARG JAR_FILE
ADD ${JAR_FILE} app.jar
 # use a volume is mor efficient and speed that filesystem
VOLUME /tmp
EXPOSE 8080
ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar /app.jar" ]
~~~

> Por seguridad, todos los contenedores deberían tener una configuración de grupo/usuario.

- Para la base de datos, tenemos un script que crea la base de datos por defecto sobre un contenedor de **Postgres** de docker-hub.

~~~sql
#!/bin/bash
set -e

psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
    CREATE USER billingapp WITH PASSWORD 'qwerty';
    CREATE DATABASE billingapp_db;
    GRANT ALL PRIVILEGES ON DATABASE billingapp_db TO billingapp;
EOSQL
~~~

- Por último tenemos  el archivo de configuración `stack-billling.yml` que usaremos en el docker compose con que definimos los cuatro servicios que vamos a utilizar:
  - Postgres
  - Adminer
  - App back
  - App front

~~~yaml
version: '3.1'

services:
#database engine service
  postgres_db:
    container_name: postgres
    image: postgres:latest
    restart: always
    environment:
    ports:
      - 5432:5432
    volumes:
        #allow *.sql, *.sql.gz, or *.sh and is execute only if data directory is empty
      - ./dbfiles:/docker-entrypoint-initdb.d
      - /var/lib/postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: qwerty
      POSTGRES_DB: postgres    
#database admin service
  adminer:
    container_name: adminer
    image: adminer
    restart: always
    depends_on: 
      - postgres_db
    ports:
       - 9090:8080
#Billin app backend service
  billingapp-back:
    build:
      context: ./java
      args:
        - JAR_FILE=*.jar
    container_name: billingApp-back      
    environment:
       - JAVA_OPTS=
         -Xms256M 
         -Xmx256M         
    depends_on:     
      - postgres_db
    ports:
      - 8080:8080 
#Billin app frontend service
  billingapp-front:
    build:
      context: ./angular 
    container_name: billingApp-front
    depends_on:     
      - billingapp-back
    ports:
      - 80:80 
~~~
En este caso para Postgres y Adminer no vamos a utilizar una imagen personalizada, sino que usaremos la imagen que proporciona docker-hub.

En la definción de postgres, en la etiqueta **volumes** indicamos el punto de entrada de la BBDD y establecemos un volumen para poder compartir datos entre la máquina host y el contenedor, de forma que queden persistentes si eliminamos el contenedor.

Para las aplicaciones back y front indicamos en el yaml el docker file desde donde tiene que construir la imagen.

