# Raiden Transport

## What is this repository

This repository contains the documentation and configuration necessary to run a
Raiden Transport Matrix server.

**Current release:** [2019.11.1](https://github.com/raiden-network/raiden-service-bundle/tree/2019.11.1)

## Table of Contents

- [Overview](#overview)
- [Requirements](#requirements)
  - [Hardware](#hardware)
  - [Software](#software)
- [Installation](#installation)
- [Upgrades](#upgrades)
- [Known issues](#known-issues)
- [Changelog](#changelog)

## Overview

The Raiden Network uses a federation of [Matrix](https://matrix.org) servers as its transport
layer. To ensure reliability, availability and neutrality it is desirable that those servers are
being operated by multiple independent entities.

Therefore we provide this repository which allows easy setup of such a transport server.
It uses docker and docker-compose for easy installation and upgrades.

Currently only this single-server configuration is supported; in the future we may also
provide configurations with services split among multiple servers.

### Used software

- docker
- docker-compose
- Synapse
- Postgres
- Traefik
- Raiden Services (Pathfinding, Monitoring)

### Structure


```
+-------------------+
|                   |
|   Raiden clients  |
|                   |
+---+-----------+---+
    |matrix://  |pfs://
====|===========|====
    |           |
+---v-----------v---+                       Federation to
|                 +-+-------------------->  other Raiden
|      Traefik    | |                       Matrix servers
|                 | |
+---------+-------+-+---------+
          |       |           |
+---------v-------v-+   +-----v----------------+ +---------------------+
|                   |   |                      | |                     |
|      Synapse      |   |  Raiden Pathfinding  | |  Raiden Monitoring  |
|                   |   |                      | |                     |
+---------+---------+   +-------------------+--+ +-+-------------------+
          |                                 |      |
+---------v---------+                     +-v- - - v -+
|                   |                     |
|     Postgres      |                        ETH_RPC  |
|                   |                     |
+-------------------+                     + - - - - - +
```


We use Traefik as a reverse proxy and also utilize its capability of automatically provisioning
Let's Encrypt TLS certificates.

The Synapse server is being run in the so-called split worker configuration which increases throughput.

The database stores the message data. Since the transport layer is considered ephemeral in Raiden it
is not necessary to arrange for backups of the database data.

### Network

After a successful deployment the following ports will be in use:

- 80 - HTTP
  - Redirects to HTTPS
  - Let's Encrypt HTTP challenge for certificate provisioning
- 443 - HTTPS
  - Synapse web and API client access
  - Synapse Server-to-Server federation
  - Raiden Pathfinding Server (on subdomain `pfs.$SERVER_NAME`)
  - Metrics export (IP restricted, see below)

## Requirements

### Hardware

Minimum recommended for a production setup:

- 16 GiB RAM
-  8 Cores
- 50 GiB SSD

Note: The default Postgres configuration assumes 16GiB of system RAM

### Software

- Docker >= 17.12
- docker-compose >= 1.21.0

### Other

- A domain (or subdomain) for exclusive use by this server

## Installation

### Preparation

1. Provision a server that meets the [hardware](#hardware) and [software](#software) requirements listed above.
1. Ensure a domain (or subdomain) is available

   Examples:
   - raidentransport.somecompany.tld
   - raidentransport-somecompany.tld
   - somecompany-raidentransport.tld

1. Configure `A` (and optionally `AAAA`) DNS records for the domain pointing to the servers IP address(es)
1. Configure `A` (and optionally `AAAA`) DNS records for `pfs.<domain>` pointing to the servers IP address(es)
1. Configure a `CNAME` DNS record for `*.<domain>` pointing back to `<domain>`

### Installing

1. Clone the [current release version of this repository](https://github.com/raiden-network/raiden-service-bundle/tree/2019.11.1)
   to a suitable location on the server:

   ```shell
   git clone -b 2019.11.1 https://github.com/raiden-network/raiden-service-bundle.git
   ```
1. Copy `.env.template` to `.env` and modify the values to fit your setup (see inline comments for details)
   - We would appreciate it if you allow us access to the monitoring interfaces
     (to do that uncomment the default values of the `CIDR_ALLOW_METRICS` and `CIDR_ALLOW_PROXY` settings).
   - We also recommend that you provide your own monitoring. The setup of which is currently out of scope of this document.
1. Make sure, that the account, configured in `KEYSTORE_FILE`, has enough funding to register as a service operator.
1. If you haven't done so before, run `./register-service-provider.sh` (it uses configuration values from `.env`).
1. Run `docker-compose up -d` to start all services
   - The services are configured to automatically restart in case of a crash or reboot
1. Verify the service is up by opening the domain in a browser. You should see a page with the Matrix logo.

### Troubleshooting
After starting, you can run `docker-compose ps` -- if any services are not in `Up`, `Up (healthy)` or `Exit 0` state, you should check the respective logs for configuration errors.
Note: some services might need a few minutes to become healthy.

### Submit a Transport Server to the federation

1. [Create an issue](https://github.com/raiden-network/raiden-service-bundle/issues/new) and submit the
   domain / URL of the newly deployed server for inclusion in the list of known servers.

## Upgrades

To upgrade to a new release please check the [changelog](#changelog) for any necessary
configuration changes and then run the following commands:

```shell
git fetch origin --tags
git reset --hard <new-release-tag>
docker-compose pull
docker-compose up -d
```
## Notes:

 - A 'purge' script/service will run once a day, pruning old state from rooms to save disk space,
   and restarting the ``synapse`` service to fetch an up-to-date whitelist of servers.

## Known issues

### Protection against Spam / (D)DoS attacks

There is currently only some protection against Spam and / or DDoS attacks.
This will be addressed in future updates.

### Known servers

The known servers the Raiden clients try to connect to are currently tracked in the *.yml files in this repository.
These lists are used by Raiden clients when the ``--matrix-server=auto`` (default) option is used,
for automatically selecting a transport server, based on response times.
We intend to change this in the future to use a decentralized scheme (for example an on-chain registry).


## Contact / Troubleshooting

To report issues or request help with the setup please [open an issue](https://github.com/raiden-network/raiden-service-bundle/issues/new)
or contact us via email at contact@raiden.nework.


## Changelog
See `CHANGELOG.md`.

## Licenses

The code and documentation in this repository are released under the [MIT license](LICENSE).

This repository contains instructions to install third party software. Those are licensed as follows:

- [Traefik](https://github.com/containous/traefik): [MIT](https://github.com/containous/traefik/blob/6a55772cda1684546a6a5456b6847e0f9b3df44d/LICENSE.md)
- [Synapse](https://github.com/matrix-org/synapse): [Apache 2.0](https://github.com/matrix-org/synapse/blob/047ac0cbba25ccffb93d92b46254a58f48ddaf9c/LICENSE)
- [PostgreSQL](https://www.postgresql.org): [PostgreSQL License](https://www.postgresql.org/about/licence/)
