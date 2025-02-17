# SW360 Docker

### Table of Contents

[Building](#building)

[Running the Image](#running-the-image)

[Extra Configurations](#extra-configurations)


## Building

* Install Docker and docker-compose.
* Build docker compose

    You will need to have a recent docker and docker-compose installed.

    Build is done by the script:

    ```sh
    docker_build.sh
    ```

    The script will download all dependencies in the deps folder.

    Docker compose for sw360 are configured with default entries on docker-compose.yml.
    The default sample environment file is under `scripts/docker-config/default.docker.env`

    The config file looks like this:

    ```ini
    # scripts/docker-config/default.docker.env
    POSTGRES_USER=liferay
    POSTGRES_PASSWORD=liferay
    POSTGRES_DB=lportal
    COUCHDB_USER=admin
    COUCHDB_PASSWORD=password
    COUCHDB_CREATE_DATABASE=yes
    SW360_DATA=./data/sw360
    ```

    By default, data for postgres, couchdb and sw360 document will be persisted under `data` on current directory. 

    If you want to override all configs, copy `scripts/docker-config/default.docker.env` to project root as `.env` file and alter for your needs.

    Then just rebuild the project with -env_file option

* Proxy setup

    To build under proxy system, add this options on your custom env file:

    ```ini
    PROXY_ENABLED=true
    PROXY_HTTP_HOST=<your_http_proxy_ip>
    PROXY_HTTPS_HOST=<your_https_proxy_ip>
    PROXY_PORT=<your_port>
    ```

### Fossology

If you want to add Fossology in the mix, add FOSSOLOGY=1 on the build:

```sh
FOSSOLOGY=1 ./docker_build.sh
```

## Running the image

* Run the resulting image:

    ```sh
    docker-compose up
    ```

    or with fossology ( see above build instructions )

    ```sh
    docker-compose -f docker-compose.yml -f fossology-docker-compose.yml up
    ```

* With custom env file

    ```sh
    docker-compose --env-file <myenvfile> up
    ```

    You can add **-d** parameter at end of line to start in daemon mode and see the logs with the following command:

    ```sh
    docker logs -f sw360
    ```

## Configurations

By default, docker image of SW360 runs without internal web server and is assigned to be SSL as default. This is configured on *portal-ext.properties*

Here's some extra configurations that can be useful to fix some details.

## Customize portal-ext

The config file __portal-ext.properties__ overrides a second file that can be created to add a custom configuration with all data related to your necessities.

This file is called __portal-sw360.properties__

To add your custom configs, create this file under config dir on project root like this ( or with your favorite editor):

```sh
cd <sw360_source>
mkdir config
cat "company.default.name=MYCOMPANY" > config/sw360-portal-ext.properties
```

Docker compose with treat config as a bind volume dir and will expose to application.

### CSS layout looks wrong

If you do not use an external web server with redirection ( see below ), you may find the main CSS theme scrambled ( not properly loaded )

This happens because current Liferay used version try to access the theme using only canonical hostname, without the port assigned, so leading to an invalid CSS url.

To fix, you will need to change *portal-ext.properties* in data directory ( or your assigned data directory ) with the following extra value:

```ini
web.server.host=<your ip/host of docker>:<port>
```

This will tell liferay where is your real host instead of trying to guess the wrong host.

### Nginx config for reverse proxy and X-Frame issues on on host machine ( not docker )

For nginx, assuming you are using default config for your sw360, this is a simple configuration for root web server under Ubuntu.

```nginx
     location / {
         resolver 127.0.0.11 valid=30s;
         proxy_pass http://localhost:8080/;
         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
         proxy_set_header Host $http_host;
         proxy_set_header X-Forwarded-Proto https;
         proxy_redirect off;
         proxy_read_timeout 3600s;
         proxy_hide_header X-Frame-Options;
         add_header X-Frame-Options "ALLOWALL";
     }
```

***WARNING*** - X-frame is enabled wide open for development purposes. If you intend to use the above config in production, remember to properly secure the web server.

### Make https only **port 443** default

Modify the following line on your custom __portal-sw360.properties__ to https:

```ini
web.server.protocol=https
```

### Liferay Redirects

Liferay by default for security reasons do not allow redirect for unknown ips/domains, mostly on admin modules, so is necessary to add your domain or ip to the redirect allowed lists in custom __portal-sw360.properties__.
    
A not proper redirect can see in logs

**IP based** - The list of ips is separated by comma

```
redirect.url.security.mode=ip
redirect.url.ips.allowed=127.0.0.1,172.17.0.1,...
```

**Domain based** - The list domains is separated by comma
    
```
redirect.url.security.mode=domain
redirect.url.domain.allowed=example.com,*.wildcard.com
```
