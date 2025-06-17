# Elastic stack (ELK) on Docker

Ref: https://github.com/deviantony/docker-elk.git

[![Elastic Stack version](https://img.shields.io/badge/Elastic%20Stack-9.0.2-00bfb3?style=flat&logo=elastic-stack)](https://www.elastic.co/blog/category/releases)
[![Build Status](https://github.com/deviantony/docker-elk/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/deviantony/docker-elk/actions/workflows/ci.yml?query=branch%3Amain)
[![Join the chat](https://badges.gitter.im/Join%20Chat.svg)](https://app.gitter.im/#/room/#deviantony_docker-elk:gitter.im)

Run the latest version of the [Elastic stack][elk-stack] with Docker and Docker Compose.

It gives you the ability to analyze any data set by using the searching/aggregation capabilities of Elasticsearch and
the visualization power of Kibana.

Based on the [official Docker images][elastic-docker] from Elastic:

* [Elasticsearch](https://github.com/elastic/elasticsearch/tree/main/distribution/docker)
* [Logstash](https://github.com/elastic/logstash/tree/main/docker)
* [Kibana](https://github.com/elastic/kibana/tree/main/src/dev/build/tasks/os_packages/docker_generator)

Other available stack variants:

## try

```sh
docker compose up setup
```

```sh
docker compose up
```

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://github.com/user-attachments/assets/6f67cbc0-ddee-44bf-8f4d-7fd2d70f5217">
  <img alt="Animated demo" src="https://github.com/user-attachments/assets/501a340a-e6df-4934-90a2-6152b462c14a">
</picture>

---

---

## Requirements

### Host setup

* [Docker Engine][docker-install] version **18.06.0** or newer
* [Docker Compose][compose-install] version **2.0.0** or newer
* 1.5 GB of RAM

> [!NOTE]
> Especially on Linux, make sure your user has the [required permissions][linux-postinstall] to interact with the Docker
> daemon.

By default, the stack exposes the following ports:

* 5044: Logstash Beats input
* 50000: Logstash TCP input
* 9600: Logstash monitoring API
* 9200: Elasticsearch HTTP
* 9300: Elasticsearch TCP transport
* 5601: Kibana

> [!WARNING]
> Elasticsearch's [bootstrap checks][bootstrap-checks] were purposely disabled to facilitate the setup of the Elastic
> stack in development environments. For production setups, we recommend users to set up their host according to the
> instructions from the Elasticsearch documentation: [Important System Configuration][es-sys-config].

### Docker Desktop

#### Windows

If you are using the legacy Hyper-V mode of _Docker Desktop for Windows_, ensure that [File
Sharing][desktop-filesharing] is enabled for the `C:` drive.

#### macOS

The default configuration of _Docker Desktop for Mac_ allows mounting files from `/Users/`, `/Volume/`, `/private/`,
`/tmp` and `/var/folders` exclusively. Make sure the repository is cloned in one of those locations or follow the
instructions from the [documentation][desktop-filesharing] to add more locations.

## Usage

> [!WARNING]
> You must rebuild the stack images with `docker compose build` whenever you switch branch or update the
> [version](#version-selection) of an already existing stack.

### Bringing up the stack

Clone this repository onto the Docker host that will run the stack with the command below:

```sh
git clone https://github.com/MelRanG/docker-elk-settings.git
```

Then, initialize the Elasticsearch users and groups required by docker-elk by executing the command:

```sh
docker compose up setup
```

If everything went well and the setup completed without error, start the other stack components:

```sh
docker compose up
```

> [!NOTE]
> You can also run all services in the background (detached mode) by appending the `-d` flag to the above command.

Give Kibana about a minute to initialize, then access the Kibana web UI by opening <http://localhost:5601> in a web
browser and use the following (default) credentials to log in:

* user: *elastic*
* password: *elastic*

> [!NOTE]
> Upon the initial startup, the `elastic`, `logstash_internal` and `kibana_system` Elasticsearch users are initialized
> with the values of the passwords defined in the [`.env`](.env) file (_"elastic"_ by default). The first one is the
> [built-in superuser][builtin-users], the other two are used by Kibana and Logstash respectively to communicate with
> Elasticsearch. This task is only performed during the _initial_ startup of the stack. To change users' passwords
> _after_ they have been initialized, please refer to the instructions in the next section.

### Initial setup

#### Setting up user authentication

> [!NOTE]
> Refer to [Security settings in Elasticsearch][es-security] to disable authentication.

> [!WARNING]
> Starting with Elastic v8.0.0, it is no longer possible to run Kibana using the bootstrapped privileged `elastic` user.

The _"elastic"_ password set by default for all aforementioned users is **unsecure**. For increased security, we will
reset the passwords of all aforementioned Elasticsearch users to random secrets.

1. Reset passwords for default users

    The commands below reset the passwords of the `elastic`, `logstash_internal` and `kibana_system` users. Take note
    of them.

    ```sh
    docker compose exec elasticsearch bin/elasticsearch-reset-password --batch --user elastic
    ```

    ```sh
    docker compose exec elasticsearch bin/elasticsearch-reset-password --batch --user logstash_internal
    ```

    ```sh
    docker compose exec elasticsearch bin/elasticsearch-reset-password --batch --user kibana_system
    ```

    If the need for it arises (e.g. if you want to [collect monitoring information][ls-monitoring] through Beats and
    other components), feel free to repeat this operation at any time for the rest of the [built-in
    users][builtin-users].

1. Replace usernames and passwords in configuration files

    Replace the password of the `elastic` user inside the `.env` file with the password generated in the previous step.
    Its value isn't used by any core component, but [extensions](#how-to-enable-the-provided-extensions) use it to
    connect to Elasticsearch.

    > [!NOTE]
    > In case you don't plan on using any of the provided [extensions](#how-to-enable-the-provided-extensions), or
    > prefer to create your own roles and users to authenticate these services, it is safe to remove the
    > `ELASTIC_PASSWORD` entry from the `.env` file altogether after the stack has been initialized.

    Replace the password of the `logstash_internal` user inside the `.env` file with the password generated in the
    previous step. Its value is referenced inside the Logstash pipeline file (`logstash/pipeline/logstash.conf`).

    Replace the password of the `kibana_system` user inside the `.env` file with the password generated in the previous
    step. Its value is referenced inside the Kibana configuration file (`kibana/config/kibana.yml`).

    See the [Configuration](#configuration) section below for more information about these configuration files.

1. Restart Logstash and Kibana to re-connect to Elasticsearch using the new passwords

    ```sh
    docker compose up -d logstash kibana
    ```

> [!NOTE]
> Learn more about the security of the Elastic stack at [Secure the Elastic Stack][sec-cluster].

#### Injecting data

Launch the Kibana web UI by opening <http://localhost:5601> in a web browser, and use the following credentials to log
in:

* user: *elastic*
* password: *elastic*

Now that the stack is fully configured, you can go ahead and inject some log entries.

The shipped Logstash configuration allows you to send data over the TCP port 50000. For example, you can use one of the
following commands — depending on your installed version of `nc` (Netcat) — to ingest the content of the log file
`/path/to/logfile.log` in Elasticsearch, via Logstash:

```sh
# Execute `nc -h` to determine your `nc` version

cat /path/to/logfile.log | nc -q0 localhost 50000          # BSD
cat /path/to/logfile.log | nc -c localhost 50000           # GNU
cat /path/to/logfile.log | nc --send-only localhost 50000  # nmap
```

You can also load the sample data provided by your Kibana installation.

### Cleanup

Elasticsearch data is persisted inside a volume by default.

In order to entirely shutdown the stack and remove all persisted data, use the following Docker Compose command:

```sh
docker compose down -v
```
