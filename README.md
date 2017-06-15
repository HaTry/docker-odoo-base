# [Docker Odoo Base Image](https://hub.docker.com/r/tecnativa/odoo-base)
Docker Odoo  Ingetive utilizando la realizada por Tecnativa
 
## Objetivo
 
Disponer de una base sobre Docker que posibilite el despliegue agil y rapido de Odoo.
Quedando recogidas un conjunto de buenas practicas y herramientas para el desarrollo sobre Odoo.
 
## Estructura
 
Todo queda dentro de `/opt/odoo`.
 
    custom/
        entrypoint.d/
        build.d/
        conf.d/
        src/
            private/
            odoo/
            addons.yaml
            repos.yaml
            requirements.txt
    common/
        entrypoint.sh
        build.sh
        entrypoint.d/
        build.d/
        conf.d/
    auto
        addons/
        odoo.conf
 
 
### `/opt/odoo/custom`
Aqui esta todo lo relacionado con el proyecto
#### `/opt/odoo/custom/entrypoint.d`
Los ejecutables que aqui se encuentren seran ejecutados al lanzar el contenedor, antes del comando solicitado.
#### `/opt/odoo/custom/build.d`
Los ejecutables que aqui se encuentren seran ejecutados al lanzar el contenedor.
#### `/opt/odoo/custom/conf.d`
Los ficheros que aqui se encuentren seran expandidos y concatenados en `/opt/odoo/auto/odoo.conf`.
#### `/opt/odoo/custom/src`
Aqui estará el código. Este puede ser insertado de 2 formas:
 
* Usando `repos.yaml`, que lo añadirá automaticamente al crear el contenedor.
* De forma manual, copiandolo directamente.
 
