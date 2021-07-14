---
id: docker
title: Docker
---

![alt Docker Pulls Count](https://dockeri.co/image/verdaccio/verdaccio "Docker Pulls Count")

Per scaricare la più recente [immagine docker](https://hub.docker.com/r/verdaccio/verdaccio/) pre costruita:

```bash
docker pull verdaccio/verdaccio
```

![Docker pull](assets/docker_verdaccio.gif)

## Versioni taggate

Dalla versione `v2.x` si possono ottenere immagini docker per [tag](https://hub.docker.com/r/verdaccio/verdaccio/tags/), come segue:

Per una versione maggiore:

```bash
docker pull verdaccio/verdaccio:4
```

Per una versione minore:

```bash
docker pull verdaccio/verdaccio:4.0
```

Per una specifica (patch) versione:

```bash
docker pull verdaccio/verdaccio:4.0.0
```

> Se si è interessati ad un elenco dei tag, [ si prega di visitare il sito Docker Hub](https://hub.docker.com/r/verdaccio/verdaccio/tags/).

## Eseguire Verdaccio utilizzando Docker

Per avviare il contenitore Docker:

```bash
docker run -it --rm --name verdaccio -p 4873:4873 verdaccio/verdaccio
```

L'ultimo argomento definisce quale immagine utilizzare. La riga sopra citata scaricherà da dockerhub l'ultima immagine pre costruita disponibile, se ciò non è ancora stato fatto.

Se è stata [costruita un'immagine localmente](#build-your-own-docker-image) utilizzare `verdaccio` come ultimo argomento.

È possibile utilizzare `-v` per montare `conf`, `storage` e `plugins` nel filesystem degli host:

```bash
V_PATH=/path/for/verdaccio; docker run -it --rm --name verdaccio \
  -p 4873:4873 \
  -v $V_PATH/conf:/verdaccio/conf \
  -v $V_PATH/storage:/verdaccio/storage \
  -v $V_PATH/plugins:/verdaccio/plugins \
  verdaccio/verdaccio
```

> if you are running in a server, you might want to add -d to run it in the background
> 
> Note: Verdaccio runs as a non-root user (uid=10001) inside the container, if you use bind mount to override default, you need to make sure the mount directory is assigned to the right user. In above example, you need to run `sudo chown -R 10001:65533 /path/for/verdaccio` otherwise you will get permission errors at runtime. [Use docker volume](https://docs.docker.com/storage/volumes/) is recommended over using bind mount.

Verdaccio 4 fornisce un nuovo set di variabili d'ambiente per modificare le autorizzazioni, la porta o il protocollo http. Qui l'elenco completo:

| Proprietà             | default          | Descrizione                                                           |
| --------------------- | ---------------- | --------------------------------------------------------------------- |
| VERDACCIO_APPDIR      | `/opt/verdaccio` | la directory di lavoro docker                                         |
| VERDACCIO_USER_NAME | `verdaccio`      | l'utente del sistema                                                  |
| VERDACCIO_USER_UID  | `10001`          | l'id utente utilizzato per applicare le autorizzazioni della cartella |
| VERDACCIO_PORT        | `4873`           | la porta di verdaccio                                                 |
| VERDACCIO_PROTOCOL    | `http`           | il protocollo http predefinito                                        |

### SELinux

If SELinux is enforced in your system, the directories to be bind-mounted in the container need to be relabeled. Otherwise verdaccio will be forbidden from reading those files.

     fatal--- cannot open config file /verdaccio/conf/config.yaml: Error: CONFIG: it does not look like a valid config file
    

If verdaccio can't read files on a bind-mounted directory and you are unsure, please check `/var/log/audit/audit.log` to confirm that it's a SELinux issue. In this example, the error above produced the following AVC denial.

    type=AVC msg=audit(1606833420.789:9331): avc:  denied  { read } for  pid=1251782 comm="node" name="config.yaml" dev="dm-2" ino=8178250 scontext=system_u:system_r:container_t:s0:c32,c258 tcontext=unconfined_u:object_r:user_home_t:s0 tclass=file permissive=0
    

`chcon` can change the labels of shared files and directories. To make a directory accessible to containers, change the directory type to `container_file_t`.

```sh
$ chcon -Rt container_file_t ./conf
```

If you want to make the directory accessible only to a specific container, use `chcat` to specify a matching SELinux category.

An alternative solution is to use [z and Z flags](https://docs.docker.com/storage/bind-mounts/#configure-the-selinux-label). To add the `z` flag to the mountpoint `./conf:/verdaccio/conf` simply change it to `./conf:/verdaccio/conf:z`. The `z` flag relabels the directory and makes it accessible by every container while the `Z` flags relables the directory and makes it accessible only to that specific container. However using these flags is dangerous. A small configuration mistake, like mounting `/home/user` or `/var` can mess up the labels on those directories and make the system unbootable.

### Plugin

Plugins can be installed in a separate directory and mounted using Docker or Kubernetes, however make sure you build plugins with native dependencies using the same base image as the Verdaccio Dockerfile.

```docker
FROM verdaccio/verdaccio

USER root

ENV NODE_ENV=production

RUN npm i && npm install verdaccio-s3-storage

USER verdaccio
```

### Docker and custom port configuration

Any `host:port` configured in `conf/config.yaml` under `listen` **is currently ignored when using docker**.

If you want to reach Verdaccio docker instance under different port, lets say `5000` in your `docker run` command add the environment variable `VERDACCIO_PORT=5000` and then expose the port `-p 5000:5000`.

```bash
V_PATH=/path/for/verdaccio; docker run -it --rm --name verdaccio \
  -e "VERDACCIO_PORT=8080" -p 8080:8080 \  
  verdaccio/verdaccio
```

Of course the numbers you give to `-p` paremeter need to match.

### Using HTTPS with Docker

You can configure the protocol verdaccio is going to listen on, similarly to the port configuration. You have to overwrite the default value("http") of the `PROTOCOL` environment variable to "https", after you specified the certificates in the config.yaml.

```bash
docker run -it --rm --name verdaccio \
  --env "VERDACCIO_PROTOCOL=https" -p 4873:4873
  verdaccio/verdaccio
```

### Using docker-compose

1. Scaricare l'ultima versione di [docker-compose](https://github.com/docker/compose).
2. Creare ed eseguire il container:

```bash
$ docker-compose up --build
```

You can set the port to use (for both container and host) by prefixing the above command with `VERDACCIO_PORT=5000`.

```yaml
version: '3.1'

services:
  verdaccio:
    image: verdaccio/verdaccio
    container_name: "verdaccio"
    networks:
      - node-network
    environment:
      - VERDACCIO_PORT=4873
    ports:
      - "4873:4873"
    volumes:
      - "./storage:/verdaccio/storage"
      - "./config:/verdaccio/conf"
      - "./plugins:/verdaccio/plugins"  
networks:
  node-network:
    driver: bridge
```

Docker will generate a named volume in which to store persistent application data. You can use `docker inspect` or `docker volume inspect` to reveal the physical location of the volume and edit the configuration, such as:

```bash
$ docker volume inspect verdaccio_verdaccio
[
    {
        "Name": "verdaccio_verdaccio",
        "Driver": "local",
        "Mountpoint": "/var/lib/docker/volumes/verdaccio_verdaccio/_data",
        "Labels": null,
        "Scope": "local"
    }
]

```

## Creare la propria immagine Docker

```bash
docker build -t verdaccio .
```

There is also an npm script for building the docker image, so you can also do:

```bash
yarn run build:docker
```

Note: The first build takes some minutes to build because it needs to run `npm install`, and it will take that long again whenever you change any file that is not listed in `.dockerignore`.

Please note that for any of the above docker commands you need to have docker installed on your machine and the docker executable should be available on your `$PATH`.

## Esempi Docker

There is a separate repository that hosts multiple configurations to compose Docker images with `verdaccio`, for instance, as reverse proxy:

<https://github.com/verdaccio/docker-examples>

## Build Personalizzate di Docker

> Se hai creato un'immagine basata su Verdaccio, aggiungila a questo elenco.

* [docker-verdaccio-multiarch](https://github.com/hertzg/docker-verdaccio-multiarch) Multiarch image mirrors
* [docker-verdaccio-gitlab](https://github.com/snics/docker-verdaccio-gitlab)
* [docker-verdaccio](https://github.com/deployable/docker-verdaccio)
* [docker-verdaccio-s3](https://github.com/asynchrony/docker-verdaccio-s3) Private NPM container that can backup to s3
* [docker-verdaccio-ldap](https://github.com/snadn/docker-verdaccio-ldap)
* [verdaccio-ldap](https://github.com/nathantreid/verdaccio-ldap)
* [verdaccio-compose-local-bridge](https://github.com/shingtoli/verdaccio-compose-local-bridge)
* [docker-verdaccio](https://github.com/Global-Solutions/docker-verdaccio)
* [verdaccio-docker](https://github.com/idahobean/verdaccio-docker)
* [verdaccio-server](https://github.com/andru255/verdaccio-server)
* [coldrye-debian-verdaccio](https://github.com/coldrye-docker/coldrye-debian-verdaccio) docker image providing verdaccio from coldrye-debian-nodejs.