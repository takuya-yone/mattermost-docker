# Production Docker deployment for Mattermost

|![](https://upload.wikimedia.org/wikipedia/commons/thumb/1/17/Warning.svg/156px-Warning.svg.png) | This project is no longer supported.
|---|---|

## NOTE: 
This repository has been a community-driven effort created when running Docker containers was just getting popular. This repository produced three images: one for Mattermost itself, another for Database, and for the Web Proxy.

We will no longer support those three images. If you have a Mattermost server running with the image mattermost/mattermost-prod-app, we recommend migrating either to mattermost/mattermost-enterprise-edition or mattermost/mattermost-team-edition images, which are the official ones and supported by Mattermost. These images support Postgres 10+ database, which we know has been a long-running challenge for the community, and you will not lose any features or functionality by moving to these new images.

If you have any issues or concerns with the migration, let us know in this GitHub issue.

In the future, this repo will contain examples of running Mattermost image in several types of services, like Swarm, AWS Beanstalk, and others, but will not have Dockerfile to build any custom image. Dockerfile will be deprecated in Mattermost version 6.0 in September 2021 and removed from the repository.

A new repository is available at https://github.com/mattermost/docker. It is still a work-in-progress, and we recommend testing a fresh setup first. If you decide to migrate to the new image, please make sure to have full data backups as we haven't fully tested the migration process yet, so there may be unforeseen issues until those tests have been completed. If any questions or feedback, let us know!

## WARNING:

The current state of this repository doesn't work out-of-the box since Mattermost server v5.31+ requires PostgreSQL versions of 10 or higher.

We're actively working on a fix to this repository. Until then, please refer to these upgrade instructions: https://github.com/mattermost/mattermost-docker/issues/489#issuecomment-790277661

This project enables a deployment of a Mattermost server in a multi-node production configuration using Docker.

[![Build Status](https://travis-ci.org/mattermost/mattermost-docker.svg?branch=master)](https://travis-ci.org/mattermost/mattermost-docker)

Notes:
- The default Mattermost edition for this repo has changed from Team Edition to Enterprise Edition. Please see [Choose Edition](#choose-edition-to-install) section.
- To install this Docker project on AWS Elastic Beanstalk please see [AWS Elastic Beanstalk Guide](contrib/aws/README.md).
- To run Mattermost on Kubernetes you can start with the [manifest examples in the kubernetes folder](contrib/kubernetes/README.md)
- To install Mattermost without Docker directly onto a Linux-based operating systems, please see [Admin Guide](https://docs.mattermost.com/guides/administrator.html#installing-mattermost).

## Installation using Docker Compose

The following instructions deploy Mattermost in a production configuration using multi-node Docker Compose set up.

### Requirements

* [docker] (version `1.12+`)
* [docker-compose] (version `1.10.0+` to support Compose file version `3.0`)

### Choose Edition to Install

If you want to install Enterprise Edition, you can skip this section.

To install the team edition, change `build: app` to `build:` and uncomment out these lines in `app:` services block to make it look like below in docker-compose.yaml file:
```yaml
app:
  build:
    context: app
    args:
      - edition=team
```
The `app` Dockerfile will read the `edition` build argument to install Team (`edition = 'team'`) or Enterprise (`edition != team`) edition.

### Database container
This repository offer a Docker image for the Mattermost database. It is a customized PostgreSQL image that you should configure with following environment variables :
* `POSTGRES_USER`: database username
* `POSTGRES_PASSWORD`: database password
* `POSTGRES_DB`: database name

It is possible to use your own PostgreSQL database, or even use MySQL. But you will need to ensure that Application container can connect to the database (see [Application container](#application-container))

#### AWS
If deploying to AWS, you could also set following variables to enable [Wal-E](https://github.com/wal-e/wal-e) backup to S3 :
* `AWS_ACCESS_KEY_ID`: AWS access key
* `AWS_SECRET_ACCESS_KEY`: AWS secret
* `WALE_S3_PREFIX`: AWS s3 bucket name
* `AWS_REGION`: AWS region

All four environment variables are required. It will enable completed WAL segments sent to archive storage (S3). The base backup and clean up can be done through the following command:
```bash
# Base backup
docker exec mattermost-db su - postgres sh -c "/usr/bin/envdir /etc/wal-e.d/env /usr/bin/wal-e backup-push /var/lib/postgresql/data"
# Keep the most recent 7 base backups and remove the old ones
docker exec mattermost-db su - postgres sh -c "/usr/bin/envdir /etc/wal-e.d/env /usr/bin/wal-e delete --confirm retain 7"
```
Those tasks can be executed through a cron job or systemd timer.

### Application container
Application container run the Mattermost application. You should configure it with following environment variables :
* `MM_USERNAME`: database username
* `MM_PASSWORD`: database password
* `MM_DBNAME`: database name

If your database use some custom host and port, it is also possible to configure them :
* `DB_HOST`: database host address
* `DB_PORT_NUMBER`: database port

Use this optional variable if your PostgreSQL connection requires encryption (you may need a certificate authority file and/or a certificate revocation list - check the documentation for your database provider).  See the [PostgreSQL notes on encrypted connections](https://www.postgresql.org/docs/current/libpq-ssl.html) for recommendations on what values to use when encryption is needed.
* `DB_SSLMODE`: defaults to `disable`, indicating no encryption

PostgreSQL allows two other variables `sslrootcert` and `sslcrl` for connection strings.  However these are not broadly supported when the connection string is specified as a URI. If you need these parameters, use the PostgreSQL-specified environment variables
* `PGSSLROOTCERT` specifies the location of CA file
* `PGSSLCRL` specifies the location of a certificate revocation list file

If you use a Mattermost configuration file on a different location than the default one (`/mattermost/config/config.json`) :
* `MM_CONFIG`: configuration file location inside the container.

If you choose to use MySQL instead of PostgreSQL, you should set a different datasource and SQL driver :
* `DB_PORT_NUMBER` : `3306`
* `MM_SQLSETTINGS_DRIVERNAME` : `mysql`
* `MM_SQLSETTINGS_DATASOURCE` : `MM_USERNAME:MM_PASSWORD@tcp(DB_HOST:DB_PORT_NUMBER)/MM_DBNAME?charset=utf8mb4,utf8&readTimeout=30s&writeTimeout=30s`
Don't forget to replace all entries (beginning by `MM_` and `DB_`) in `MM_SQLSETTINGS_DATASOURCE` with the real variables values.

If you want to push Mattermost application to **Cloud Foundry**, use a `manifest.yml` like this one (with external PostgreSQL service):

```
---
applications:
- name: mattermost
  docker:
    image: mattermost/mattermost-prod-app
  instances: 1
  memory: 1G
  disk_quota: 256M
  env:
    DB_HOST: database host address
    DB_PORT_NUMBER: database port
    MM_DBNAME: database name
    MM_USERNAME: database username
    MM_PASSWORD: database password

```

### Web server container
This image is optional, you should **not** use it when you have your own reverse-proxy. It is a simple front Web server for the Mattermost app container. If you use the provided `docker-compose.yml` file, you don't have to configure anything. But if your application container is reachable on custom host and/or port (eg. if you use a container provider), you should add those two environment variables :
* `APP_HOST`: application host address
* `APP_PORT_NUMBER`: application HTTP port

If you plan to upload large files to your Mattermost instance, Nginx will need to write some temporary files. In that case, the `read_only: true` option on the `web` container should be removed from your `docker-compose.yml` file.

#### Install with SSL certificate
Put your SSL certificate as `./volumes/web/cert/cert.pem` and the private key that has
no password as `./volumes/web/cert/key-no-password.pem`. If you don't have
them you may generate a self-signed SSL certificate.

#### Configure SSO with GitLab
If you are looking for SSO with GitLab and you use self signed certificate you have to add the PKI chain of your authority in app because Alpine doesn't know him. This is required to avoid **Token request failed: certificate signed by unknown authority**

For that uncomment this line and replace with the correct path of your PKI chain:
```
# - <path_to_your_gitlab_pki>/pki_chain.pem:/etc/ssl/certs/pki_chain.pem:ro
```

### Starting/Stopping Docker

#### Start
If you are running docker with non root user, make sure the UID and GID in app/Dockerfile are the same as your current UID/GID
```
mkdir -p ./volumes/app/mattermost/{data,logs,config,plugins}
chown -R 2000:2000 ./volumes/app/mattermost/
docker-compose start
```

#### Stop
```
docker-compose stop
```

### Removing Docker

#### Remove the containers
```
docker-compose stop && docker-compose rm
```

#### Remove the data and settings of your Mattermost instance
```
sudo rm -rf volumes
```

## Update Mattermost to latest version

First, shutdown your containers to back up your data.

```
docker-compose down
```

Back up your mounted volumes to save your data. If you use the default `docker-compose.yml` file proposed on this repository, your data is on `./volumes/` folder.

Then run the following commands.

```
git pull
docker-compose build
docker-compose up -d
```

Your Docker image should now be on the latest Mattermost version.


## Upgrading Mattermost to 4.9+

Docker images for `4.9.0` release introduce some important changes from [PR #241](https://github.com/mattermost/mattermost-docker/pull/241) to improve production use of Mattermost with Docker.
**There are 2 important changes for existing installations**

One important change is that we don't use `root` user by default to run the Mattermost application. So, as explained on [the README](https://github.com/mattermost/mattermost-docker#start), if you use host mounted volume you have to be sure that files on your host server have the correct UID/GID (by default those values are `2000`). In practice, you should just run following commands :
```
mkdir -p ./volumes/app/mattermost/{data,logs,config,plugins}
chown -R 2000:2000 ./volumes/app/mattermost/
```

The second important change is the port used by Mattermost application container. The default port is now `8000`, and existing installations that use port `80` will not work without a little configuration change. You have to open your Mattermost configuration file (`./volumes/app/mattermost/config/config.json` by default) and change the key `ServiceSettings.ListenAddress` to `:8000`.
Also if you use your own web-server/reverse-proxy you need to change its configuration to reach port `8000` of the Mattermost container.


## Upgrading to Team Edition 3.0.x from 2.x

You need to migrate your database before upgrading Mattermost to `3.0.x` from
`2.x`. Run these commands in the latest `mattermost-docker` directory.
```
docker-compose rm -f app
docker-compose build app
docker-compose run app -upgrade_db_30
docker-compose up -d
```
See the [official Upgrade Guide](http://docs.mattermost.com/administration/upgrade.html) for more details.

## Installation using Docker Swarm Mode

The following instructions deploy Mattermost in a production configuration using docker swarm mode on one node.
Running containerized applications on multi-node swarms involves specific data portability and replication handling that are not covered here.

### Requirements

* [docker] (1.12.0+)

### Swarm Mode Installation

First, create mattermost directory structure on the docker hosts:
```
mkdir -p /var/lib/mattermost/{cert,config,data,logs,plugins}
```

Then, fire up the stack in your swarm:
```
docker stack deploy -c contrib/swarm/docker-stack.yml mattermost
```

## Known Issues

* Do not modify the Listen Address in Service Settings.
* Rarely `app` container fails to start because of "connection refused" to
  database. Workaround: Restart the container.

## More information

If you want to know how to use docker-compose, see [the overview
page](https://docs.docker.com/compose).

For the server configurations, see [prod-ubuntu.rst] of Mattermost.

[docker]: http://docs.docker.com/engine/installation/
[docker-compose]: https://docs.docker.com/compose/install/
[prod-ubuntu.rst]: https://docs.mattermost.com/install/install-ubuntu-1604.html
