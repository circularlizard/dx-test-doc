---
id: k8s-next-postgres-prom
title: Postgres Metrics
---

## PoC Goals

The goal is to evaluate how we can expose Prometheus compatible metrics in our Postgres persistence. The outcome of that PoC should be documented so it can be reused for a thorough implementation in our Helm deployment.

The PoC will use the new persistence deployment with [Pgpool](https://www.pgpool.net/) and [Repmgr](https://repmgr.org/) (DX internally called *"dbHA"*), but the described pattern can be used for the "old" persistence as well.

## Postgres Metrics

A Postgres Prometheus exporter exists as a prebuilt Docker image as well as [on Github](https://github.com/prometheus-community/postgres_exporter). It connects to the database and exposes the Prometheus compatible metrics as an HTTP endpoint. The default metrics are described in the [`queries.yaml`](https://github.com/prometheus-community/postgres_exporter/blob/master/queries.yaml) file. Custom metrics can be defined as a YAML file and attached when the exporter is started by using the [`extend.query-path` flag](https://github.com/prometheus-community/postgres_exporter#flags).

### Adding a sidecar container to the Postgres Pod

The preferred way to run the Exporter is to add it to the Postgres Pod as a sidecar container. The configuration below describes the basic setup using the public postgres-exporter image.

```yaml
spec:
  template:
    metadata:
      annotations:
        # Tell Prometheus to scrape the metrics and where to find them
        prometheus.io/path: "/metrics"
        prometheus.io/port: "9187"
        prometheus.io/scrape: "true"
    spec:
      containers:
        - name: "persistence-node-metrics"
          image: "quay.io/prometheuscommunity/postgres-exporter"
          env:
            - name: DATA_SOURCE_URI
              # If implemented like that, we should check fo a way to not hard-code the "dxmediadb"
              value: "127.0.0.1:5432/dxmediadb?sslmode=disable"
            - name: "DATA_SOURCE_PASS"
              valueFrom:
                secretKeyRef:
                  key: "password"
                  name: "{{ .Release.Name }}-persistence-user"
            - name: "DATA_SOURCE_USER"
              valueFrom:
                secretKeyRef:
                  key: "username"
                  name: "{{ .Release.Name }}-persistence-user"
          ports:
            - name: metrics
              containerPort: 9187
              protocol: TCP
```
### Create Postgres Exporter image based on the UBI image

When shipped as part of DX, the Postgres Exporter should be included as a separate image, based on the UBI image. We can use the published releases from Github and add it to the UBI image.

```Dockerfile
#
####################################################################
# Licensed Materials - Property of HCL                             #
#                                                                  #
# Copyright HCL Technologies Ltd. 2021. All Rights Reserved.       #
#                                                                  #
# Note to US Government Users Restricted Rights:                   #
#                                                                  #
# Use, duplication or disclosure restricted by GSA ADP Schedule    #
####################################################################
#

ARG REPOSITORY_URL="quintana-docker-prod.artifactory.cwp.pnp-hcl.com"
ARG DOCKER_UBI_BASE_IMAGE="dx-build-output/common/dxubi:v1.0.0_8.4-205"

FROM $REPOSITORY_URL/$DOCKER_UBI_BASE_IMAGE

ARG BUILD_LABEL
ARG VERSION

LABEL "product"="HCL Digital Experience Postgres exporter"
LABEL "version"="${VERSION}"
LABEL "description"="DX postgres exporter container"
LABEL "io.k8s.description"="DX postgres exporter container"
LABEL "io.k8s.display-name"="DX postgres exporter container"
LABEL "summary"="DX postgres exporter container"
LABEL "name"="dx-postgresql-exporter"
LABEL "release"="${BUILD_LABEL}"
LABEL "maintainer"="HCL Software"
LABEL "vendor"="HCL Software"
LABEL "io.openshift.tags"="hcl dx"
LABEL "url"=""
LABEL "authoritative-source-url"=""

MAINTAINER HCL Software

RUN curl -LJO https://github.com/prometheus-community/postgres_exporter/releases/download/v0.10.0/postgres_exporter-0.10.0.linux-amd64.tar.gz && \
    tar -xvf postgres_exporter-0.10.0.linux-amd64.tar.gz && \
    rm postgres_exporter-0.10.0.linux-amd64.tar.gz && \
    mv postgres_exporter-0.10.0.linux-amd64/postgres_exporter /usr/bin/postgres_exporter && \
    rm -r postgres_exporter-0.10.0.linux-amd64

USER dx_user:dx_users
EXPOSE 9187
ENTRYPOINT ["/usr/bin/postgres_exporter"]
```

## Pgpool Metrics

Similar to the Postgres exporter, a docker image exists to export the metrics of Pgpool. The code and releases can be found [on Github](https://github.com/pgpool/pgpool2_exporter). The exposed metrics are described on the same page. As we are currently using Pgpool in version 4.1, some of the metrics are not available for the exporter. To use it to it's full potential, an upgrade of Pgpool to version 4.2 should be considered.

### Adding a sidecar container to the Pgpool Pod

The preferred way to run the Exporter is to add it to the Pgpool Pod as a sidecar container. The configuration below describes the basic setup using the public pgpool-exporter image.

```yaml
spec:
  template:
    metadata:
      annotations:
        # Tell Prometheus to scrape the metrics and where to find them
        prometheus.io/path: "/metrics"
        prometheus.io/port: "9719"
        prometheus.io/scrape: "true"
    spec:
      containers:
        - name: "persistence-connection-pool-stats"
          image: pgpool/pgpool2_exporter
          env:
          - name: "POSTGRES_USERNAME"
            valueFrom:
              secretKeyRef:
                key: "username"
                name: "{{ .Release.Name }}-persistence-user" 
          - name: "POSTGRES_PASSWORD"
            valueFrom:
              secretKeyRef:
                key: "password"
                name: "{{ .Release.Name }}-persistence-user"      
          - name: PGPOOL_SERVICE
            value: "localhost"
          - name: PGPOOL_SERVICE_PORT
            value: "5432"
          ports:
            - name: metrics
              containerPort: 9719
              protocol: TCP
```

### Create Pgpool Exporter image based on the UBI image

When shipped as part of DX, the Pgpool Exporter should be included as a separate image, based on the UBI image. We can use the published releases from Github and add it to the UBI image.

```Dockerfile
#
####################################################################
# Licensed Materials - Property of HCL                             #
#                                                                  #
# Copyright HCL Technologies Ltd. 2021. All Rights Reserved.       #
#                                                                  #
# Note to US Government Users Restricted Rights:                   #
#                                                                  #
# Use, duplication or disclosure restricted by GSA ADP Schedule    #
####################################################################
#

ARG REPOSITORY_URL="quintana-docker-prod.artifactory.cwp.pnp-hcl.com"
ARG DOCKER_UBI_BASE_IMAGE="dx-build-output/common/dxubi:v1.0.0_8.4-205"

FROM $REPOSITORY_URL/$DOCKER_UBI_BASE_IMAGE

ARG BUILD_LABEL
ARG VERSION

LABEL "product"="HCL Digital Experience Pgpool exporter"
LABEL "version"="${VERSION}"
LABEL "description"="DX pgpool exporter container"
LABEL "io.k8s.description"="DX pgpool exporter container"
LABEL "io.k8s.display-name"="DX pgpool exporter container"
LABEL "summary"="DX pgpool exporter container"
LABEL "name"="dx-pgpool-exporter"
LABEL "release"="${BUILD_LABEL}"
LABEL "maintainer"="HCL Software"
LABEL "vendor"="HCL Software"
LABEL "io.openshift.tags"="hcl dx"
LABEL "url"=""
LABEL "authoritative-source-url"=""

MAINTAINER HCL Software

ENV POSTGRES_USERNAME postgres
ENV POSTGRES_PASSWORD postgres
ENV PGPOOL_SERVICE localhost
ENV PGPOOL_SERVICE_PORT 9999

RUN curl -LJO https://github.com/pgpool/pgpool2_exporter/releases/download/v1.0.0/pgpool2_exporter-1.0.0.linux-amd64.tar.gz && \
    tar -xvf pgpool2_exporter-1.0.0.linux-amd64.tar.gz && \
    rm pgpool2_exporter-1.0.0.linux-amd64.tar.gz && \
    mv pgpool2_exporter-1.0.0.linux-amd64/pgpool2_exporter /usr/bin/pgpool2_exporter && \
    rm -r pgpool2_exporter-1.0.0.linux-amd64

USER dx_user:dx_users
EXPOSE 9719
ENTRYPOINT ["/bin/sh", "-c", "export DATA_SOURCE_NAME=\"postgresql://${POSTGRES_USERNAME}:${POSTGRES_PASSWORD}@${PGPOOL_SERVICE}:${PGPOOL_SERVICE_PORT}/postgres?sslmode=disable\"; /usr/bin/pgpool2_exporter"]
```

## Using existing Grafana Dashboards

There are also existing Grafana dashboards that can be leveraged.

One that is related to Postgres with Prometheuse can be found here [Grafana Dashboard 9628](https://grafana.com/grafana/dashboards/9628). For the Pgpool exporter, a predefined Dashboard is not available in the official Grafana Dashboard directory.
