
## subwire-logging-stack

This is a docker logging stack aimed at colelcting logs from network devices via syslog. It consists of a syslog daemon which accepts logs from devices on port 514, it then forwards logs to Promtail, then Loki, which stores logs in Minio; and finally makes them all available in Grafana for making pretty visualisations and dashboards.

## Getting Started

The project is built around a pre-configured Docker stack of the following:

 - [Grafana](https://grafana.com/oss/grafana/)
 - [Grafana Loki](https://grafana.com/oss/loki/) (configured for [MinIO](https://min.io/))
 - [Grafana Promtail](https://grafana.com/docs/loki/latest/clients/promtail/)
 - [syslog-ng](https://www.syslog-ng.com/)

The stack has been extended to include pre-configured monitoring with:

- [Prometheus](https://grafana.com/oss/prometheus/)
- [Node-Exporter](https://github.com/prometheus/node_exporter)
- [cAdvisor](https://github.com/google/cadvisor)


## Prerequisites

- [Docker](https://docs.docker.com/install)
- [Docker Compose](https://docs.docker.com/compose/install)

## Using

This project is built and tested on Ubuntu 22.04. To get started, download the code from this repository and extract it into an empty directory. For example:

    wget https://github.com/Subwire-Labs/subwire-logging-stack/archive/main.zip
    unzip main.zip
    cd subwire-logging-stack
    
From that directory, run the docker-compose command:

**Full Example Stack:** Grafana, Loki with s3/MinIO, Promtail, syslog-ng, Prometheus, cAdvisor, node-exporter

    docker volume create grafana-storage

    docker-compose -f ./docker-compose.yml up -d

This will start to download all of the needed application containers and start them up. 

*(Optional docker-compose configurations are listed under **Options** below)*

**Grafana Dashboards**

Once all of the docker containers are started, point your Web browser to the Grafana page, typically http://hostname:3000/ - with hostname being the name of the server you ran the docker-compose up -d command on.

    
**Getting Started With Loki**

Here are some additional resources you might find helpful if you're just getting started with Loki:

- [Getting started with Grafana and Loki in under 4
   minutes](https://grafana.com/go/webinar/loki-getting-started/)
- [An (only slightly technical) introduction to Loki](https://grafana.com/blog/2020/05/12/an-only-slightly-technical-introduction-to-loki-the-prometheus-inspired-open-source-logging-system/)
- [Video tutorial: Effective troubleshooting queries with Grafana
   Loki](https://grafana.com/blog/2021/01/07/video-tutorial-effective-troubleshooting-queries-with-grafana-loki/)

## Stack Options:

A few other docker-compose files are also available:

**Full Example Stack with Syslog Generator:** Grafana, Loki with s3/MinIO, Promtail, syslog-ng, Prometheus, cAdvisor, node-exporter, Syslog Generator

    docker-compose -f ./docker-compose-with-generator.yml up -d

**Example Stack without monitoring or Syslog generator**: Grafana, Loki with s3/MinIO, Promtail, syslog-ng

    docker-compose -f ./docker-compose-without-monitoring.yml up -d

**Example Stack without MinIO, monitoring, or Syslog generator:** Grafana, Loki with the filesystem, Promtail, syslog-ng

    docker-compose -f ./docker-compose-filesystem.yml up -d

The *Syslog Generator* configuration will need access to the Internet to do a local docker build from the configurations location in ./generator. It'll provide some named hosts and random INFO, WARN, DEBUG, ERROR logs sent over to syslog-ng/Loki.


## Configuration Review:

The default Loki storage configuration docker-compose.yml uses S3 storage with MinIO. If you want to use the filesystem instead, use the different docker-compose configurations listed above or change the configuration directly. An example would be:

    volumes:
    - ./config/loki-config-filesystem.ym:/etc/loki/loki-config.yml:ro

**Changing MinIO Keys**

The MinIO configurations default the Access Key and Secret Key at startup. If you want to change them, you'll need to update two files:

./docker-compose.yml

      MINIO_ACCESS_KEY: changemeGWPMTCezxtK0QF2wy
      MINIO_SECRET_KEY: changeme4CVbMbSfpC9MxAySY5humuee
      
./config/loki-config-s3.yml

     aws:
      s3: s3://changemeGWPMTCezxtK0QF2wy:changeme4CVbMbSfpC9MxAySY5humuee@minio.:9000/loki

## Changed Default Configurations In syslog-ng and Promtail

To set this example All In One project up, the following configurations have been added to the docker-compose.yml. If you already have syslog-ng running on your deployment server - make similar changes below and comment out the docker container stanza.

#### SYSLOG-NG CONFIGURATION (docker container listens on port 514)

**# syslog-ng.conf**

    source s_local {
        internal();
    };
    
    source s_network {
        default-network-drivers(
        );
    };
    
    destination d_loki {
        syslog("promtail" transport("tcp") port("1514"));
    };
    
    log {
            source(s_local);
            source(s_network);
            destination(d_loki);
    };

> Note: the above "`promtail`" configuration for `destination d_loki` is
> the *hostname* where Promtail is running. Is this example, it happens
> to be the Promtail *docker container* name that I configured for the
> All-In-One example.

#### PROMTAIL CONFIGURATION (docker container listens on port 1514)

 **# promtail-config.yml**

    server:
      http_listen_port: 9080
      grpc_listen_port: 0
    
    positions:
      filename: /tmp/positions.yaml
    
    clients:
      - url: http://loki:3100/loki/api/v1/push
    
    scrape_configs:
    
    - job_name: syslog
      syslog:
        listen_address: 0.0.0.0:1514
        idle_timeout: 60s
        label_structured_data: yes
        labels:
          job: "syslog"
      relabel_configs:
        - source_labels: ['__syslog_message_hostname']
          target_label: 'host'

