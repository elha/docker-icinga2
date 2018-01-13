docker-icinga2
==============

Installs an working icinga2 Master or Satellite based on [alpine-linux](https://www.alpinelinux.org/about/).

This Version includes also an small REST-Service to generate the Certificates for a Satellite via REST Service.

It also include an docker-compose **example** to create a set of one Master and 2 Satellites with automatich Certificate Exchange.

More then one API User can also be created over one Environment Variables.


# Status

[![Docker Pulls](https://img.shields.io/docker/pulls/bodsch/docker-icinga2.svg?branch)][hub]
[![Image Size](https://images.microbadger.com/badges/image/bodsch/docker-icinga2.svg?branch)][microbadger]
[![Build Status](https://travis-ci.org/bodsch/docker-icinga2.svg?branch)][travis]

[hub]: https://hub.docker.com/r/bodsch/docker-icinga2/
[microbadger]: https://microbadger.com/images/bodsch/docker-icinga2
[travis]: https://travis-ci.org/bodsch/docker-icinga2


# Build
Your can use the included Makefile.
- To build the Container: `make`
- To remove the builded Docker Image: `make clean`
- Starts the Container: `make run`
- Starts the Container with Login Shell: `make shell`
- Entering the Container: `make exec`
- Stop (but **not kill**): `make stop`
- History `make history`

# Contribution
Please read [Contribution](CONTRIBUTIONG.md)

# Development,  Branches (Github Tags)
The `master` Branch is my *Working Horse* includes the "latest, hot shit" and can be complete broken!

If you want to use something stable, please use a [Taged Version](https://github.com/bodsch/docker-icinga2/tags) or an [Branch](https://github.com/bodsch/docker-icinga2/branches) like `1712` or `1801`

# side-channel / custom scripts
if use need some enhancements, you can add some (bash) scripts and add them via volume to the conatiner:

```bash
--volume=/${PWD}/tmp/test.sh:/init/custom.d/test.sh
```

***This scripts will be started before everything else!***

***YOU SHOULD KNOW WHAT YOU'RE DOING.***

***THIS CAN BREAK THE COMPLETE ICINGA2 CONFIGURATION!***


# Availability
I use the official [Icinga2 packages](https://pkgs.alpinelinux.org/packages?name=icinga2&branch=&repo=&arch=&maintainer=) from Apline.

If one of them is removed, please contact Alpine and don't complain here!

I remove branches as soon as they are disfunctional (e. g. if a package is no longer available at Alpine). Not immediately, but certainly after 2 months.


# Docker Hub
You can find the Container also at  [DockerHub](https://hub.docker.com/r/bodsch/docker-icinga2/)


# Notices
The actuall Container Supports a stable MySQL Backand to store all needed Datas into it.

the graphite feature is **experimentally** and not conclusively tested.


## activated Icinga2 Features
- command
- checker
- mainlog
- notification
- graphite (only with API User and with )


# certificate service (**EXPERIMENTAL**)

[Sourcecode](https://github.com/bodsch/ruby-icinga-cert-service)

To connect a satellite to a master you need a certificate, which is issued by the master and signed by its CA.

The Icinga2 documentation provides more information about [Distributed Monitoring and Certificates](https://github.com/Icinga/icinga2/blob/master/doc/06-distributed-monitoring.md#signing-certificates-on-the-master-).

**I strongly recommend a study of the documentation!**

Within a docker environment this is a bit more difficult, so an external service is used to simplify this.
This service is constantly being developed further, but is integrated into the docker container in a stable version.

**The certificate service is only available at an Icinga2 Master!**

## usage

Certificate exchange is automated within the docker containers.
If you want to issue your own certificate, you can use the following API calls.

**You need a valid and configured API User in Icinga2.**



### new way (since Icinga2 2.8)

You can use `expect` on a *satellite* or *agent* to create an certificate request with the *icinga2 node wizard*:

    expect /init/node-wizard.expect

After this, you can use the *cert-service* to sign this request:

    curl \
      --user ${ICINGA_CERT_SERVICE_BA_USER}:${ICINGA_CERT_SERVICE_BA_PASSWORD} \
      --silent \
      --request GET \
      --header "X-API-USER: ${ICINGA_CERT_SERVICE_API_USER}" \
      --header "X-API-PASSWORD: ${ICINGA_CERT_SERVICE_API_PASSWORD}" \
      --write-out "%{http_code}\n" \
      --output /tmp/sign_${HOSTNAME}.json \
      http://${ICINGA_CERT_SERVICE_SERVER}:${ICINGA_CERT_SERVICE_PORT}/v2/sign/${HOSTNAME}

After a restart of the Icinga2 Master the certificate is active and a secure connection can be established.


### old way (pre Icinga2 2.8)

To create a certificate:

    curl \
      --request GET \
      --user ${ICINGA_CERT_SERVICE_BA_USER}:${ICINGA_CERT_SERVICE_BA_PASSWORD} \
      --silent \
      --header "X-API-USER: ${ICINGA_CERT_SERVICE_API_USER}" \
      --header "X-API-PASSWORD: ${ICINGA_CERT_SERVICE_API_PASSWORD}" \
      --output /tmp/request_${HOSTNAME}.json \
      http://${ICINGA_CERT_SERVICE_SERVER}:${ICINGA_CERT_SERVICE_PORT}/v2/request/${HOSTNAME}

Extract the session checksum from the request above.

    checksum=$(jq --raw-output .checksum /tmp/request_${HOSTNAME}.json)

Download the created certificate:

    curl \
      --request GET \
      --user ${ICINGA_CERT_SERVICE_BA_USER}:${ICINGA_CERT_SERVICE_BA_PASSWORD} \
      --silent \
      --header "X-API-USER: ${ICINGA_CERT_SERVICE_API_USER}" \
      --header "X-API-PASSWORD: ${ICINGA_CERT_SERVICE_API_PASSWORD}" \
      --header "X-CHECKSUM: ${checksum}" \
      --output /tmp/${HOSTNAME}/${HOSTNAME}.tgz \
       http://${ICINGA_CERT_SERVICE_SERVER}:${ICINGA_CERT_SERVICE_PORT}/v2/cert/${HOSTNAME}


**The generated Certificate has an Timeout from 10 Minutes between beginning of creation and download.**

You can also look into `rootfs/init/examples/use_cert-service.sh`

For Examples to create a certificate with commandline tools look into `rootfs/init/examples/cert-manager.sh`


# supported Environment Vars

**make sure you only use the environment variable you need!**

## database support

| Environmental Variable             | Default Value        | Description                                                     |
| :--------------------------------- | :-------------       | :-----------                                                    |
| `MYSQL_HOST`                       | -                    | MySQL Host                                                      |
| `MYSQL_PORT`                       | `3306`               | MySQL Port                                                      |
| `MYSQL_ROOT_USER`                  | `root`               | MySQL root User                                                 |
| `MYSQL_ROOT_PASS`                  | *randomly generated* | MySQL root password                                             |
| `IDO_DATABASE_NAME`                | `icinga2core`        | Schema Name for IDO                                             |
| `IDO_PASSWORD`                     | *randomly generated* | MySQL password for IDO                                          |

## create API User

| Environmental Variable             | Default Value        | Description                                                     |
| :--------------------------------- | :-------------       | :-----------                                                    |
| `ICINGA_API_USERS`                 | -                    | comma separated List to create API Users. The Format are `username:password` |
|                                    |                      | (e.g. `admin:admin,dashing:dashing` and so on)                  |

## support Carbon/Graphite

| Environmental Variable             | Default Value        | Description                                                     |
| :--------------------------------- | :-------------       | :-----------                                                    |
|                                    |                      |                                                                 |
| `CARBON_HOST`                      | -                    | hostname or IP address where Carbon/Graphite daemon is running  |
| `CARBON_PORT`                      | `2003`               | Carbon port for graphite                                        |

## support the Icinga Cert-Service

| Environmental Variable             | Default Value        | Description                                                     |
| :--------------------------------- | :-------------       | :-----------                                                    |
| `ICINGA_MASTER`                    | -                    | The Icinga2-Master FQDN for a Satellite Node                    |
| `ICINGA_PARENT`                    | -                    | The Parent Node for an Cluster Setup                            |
|                                    |                      |                                                                 |
| `BASIC_AUTH_USER`                  | `admin`              | both `BASIC_AUTH_*` and the `ICINGA_MASTER` are importand, if you |
| `BASIC_AUTH_PASS`                  | `admin`              | use and modify the authentication of the *icinga-cert-service*  |
|                                    |                      |                                                                 |
| `ICINGA_CERT_SERVICE_BA_USER`      | `admin`              | The Basic Auth User for the certicate Service                   |
| `ICINGA_CERT_SERVICE_BA_PASSWORD`  | `admin`              | The Basic Auth Password for the certicate Service               |
| `ICINGA_CERT_SERVICE_API_USER`     | -                    | The Certificate Service needs also an API Users                 |
| `ICINGA_CERT_SERVICE_API_PASSWORD` | -                    |                                                                 |
| `ICINGA_CERT_SERVICE_SERVER`       | `localhost`          | Certificate Service Host                                        |
| `ICINGA_CERT_SERVICE_PORT`         | `80`                 | Certificate Service Port                                        |
| `ICINGA_CERT_SERVICE_PATH`         | `/`                  | Certificate Service Path (needful, when they run behind a Proxy |

## notifications over SMTP

| Environmental Variable             | Default Value        | Description                                                     |
| :--------------------------------- | :-------------       | :-----------                                                    |
| `ICINGA_SSMTP_RELAY_SERVER`        | -                    | SMTP Service to send Notifications                              |
| `ICINGA_SSMTP_REWRITE_DOMAIN`      | -                    |                                                                 |
| `ICINGA_SSMTP_RELAY_USE_STARTTLS`  | -                    |                                                                 |
| `ICINGA_SSMTP_SENDER_EMAIL`        | -                    |                                                                 |
| `ICINGA_SSMTP_SMTPAUTH_USER`       | -                    |                                                                 |
| `ICINGA_SSMTP_SMTPAUTH_PASS`       | -                    |                                                                 |
| `ICINGA_SSMTP_ALIASES`             | -                    |                                                                 |

## activate some Demodata (taken from the official Icinga-Vagrant repository)

| Environmental Variable             | Default Value        | Description                                                     |
| :--------------------------------- | :-------------       | :-----------                                                    |
| `DEMO_DATA`                        | `false`              | copy demo data from `/init/demo-data` into `/etc/icinga2` config path |

