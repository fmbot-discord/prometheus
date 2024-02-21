# .fmbot monitoring stack

This is a fork from the default prometheus monitoring stack, customized for .fmbot.

The below readme is a default description of the project. We should probably improve this, because all the bloat from the fork is not required.

The grafana dashboard is currently not publicly available.

Running:

    docker stack deploy -c docker-stack.yml prom

Removing:

    docker stack rm prom

Logs for a specific container:

	docker service logs prom_prometheus

Required `~/.bashrc` variables are `SEQ_PASSWORD` and `GRAFANA_PASSWORD`

# Contents

- Introduction
  - [Overview](#a-prometheus--grafana-docker-compose-stack)
  - [Pre-requisites](#pre-requisites)
  - [Installation & Configuration](#installation--configuration)
    - [Add Datasources & Dashboards](#add-datasources-and-dashboards)
    - [Install Dashboards the Old Way](#install-dashboards-the-old-way)
  	- [Alerting](#alerting)
  	- [Test Alerts](#test-alerts)
    - [Add additional Datasources](#add-additional-datasources)
  - [Deploy Prometheus stack with Traefik](#deploy-prometheus-stack-with-traefik)
  - [Security Considerations](#security-considerations)
  	- [Production Security](#production-security)
  - [Troubleshooting](#troubleshooting)
  	- [Mac Users](#mac-users)
  - [Interesting Projects that use this Repo](#interesting-projects-that-use-this-repo)

# A Prometheus & Grafana docker-compose stack

Here's a quick start using Play-With-Docker (PWD) to start-up a [Prometheus](http://prometheus.io/) stack containing Prometheus, Grafana and Node scraper to monitor your Docker infrastructure. The Try in PWD below allows you to quickly deploy the entire Prometheus stack with a click of the button. This will allow you to quickly test the stack to see if it meets your needs.

[![Try in PWD](https://github.com/play-with-docker/stacks/raw/master/assets/images/button.png)](https://labs.play-with-docker.com/?stack=https://raw.githubusercontent.com/vegasbrianc/prometheus/master/pwd-stack.yml)

# Pre-requisites
Before we get started installing the Prometheus stack. Ensure you install the latest version of docker and [docker swarm](https://docs.docker.com/engine/swarm/swarm-tutorial/) on your Docker host machine. Docker Swarm is installed automatically when using Docker for Mac or Docker for Windows.

# Installation & Configuration
Clone the project locally to your Docker host.

If you would like to change which targets should be monitored or make configuration changes edit the [/prometheus/prometheus.yml](prometheus/prometheus.yml) file. The targets section is where you define what should be monitored by Prometheus. The names defined in this file are actually sourced from the service name in the docker-compose file. If you wish to change names of the services you can add the "container_name" parameter in the `docker-compose.yml` file.

Once configurations are done let's start it up. From the /prometheus project directory run the following command:

    $ HOSTNAME=$(hostname) docker stack deploy -c docker-stack.yml prom


That's it the `docker stack deploy' command deploys the entire Grafana and Prometheus stack automagically to the Docker Swarm. By default cAdvisor and node-exporter are set to Global deployment which means they will propogate to every docker host attached to the Swarm.

The Grafana Dashboard is now accessible via: `http://<Host IP Address>:3000` for example http://192.168.10.1:3000

	username - admin
	password - foobar (Password is stored in the `/grafana/config.monitoring` env file)

In order to check the status of the newly created stack:

    $ docker stack ps prom

View running services:

    $ docker service ls

View logs for a specific service

    $ docker service logs prom_<service_name>

## Add Datasources and Dashboards
Grafana version 5.0.0 has introduced the concept of provisioning. This allows us to automate the process of adding Datasources & Dashboards. The `/grafana/provisioning/` directory contains the `datasources` and `dashboards` directories. These directories contain YAML files which allow us to specify which datasource or dashboards should be installed. 

If you would like to automate the installation of additional dashboards just copy the Dashboard `JSON` file to `/grafana/provisioning/dashboards` and it will be provisioned next time you stop and start Grafana.

## Install Dashboards the old way

I created a Dashboard template which is available on [Grafana Docker Dashboard](https://grafana.net/dashboards/179). Simply select Import from the Grafana menu -> Dashboards -> Import and provide the Dashboard ID [#179](https://grafana.net/dashboards/179)

This dashboard is intended to help you get started with monitoring. If you have any changes you would like to see in the Dashboard let me know so I can update Grafana site as well.

Here's the Dashboard Template

![Grafana Dashboard](https://raw.githubusercontent.com/vegasbrianc/prometheus/master/images/Dashboard.png)

Grafana Dashboard - `dashboards/Grana_Dashboad.json`
Alerting Dashboard


## Alerting
Alerting has been added to the stack with Slack integration. 2 Alerts have been added and are managed

Alerts              - `prometheus/alert.rules`
Slack configuration - `alertmanager/config.yml`

View Prometheus alerts `http://<Host IP Address>:9090/alerts`
View Alert Manager `http://<Host IP Address>:9093`

### Test Alerts
A quick test for your alerts is to stop a service. Stop the node_exporter container and you should notice shortly the alert arrive in Slack. Also check the alerts in both the Alert Manager and Prometheus Alerts just to understand how they flow through the system.

High load test alert - `docker run --rm -it busybox sh -c "while true; do :; done"`

Let this run for a few minutes and you will notice the load alert appear. Then Ctrl+C to stop this container.

### Add Additional Datasources
Now we need to create the Prometheus Datasource in order to connect Grafana to Prometheus 
* Click the `Grafana` Menu at the top left corner (looks like a fireball)
* Click `Data Sources`
* Click the green button `Add Data Source`.

**Ensure the Datasource name `Prometheus`is using uppercase `P`**

<img src="https://raw.githubusercontent.com/vegasbrianc/prometheus/master/images/Add_Data_Source.png" width="400" heighth="400">


## Prometheus & Grafana now have hostnames

* Grafana - http://grafana.localhost
* Prometheus - http://prometheus.localhost


# Production Security:

Here are just a couple security considerations for this stack to help you get started.
* Remove the published ports from Prometheus and Alerting servicesi and only allow Grafana to be accessed
* Enable SSL for Grafana with a Proxy such as [jwilder/nginx-proxy](https://hub.docker.com/r/jwilder/nginx-proxy/) or [Traefik](https://traefik.io/) with Let's Encrypt
* Add user authentication via a Reverse Proxy [jwilder/nginx-proxy](https://hub.docker.com/r/jwilder/nginx-proxy/) or [Traefik](https://traefik.io/) for services cAdvisor, Prometheus, & Alerting as they don't support user authenticaiton
* Terminate all services/containers via HTTPS/SSL/TLS