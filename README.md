# Elastic stack (ELK) on Docker

![Elastic Stack version](https://img.shields.io/badge/ELK--Version-7.0.1-green.svg?style=for-the-badge)
[![Main Project](https://img.shields.io/badge/Main%20Project-deviantony%2Fdocker--elk-blue.svg?style=for-the-badge)](https://github.com/deviantony/docker-elk)

Run the latest version of the [Elastic stack][elk-stack] with Docker and Docker Compose.

It gives you the ability to analyze any data set by using the searching/aggregation capabilities of Elasticsearch and
the visualization power of Kibana.

Based on the official Docker images from Elastic:

* [elasticsearch](https://github.com/elastic/elasticsearch-docker)
* [logstash](https://github.com/elastic/logstash-docker)
* [kibana](https://github.com/elastic/kibana-docker)


## Contents

- [Elastic stack (ELK) on Docker](#elastic-stack-elk-on-docker)
  - [Contents](#contents)
  - [Requirements](#requirements)
    - [Host setup](#host-setup)
  - [Usage](#usage)
    - [Bringing up the stack](#bringing-up-the-stack)
  - [Initial setup](#initial-setup)
    - [Setting up user authentication](#setting-up-user-authentication)
    - [Default Kibana index pattern creation](#default-kibana-index-pattern-creation)
      - [Via the Kibana web UI](#via-the-kibana-web-ui)
  - [Configuration](#configuration)
    - [How to configure Elasticsearch](#how-to-configure-elasticsearch)
    - [How to configure Kibana](#how-to-configure-kibana)
    - [How to configure Logstash](#how-to-configure-logstash)
  - [Storage](#storage)
    - [How to persist Elasticsearch data](#how-to-persist-elasticsearch-data)
  - [Extensibility](#extensibility)
    - [How to add plugins](#how-to-add-plugins)
    - [How to enable the provided extensions](#how-to-enable-the-provided-extensions)
  - [JVM tuning](#jvm-tuning)
    - [How to specify the amount of memory used by a service](#how-to-specify-the-amount-of-memory-used-by-a-service)
  - [Going further](#going-further)
    - [Using a newer stack version](#using-a-newer-stack-version)
    - [Plugins and integrations](#plugins-and-integrations)

## Requirements

### Host setup

1. Install [Docker](https://www.docker.com/community-edition#/download) version **17.05+**
2. Install [Docker Compose](https://docs.docker.com/compose/install/) version **1.6.0+**
3. Clone this repository

By default, the stack exposes the following ports:
* 5000: Logstash TCP input
* 9200: Elasticsearch HTTP
* 9300: Elasticsearch TCP transport
* 5601: Kibana

## Usage

### Bringing up the stack

Start the stack using Docker Compose:

```console
$ docker-compose up
```

You can also run all services in the background (detached mode) by adding the `-d` flag to the above command.

> :information_source: You must run `docker-compose build` first whenever you switch branch or update a base image.

If you are starting the stack for the very first time, please read the section below attentively.

## Initial setup

### Setting up user authentication

The stack is pre-configured with the following **privileged** bootstrap user:

* user: *elastic*
* password: *changeme*

Although all stack components work out-of-the-box with this user, we strongly recommend using the unprivileged [built-in
users][builtin-users] instead for increased security. Passwords for these users must be initialized:

```console
$ docker-compose exec -T elasticsearch 'bin/elasticsearch-setup-passwords' auto --batch
```

Passwords for all 6 built-in users will be randomly generated. Take note of them and replace the `elastic` username with
`kibana` and `logstash_system` inside the Kibana and Logstash *pipeline* configuration files respectively. See the
[Configuration](#configuration) section below.

Restart Kibana and Logstash to apply the passwords you just wrote to the configuration files.

```console
$ docker-compose restart kibana logstash
```

Give Kibana a few seconds to initialize, then access the Kibana web UI by hitting
[http://localhost:5601](http://localhost:5601) with a web browser and use the following default credentials to login:

* user: *elastic*
* password: *\<your generated elastic password>*

Now that the stack is running, you can go ahead and inject some log entries. The shipped Logstash configuration allows
you to send content via TCP:

```console
$ nc localhost 5000 < /path/to/logfile.log
```

### Default Kibana index pattern creation

When Kibana launches for the first time, it is not configured with any index pattern.

#### Via the Kibana web UI

> :information_source: You need to inject data into Logstash before being able to configure a Logstash index pattern via
the Kibana web UI. Then all you have to do is hit the *Create* button.

Refer to [Connect Kibana with Elasticsearch][connect-kibana] for detailed instructions about the index pattern
configuration.

## Configuration

> :information_source: Configuration is not dynamically reloaded, you will need to restart individual components after
any configuration change.

### How to configure Elasticsearch

The Elasticsearch configuration is stored in [`elasticsearch/config/elasticsearch.yml`][config-es].

### How to configure Kibana

The Kibana default configuration is stored in [`kibana/config/kibana.yml`][config-kbn].

It is also possible to map the entire `config` directory instead of a single file.

### How to configure Logstash

The Logstash configuration is stored in [`logstash/config/logstash.yml`][config-ls].

It is also possible to map the entire `config` directory instead of a single file, however you must be aware that
Logstash will be expecting a [`log4j2.properties`][log4j-props] file for its own logging.

## Storage

### How to persist Elasticsearch data

The data stored in Elasticsearch will be persisted after container reboot but not after container removal.

In order to persist Elasticsearch data even after removing the Elasticsearch container, you'll have to mount a volume on
your Docker host. Update the `elasticsearch` service declaration to:

```yml
elasticsearch:

  volumes:
    - /path/to/storage:/usr/share/elasticsearch/data
```

This will store Elasticsearch data inside `/path/to/storage`.

> :information_source: (Linux users) Beware that the Elasticsearch process runs as the [unprivileged `elasticsearch`
user][esuser] is used within the Elasticsearch image, therefore the mounted data directory must be writable by the uid
`1000`.

## Extensibility

### How to add plugins

To add plugins to any ELK component you have to:

1. Add a `RUN` statement to the corresponding `Dockerfile` (eg. `RUN logstash-plugin install logstash-filter-json`)
2. Add the associated plugin code configuration to the service configuration (eg. Logstash input/output)
3. Rebuild the images using the `docker-compose build` command

### How to enable the provided extensions

A few extensions are available inside the [`extensions`](extensions) directory. These extensions provide features which
are not part of the standard Elastic stack, but can be used to enrich it with extra integrations.

The documentation for these extensions is provided inside each individual subdirectory, on a per-extension basis. Some
of them require manual changes to the default ELK configuration.

## JVM tuning

### How to specify the amount of memory used by a service

By default, both Elasticsearch and Logstash start with [1/4 of the total host
memory](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#default_heap_size) allocated to
the JVM Heap Size.

The startup scripts for Elasticsearch and Logstash can append extra JVM options from the value of an environment
variable, allowing the user to adjust the amount of memory that can be used by each component:

| Service       | Environment variable |
| ------------- | -------------------- |
| Elasticsearch | ES_JAVA_OPTS         |
| Logstash      | LS_JAVA_OPTS         |

To accomodate environments where memory is scarce (Docker for Mac has only 2 GB available by default), the Heap Size
allocation is capped by default to 256MB per service in the `docker-compose.yml` file. If you want to override the
default JVM configuration, edit the matching environment variable(s) in the `docker-compose.yml` file.

For example, to increase the maximum JVM Heap Size for Logstash:

```yml
logstash:

  environment:
    LS_JAVA_OPTS: "-Xmx1g -Xms1g"
```

## Going further

### Using a newer stack version

To use a different Elastic Stack version than the one currently available in the repository, simply change the version
number inside the `.env` file, and rebuild the stack with:

```console
$ docker-compose build
$ docker-compose up
```

> :information_source: Always pay attention to the [upgrade instructions][upgrade] for each individual component before
performing a stack upgrade.

### Plugins and integrations

See the following Wiki pages:

* [External applications](https://github.com/deviantony/docker-elk/wiki/External-applications)
* [Popular integrations](https://github.com/deviantony/docker-elk/wiki/Popular-integrations)


[elk-stack]: https://www.elastic.co/elk-stack
[stack-features]: https://www.elastic.co/products/stack

[builtin-users]: https://www.elastic.co/guide/en/x-pack/current/setting-up-authentication.html#built-in-users

[connect-kibana]: https://www.elastic.co/guide/en/kibana/current/connect-to-elasticsearch.html

[config-es]: ./elasticsearch/config/elasticsearch.yml
[config-kbn]: ./kibana/config/kibana.yml
[config-ls]: ./logstash/config/logstash.yml

[log4j-props]: https://github.com/elastic/logstash-docker/tree/master/build/logstash/config
[esuser]: https://github.com/elastic/elasticsearch-docker/blob/c2877ef/.tedi/template/bin/docker-entrypoint.sh#L9-L10

[upgrade]: https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-upgrade.html