##### `/opt/odoo/custom/src/respos.yaml`
Se usará `repos.yaml` para indicar todo que hay que añadir excepto `private`. 
Todo lo que añadamos menos `private` debe ser excluido de los repositorios; por tanto, en `.gitignore` se añaden las siguiente reglas:
```sh
odoo/custom/src/*
!odoo/custom/src/private
!odoo/custom/src/*.*
```
##### `/opt/odoo/custom/src/addons.yaml`
Mediante `addons.yaml` , que es un fichero de configuración de [git-aggregator](https://github.com/acsone/git-aggregator), indicaremos los addons visibles, una entrada por repositorio y addon.
```yaml
website:
    - website_cookie_notice
    - website_legal_page
web:
    - web_responsive
```
Podremos unir varios, utilizando `---`:
 
```yaml
# Spanish Localization
l10n-spain:
    - l10n_es
server-tools:
    - date_range
---
# SEO tools
website:
    - website_blog_excertp_img
server-tools: # Repetimos server-tools, no hay problema,  es un
              # documento diferente
    - html_image_url_extractor
    - html_text
```
En `addons.yaml` tenemos que tener en cuenta:
* No indicar ninguno de los repositorios de los directorios `odoo`y `private`, estos se añaden de forma automatica. 
* Solo los addons aqui reflejados son symlinked en `/opt/odoo/auto/addons`, que es directorio indicado en el fichero de configuración `odoo.conf` para cargar los addons.
* En caso de que tengamos un addon con el mismo nombre, la prioridad es la siguiente:
    * Addons en `private`
    * Addons indicados en `addons.yaml`
    * Addons de Odoo
 
##### `/opt/odoo/custom/src/requirements.txt`
Es un fichero pip requirements.txt, mediante el cual se instalaran las dependencias de los addons al construir la imagen.
 
##### `/opt/odoo/custom/src/odoo`
Código fuente de Odoo para el proyecto.
Desde el `repos.yaml` podemos elegir la versión de Odoo a utilizar, e incluso fusionar PR de varias. Algunas de las fuentes a considerar:
* [Odoo Original](https://github.com/odoo/odoo), por [Odoo, S.A.](https://www.odoo.com/).
* [OCB (Odoo Community Backports)](https://github.com/OCA/OCB), por [OCA](https://odoo-community.org/). El original con alguna caracteristica adicional, pero menor estabilidad.
* [OpenUpgrade](), por [OCA](https://odoo-community.org/), El original, en el lanzamiento mas los scripts de migracion.
 
##### `/opt/odoo/custom/src/private`
Aquí almacenaremos los addons privados del proyecto. Ojo¡¡¡ addons, no carpetas.
 
### `/opt/odoo/common`
Es donde se encuentran los scripts que se encargan de que esto funcione.
Desde aqui se elimina todo el codigo no necesario, que no esta indicado en `addons.yaml`.
 
### `/opt/odoo/auto`
#### `/opt/odoo/auto/addons`
Estaran los symlinks a los addons indicados en `addons.yaml`.
#### `/opt/odoo/auto/odoo.conf`
El resultado de fusionar todas las configuraciones indicadas en `
#### `/opt/odoo/auto/addons`/opt/odoo/{common,custom}/conf.d/`
#### `/opt/odoo/auto/addons`
 
## Dockerfile
Utilizamos el de Tecnativa 
```sh
FROM tecnativa/odoo-base:10.0
```
## Scaffolding
Levanta y pon en marcha de forma rapida un proyecto.
### Clonamos proyecto
```sh
git clone -b scaffolding https://github.com/Ingetive/docker-odoo-base.git myproject
cd myproject
```
Mediante la generación de un symlink definimos el `docker-compose.yml` a utilizar, `devel.yaml` por ejemplo:
```sh
ln -s devel.yaml docker-compose.yml
```
 
### Entornos disponibles
Estan disponibles varias configuraciones para distintos entornos, cada una de ellas es un __Docker Compose File__. Deben entenderse como plantillas y seran necesario ajustarlos adecuadamente.
 
#### Desarrollo
Configuramos el entorno de desarrollo
```sh
docker-compose -f setup-devel.yaml run --rm odoo
```
Una vez terminado, puedes comenzar Odoo con:
```sh
docker-compose -f devel.yaml up --build
```
#### Test
No incluye `smtp`ni `backup`.
Lo ponemos en marcha con:
```sh
docker-compose -f test.yaml up --build
```
Sera necesario un inverse proxy corriendo.
 
#### Produccion
Incluye servicios de `smtp` y `backup`.
Lo ponemos en marcha con:
```sh
docker-compose -f prod.yaml up --build --remove-orphans
```
Sera necesario un inverse proxy corriendo.
 
#### Global Inverse proxy
Usamos el siguiente fichero `inverseproxy.yaml`:
```sh
version: "2.1"
 
services:
    proxy:
        image: traefik:1.3-alpine
        networks:
            shared:
            private:
        volumes:
            - acme:/etc/traefik/acme:rw,Z
        ports:
            - "80:80"
            - "443:443"
        depends_on:
            - dockersocket
        restart: unless-stopped
        privileged: true
        tty: true
        command:
            - --ACME.ACMELogging
            - --ACME.Email=you@example.com
            - --ACME.EntryPoint=https
            - --ACME.OnHostRule
            - --ACME.Storage=/etc/traefik/acme/acme.json
            - --DefaultEntryPoints=http,https
            - --EntryPoints=Name:http Address::80 Redirect.EntryPoint:https
            - --EntryPoints=Name:https Address::443 TLS
            - --LogLevel=INFO
            - --Docker
            - --Docker.EndPoint=http://dockersocket:2375
            - --Docker.ExposedByDefault=false
            - --Docker.Watch
 
    dockersocket:
        image: tecnativa/docker-socket-proxy
        privileged: true
        networks:
            private:
        volumes:
            - /var/run/docker.sock:/var/run/docker.sock
        environment:
            CONTAINERS: 1
            NETWORKS: 1
            SERVICES: 1
            SWARM: 1
            TASKS: 1
        restart: unless-stopped
 
networks:
    shared:
        driver_opts:
            encrypted: 1
 
    private:
        driver_opts:
            encrypted: 1
 
volumes:
    acme:
```
Y lo lanzaremos con:
```sh
docker-compose -p inverseproxy -f inverseproxy.yaml up -d
```
El proxy realiza las siguientes funciones:
* Descarga e instala los certificados SSL de [Let's Encrypt](https://letsencrypt.org) cada vez que arranques una nueva instncia de produccion.
* Interceptara todas las requests al puerto 80 (`http`) y las redigira al 443 (`https`).
* Añade los _proxy headers_ requeridos y redirecciona todo el trafico a/de Odoo automaticamente.
 
Se incluye un [security enaced proxy](https://hub.docker.com/r/tecnativa/docker-socket-proxy/) que reduce la superficie de ataque. Permite:
* Tener multiples dominios por cada instancia de Odoo.
* Tener multiples instancias de Odoo en cada nodo.
* Añadir SSL gratis.
 
#### Otros
##### Inspeccionar la base de datos
```sh
docker-compose run --rm odoo psql
```
##### Restart Odoo
Necesario ante cualquier cambio de código Python
```sh
docker-compose restart -t0 odoo
```
En produccion
```sh
docker-compose restart odoo https
```
##### Logs
Completo:
```sh
docker-compose logs -f --tail 10
```
Solo Odoo:
```sh
docker-compose logs -f --tail 10 odoo
```
##### Instalar addons sin detener Odoo
```sh
docker-compose run --rm odoo odoo -i addon1, addon2 --stop-after-init
```
##### Actualizar addons sin detener Odoo
```sh
docker-compose run --rm odoo odoo -u addon1, addon2 --stop-after-init
```
##### Exportar la translacion de addons a stdout
```sh
docker-compose run --rm odoo odoo -i addon1, addon2 --stop-after-init
```
##### Abrir el shell de Odoo
```sh
docker-compose run --rm odoo odoo shell
```
##### Abrir otra instancia UI linkada al mismo filestore y database
```sh
docker-compose run --rm -p 127:0.0.1:$some_free_port:8069 odoo
```
## FAQ
### Añadir o eliminar prefijo www. en produccion
Traefik de momento no soporta esta funcionalidad, asi que, por ejemplo, para eliminarlo bastara con añadir el siguiente servicio a nuestro `prod.yaml`:
```sh
www_remove:
    image: tecnativa/odoo-proxy
    environment:
        FORCEHOST: $DOMAIN_PROD
    networks:
        default:
        inverseproxy_shared:
    labels:
        traefik.docker.network: "inverseproxy_shared"
        traefik.enable: "true"
        traefik.frontend.passHostHeader: "true"
        traefik.port: "80"
        traefik.frontend.rule: "Host:www.${DOMAIN_PROD}"
```
### Permitir acceso desde varios dominios
En `.env` cargar la variable `DOMAIN_PROD` con `host1.com, host2.com, host3.com`
 
 
 
# [Docker Odoo Base Image](https://hub.docker.com/r/tecnativa/odoo-base)
Docker Odoo  Ingetive utilizando la realizada por Tecnativa
 
## Objetivo
 
Disponer de una base sobre Docker que posibilite el despliegue agil y rapido de Odoo.
Quedando recogidas un conjunto de buenas practicas y herramientas para el desarrollo sobre Odoo.
 
## Estructura
 
Todo queda dentro de `/opt/odoo`.
 
    custom/
        entrypoint.d/
        build.d/
        conf.d/
        src/
            private/
            odoo/
            addons.yaml
            repos.yaml
            requirements.txt
    common/
        entrypoint.sh
        build.sh
        entrypoint.d/
        build.d/
        conf.d/
    auto
        addons/
        odoo.conf
 
 
### `/opt/odoo/custom`
Aqui esta todo lo relacionado con el proyecto
#### `/opt/odoo/custom/entrypoint.d`
Los ejecutables que aqui se encuentren seran ejecutados al lanzar el contenedor, antes del comando solicitado.
#### `/opt/odoo/custom/build.d`
Los ejecutables que aqui se encuentren seran ejecutados al lanzar el contenedor.
#### `/opt/odoo/custom/conf.d`
Los ficheros que aqui se encuentren seran expandidos y concatenados en `/opt/odoo/auto/odoo.conf`.
#### `/opt/odoo/custom/src`
Aqui estará el código. Este puede ser insertado de 2 formas:
 
* Usando `repos.yaml`, que lo añadirá automaticamente al crear el contenedor.
* De forma manual, copiandolo directamente.
 
##### `/opt/odoo/custom/src/respos.yaml`
Se usará `repos.yaml` para indicar todo que hay que añadir excepto `private`. 
Todo lo que añadamos menos `private` debe ser excluido de los repositorios; por tanto, en `.gitignore` se añaden las siguiente reglas:
```sh
odoo/custom/src/*
!odoo/custom/src/private
!odoo/custom/src/*.*
```
##### `/opt/odoo/custom/src/addons.yaml`
Mediante `addons.yaml` , que es un fichero de configuración de [git-aggregator](https://github.com/acsone/git-aggregator), indicaremos los addons visibles, una entrada por repositorio y addon.
```yaml
website:
    - website_cookie_notice
    - website_legal_page
web:
    - web_responsive
```
Podremos unir varios, utilizando `---`:
 
```yaml
# Spanish Localization
l10n-spain:
    - l10n_es
server-tools:
    - date_range
---
# SEO tools
website:
    - website_blog_excertp_img
server-tools: # Repetimos server-tools, no hay problema,  es un
              # documento diferente
    - html_image_url_extractor
    - html_text
```
En `addons.yaml` tenemos que tener en cuenta:
* No indicar ninguno de los repositorios de los directorios `odoo`y `private`, estos se añaden de forma automatica. 
* Solo los addons aqui reflejados son symlinked en `/opt/odoo/auto/addons`, que es directorio indicado en el fichero de configuración `odoo.conf` para cargar los addons.
* En caso de que tengamos un addon con el mismo nombre, la prioridad es la siguiente:
    * Addons en `private`
    * Addons indicados en `addons.yaml`
    * Addons de Odoo
 
##### `/opt/odoo/custom/src/requirements.txt`
Es un fichero pip requirements.txt, mediante el cual se instalaran las dependencias de los addons al construir la imagen.
 
##### `/opt/odoo/custom/src/odoo`
Código fuente de Odoo para el proyecto.
Desde el `repos.yaml` podemos elegir la versión de Odoo a utilizar, e incluso fusionar PR de varias. Algunas de las fuentes a considerar:
* [Odoo Original](https://github.com/odoo/odoo), por [Odoo, S.A.](https://www.odoo.com/).
* [OCB (Odoo Community Backports)](https://github.com/OCA/OCB), por [OCA](https://odoo-community.org/). El original con alguna caracteristica adicional, pero menor estabilidad.
* [OpenUpgrade](), por [OCA](https://odoo-community.org/), El original, en el lanzamiento mas los scripts de migracion.
 
##### `/opt/odoo/custom/src/private`
Aquí almacenaremos los addons privados del proyecto. Ojo¡¡¡ addons, no carpetas.
 
### `/opt/odoo/common`
Es donde se encuentran los scripts que se encargan de que esto funcione.
Desde aqui se elimina todo el codigo no necesario, que no esta indicado en `addons.yaml`.
 
### `/opt/odoo/auto`
#### `/opt/odoo/auto/addons`
Estaran los symlinks a los addons indicados en `addons.yaml`.
#### `/opt/odoo/auto/odoo.conf`
El resultado de fusionar todas las configuraciones indicadas en `
#### `/opt/odoo/auto/addons`/opt/odoo/{common,custom}/conf.d/`
#### `/opt/odoo/auto/addons`
 
## Dockerfile
Utilizamos el de Tecnativa 
```sh
FROM tecnativa/odoo-base:10.0
```
## Scaffolding
Levanta y pon en marcha de forma rapida un proyecto.
### Clonamos proyecto
```sh
git clone -b scaffolding https://github.com/Ingetive/docker-odoo-base.git myproject
cd myproject
```
Mediante la generación de un symlink definimos el `docker-compose.yml` a utilizar, `devel.yaml` por ejemplo:
```sh
ln -s devel.yaml docker-compose.yml
```
 
### Entornos disponibles
Estan disponibles varias configuraciones para distintos entornos, cada una de ellas es un __Docker Compose File__. Deben entenderse como plantillas y seran necesario ajustarlos adecuadamente.
 
#### Desarrollo
Configuramos el entorno de desarrollo
```sh
docker-compose -f setup-devel.yaml run --rm odoo
```
Una vez terminado, puedes comenzar Odoo con:
```sh
docker-compose -f devel.yaml up --build
```
#### Test
No incluye `smtp`ni `backup`.
Lo ponemos en marcha con:
```sh
docker-compose -f test.yaml up --build
```
Sera necesario un inverse proxy corriendo.
 
#### Produccion
Incluye servicios de `smtp` y `backup`.
Lo ponemos en marcha con:
```sh
docker-compose -f prod.yaml up --build --remove-orphans
```
Sera necesario un inverse proxy corriendo.
 
#### Global Inverse proxy
Usamos el siguiente fichero `inverseproxy.yaml`:
```sh
version: "2.1"
 
services:
    proxy:
        image: traefik:1.3-alpine
        networks:
            shared:
            private:
        volumes:
            - acme:/etc/traefik/acme:rw,Z
        ports:
            - "80:80"
            - "443:443"
        depends_on:
            - dockersocket
        restart: unless-stopped
        privileged: true
        tty: true
        command:
            - --ACME.ACMELogging
            - --ACME.Email=you@example.com
            - --ACME.EntryPoint=https
            - --ACME.OnHostRule
            - --ACME.Storage=/etc/traefik/acme/acme.json
            - --DefaultEntryPoints=http,https
            - --EntryPoints=Name:http Address::80 Redirect.EntryPoint:https
            - --EntryPoints=Name:https Address::443 TLS
            - --LogLevel=INFO
            - --Docker
            - --Docker.EndPoint=http://dockersocket:2375
            - --Docker.ExposedByDefault=false
            - --Docker.Watch
 
    dockersocket:
        image: tecnativa/docker-socket-proxy
        privileged: true
        networks:
            private:
        volumes:
            - /var/run/docker.sock:/var/run/docker.sock
        environment:
            CONTAINERS: 1
            NETWORKS: 1
            SERVICES: 1
            SWARM: 1
            TASKS: 1
        restart: unless-stopped
 
networks:
    shared:
        driver_opts:
            encrypted: 1
 
    private:
        driver_opts:
            encrypted: 1
 
volumes:
    acme:
```
Y lo lanzaremos con:
```sh
docker-compose -p inverseproxy -f inverseproxy.yaml up -d
```
El proxy realiza las siguientes funciones:
* Descarga e instala los certificados SSL de [Let's Encrypt](https://letsencrypt.org) cada vez que arranques una nueva instncia de produccion.
* Interceptara todas las requests al puerto 80 (`http`) y las redigira al 443 (`https`).
* Añade los _proxy headers_ requeridos y redirecciona todo el trafico a/de Odoo automaticamente.
 
Se incluye un [security enaced proxy](https://hub.docker.com/r/tecnativa/docker-socket-proxy/) que reduce la superficie de ataque. Permite:
* Tener multiples dominios por cada instancia de Odoo.
* Tener multiples instancias de Odoo en cada nodo.
* Añadir SSL gratis.
 
#### Otros
##### Inspeccionar la base de datos
```sh
docker-compose run --rm odoo psql
```
##### Restart Odoo
Necesario ante cualquier cambio de código Python
```sh
docker-compose restart -t0 odoo
```
En produccion
```sh
docker-compose restart odoo https
```
##### Logs
Completo:
```sh
docker-compose logs -f --tail 10
```
Solo Odoo:
```sh
docker-compose logs -f --tail 10 odoo
```
##### Instalar addons sin detener Odoo
```sh
docker-compose run --rm odoo odoo -i addon1, addon2 --stop-after-init
```
##### Actualizar addons sin detener Odoo
```sh
docker-compose run --rm odoo odoo -u addon1, addon2 --stop-after-init
```
##### Exportar la translacion de addons a stdout
```sh
docker-compose run --rm odoo odoo -i addon1, addon2 --stop-after-init
```
##### Abrir el shell de Odoo
```sh
docker-compose run --rm odoo odoo shell
```
##### Abrir otra instancia UI linkada al mismo filestore y database
```sh
docker-compose run --rm -p 127:0.0.1:$some_free_port:8069 odoo
```
## FAQ
### Añadir o eliminar prefijo www. en produccion
Traefik de momento no soporta esta funcionalidad, asi que, por ejemplo, para eliminarlo bastara con añadir el siguiente servicio a nuestro `prod.yaml`:
```sh
www_remove:
    image: tecnativa/odoo-proxy
    environment:
        FORCEHOST: $DOMAIN_PROD
    networks:
        default:
        inverseproxy_shared:
    labels:
        traefik.docker.network: "inverseproxy_shared"
        traefik.enable: "true"
        traefik.frontend.passHostHeader: "true"
        traefik.port: "80"
        traefik.frontend.rule: "Host:www.${DOMAIN_PROD}"
```
### Permitir acceso desde varios dominios
En `.env` cargar la variable `DOMAIN_PROD` con `host1.com, host2.com, host3.com`
 
 
 
 
