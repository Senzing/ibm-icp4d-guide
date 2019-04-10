# ibm-icp4d-guide

## Overview

The following diagram shows the relationship of the Helm charts, docker containers, and code in this Kubernetes demonstration.

![Image of architecture](architecture.png)

### Contents

1. [Expectations](#expectations)
    1. [Space](#space)
    1. [Time](#time)
    1. [Background knowledge](#background-knowledge)
1. [Demonstrate](#demonstrate)
    1. [Clone repository](#clone-repository)
    1. [Prerequisites](#prerequisites)
    1. [Set environment variables](#set-environment-variables)
    1. [Add helm repositories](#add-helm-repositories)
    1. [Initialize database](#initialize-database)
    1. [Install mock-data-generator](#install-mock-data-generator)
    1. [Install stream-loader](#install-stream-loader)
    1. [Install senzing-api-server](#install-senzing-api-server)
    1. [Test Senzing REST API server](#test-senzing-rest-api-server)
1. [Cleanup](#cleanup)
    1. [Delete everything in project](#delete-everything-in-project)

## Expectations

### Space

This repository and demonstration require 20 GB free disk space.

### Time

Budget 4 hours to get the demonstration up-and-running, depending on CPU and network speeds.

### Background knowledge

This repository assumes a working knowledge of:

1. [Docker](https://github.com/Senzing/knowledge-base/blob/master/WHATIS/docker.md)
1. [Kubernetes](https://github.com/Senzing/knowledge-base/blob/master/WHATIS/kubernetes.md)
1. [Helm](https://github.com/Senzing/knowledge-base/blob/master/WHATIS/helm.md)
1. [IBM ICP4D](https://github.com/Senzing/knowledge-base/blob/master/WHATIS/ibm-icp4d.md)

## Demonstrate

### Clone repository

The Git repository has files that will be used in the `helm install --values` parameter.

1. Using these environment variable values:

    ```console
    export GIT_ACCOUNT=senzing
    export GIT_REPOSITORY=ibm-icp4d-guide
    ```

1. Follow steps in [clone-repository](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/clone-repository.md) to install the Git repository.

1. After the Git repository has been cloned, be sure the following environment variables are set:

    ```console
    export GIT_ACCOUNT_DIR=~/${GIT_ACCOUNT}.git
    export GIT_REPOSITORY_DIR="${GIT_ACCOUNT_DIR}/${GIT_REPOSITORY}"
    ```

### Prerequisites

#### Docker images

1. **FIXME:**  Describe how to accept terms and conditions for the senzing/senzing-package docker image.

#### Initialize database

1. If needed, create a database for Senzing data. Example:

    ```console
    su - db2inst1
    db2 create database g2 using codeset utf-8 territory us
    ```

1. **FIXME:** Craft instructions on how to obtain `g2core-schema-db2-create.sql`.

1. Create tables in the database. Example:

    ```console
    su - db2inst1
    db2 connect to g2
    db2 -tf g2core-schema-db2-create.sql
    db2 connect reset
    ```

#### Database connection information

1. Craft the `SENZING_DATABASE_URL`.  It will be used in "helm values" files.

    ```console
    export DATABASE_USERNAME=my-username
    export DATABASE_PASSWORD=my-password
    export DATABASE_HOST=my.database.com
    export DATABASE_PORT=50000
    export DATABASE_DATABASE=G2

    export SENZING_DATABASE_URL="db2://${DATABASE_USERNAME}:${DATABASE_PASSWORD}@${DATABASE_HOST}:${DATABASE_PORT}/${DATABASE_DATABASE}"

    echo ${SENZING_DATABASE_URL}
    ```

#### Kafka connection information

1. **FIXME:**

    ```console
    echo ${KAFKA_HOST}
    ```

### Set environment variables

1. Environment variables that need customization.  Example:

    ```console
    export DEMO_NAMESPACE=zen
    ```

1. Identify Helm values files location. Example:

    ```console
    export HELM_VALUES_DIR=${GIT_REPOSITORY_DIR}/helm-values
    ```

1. If you are using Transport Layer Security (TLS), then set the following environment variable:

    ```console
    export HELM_TLS="--tls"
    ```

### Add helm repositories

1. Add Senzing Helm repository.  Example:

    ```console
    helm repo add senzing https://senzing.github.io/charts/
    ```

1. Update repositories.

    ```console
    helm repo update
    ```

1. Review repositories.

    ```console
    helm repo list
    ```

1. Optional:  View Senzing Helm charts in repository.

    ```console
    helm repo search senzing/
    ```

1. References:
    1. [helm repo](https://helm.sh/docs/helm/#helm-repo)
    1. [Senzing charts](https://github.com/Senzing/charts)

### Deploy Senzing_API.tgz package

1. Deploy the contents of
   [Senzing_API.tgz](https://github.com/Senzing/knowledge-base/blob/master/WHATIS/senzing-api-tgz.md)
   onto a Kubernetes Persistent Volume.
    1. References:
        1. [GitHub repository](https://github.com/Senzing/senzing-package)
        1. [Helm chart](https://github.com/Senzing/charts/tree/master/charts/senzing-package)
        1. [Docker](https://hub.docker.com/r/senzing/senzing-package)

1. Review helm values in `${HELM_VALUES_DIR}/senzing-package.yaml`.
    1. `senzing.optSenzingClaim` is the Persistent Volume Claim for use by Senzing as `/opt/senzing`.

1. Review helm values in `${HELM_VALUES_DIR}/senzing-package-sleep.yaml`.
    1. `senzing.optSenzingClaim` is the Persistent Volume Claim for use by Senzing as `/opt/senzing`.

1. Perform Helm install. Example:

    ```console
    helm install ${HELM_TLS} \
      --name senzing-package \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/senzing-package.yaml \
      senzing/senzing-package
    ```

1. To inspect the `/opt/senzing` volume, run a second copy in "sleep" mode. Example:

    ```console
    helm install ${HELM_TLS} \
      --name senzing-package-sleep \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/senzing-package-sleep.yaml \
      senzing/senzing-package
    ```

    ```console
    kubectl get pods --namespace ${DEMO_NAMESPACE}

    export POD_NAME=my-senzing-package-sleep-XXXXXX
    kubectl exec -it --namespace ${DEMO_NAMESPACE} ${POD_NAME} -- /bin/bash
    ```

### Install mock-data-generator

1. This component reads JSON LINES from file and pushes to Kafka.
More details in the [GitHub repository](https://github.com/Senzing/mock-data-generator).
    1. References:
        1. [GitHub repository](https://github.com/Senzing/mock-data-generator)
        1. [Helm chart](https://github.com/Senzing/charts/tree/master/charts/senzing-mock-data-generator)
        1. [Docker](https://hub.docker.com/r/senzing/mock-data-generator)

1. Review helm values in `${HELM_VALUES_DIR}/mock-data-generator.yaml`.
    1. `senzing.kafkaBootstrapServerHost` is the value of ${KAFKA_HOST}.
    1. `senzing.inputUrl` is a URL addressable file of JSON LINES. (e.g. `file://`, `http://`).
    1. `senzing.recordMax` is the maximum number of JSON LINES to read from the file.  Remove or 0 to read all lines.

1. Perform Helm install. Example:

    ```console
    helm install ${HELM_TLS} \
      --name senzing-mock-data-generator \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/mock-data-generator.yaml \
      senzing/senzing-mock-data-generator
    ```

### Install stream-loader

1. This component reads from a Kafka topic and sends to Senzing which populates the DB2 database.
More details in the [GitHub repository](https://github.com/Senzing/stream-loader).
    1. References:
        1. [GitHub repository](https://github.com/Senzing/stream-loader)
        1. [Helm chart](https://github.com/Senzing/charts/tree/master/charts/senzing-stream-loader)
        1. [Docker](https://hub.docker.com/r/senzing/stream-loader)

1. Review helm values in `${HELM_VALUES_DIR}/stream-loader.yaml`.
    1. `senzing.databaseUrl` is the value of ${SENZING_DATABASE_URL}.
    1. `senzing.kafkaBootstrapServerHost` is the value of ${KAFKA_HOST}.
    1. `senzing.optSenzingClaim` is the Persistent Volume Claim for use by Senzing as `/opt/senzing`.

1. Example:

    ```console
    helm install ${HELM_TLS} \
      --name senzing-stream-loader \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/stream-loader-db2.yaml \
      senzing/senzing-stream-loader
    ```

### Install senzing-api-server

1. This component creates an HTTP service that implements the
[Senzing REST API](https://github.com/Senzing/senzing-rest-api).
More details in the [GitHub repository](https://github.com/Senzing/senzing-api-server).
    1. References:
        1. [GitHub repository](https://github.com/Senzing/senzing-api-server)
        1. [Helm chart](https://github.com/Senzing/charts/tree/master/charts/senzing-api-server)
        1. [Docker](https://hub.docker.com/r/senzing/senzing-api-server)

1. Example:

    ```console
    helm install ${HELM_TLS} \
      --name senzing-api-server \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/senzing-api-server-db2.yaml \
      senzing/senzing-api-server
    ```

1. Wait for pods to run. Example:

    ```console
    watch -n 5 -d kubectl get pods --namespace ${DEMO_NAMESPACE}
    ```

1. Port forward to local machine.  Run in a separate terminal window. Example:

    ```console
    export DEMO_NAMESPACE=zen

    kubectl port-forward \
      --address 0.0.0.0 \
      --namespace ${DEMO_NAMESPACE} \
      svc/${DEMO_PREFIX}-senzing-api-server 8889:80
    ```

### Test Senzing REST API server

*Note:* port 8889 on the localhost has been mapped to port 80 in the docker container.
See `kubectl port-forward ...` above.

1. Example:

    ```console
    export SENZING_API_SERVICE=http://localhost:8889

    curl -X GET ${SENZING_API_SERVICE}/heartbeat
    curl -X GET ${SENZING_API_SERVICE}/license
    curl -X GET ${SENZING_API_SERVICE}/entities/1
    ```

## Cleanup

### Delete everything in project

1. Example:

    ```console
    helm delete ${HELM_TLS} --purge ${DEMO_PREFIX}-senzing-api-server
    helm delete ${HELM_TLS} --purge ${DEMO_PREFIX}-senzing-stream-loader
    helm delete ${HELM_TLS} --purge ${DEMO_PREFIX}-senzing-mock-data-generator
    helm delete ${HELM_TLS} --purge ${DEMO_PREFIX}-senzing-package-sleep
    helm delete ${HELM_TLS} --purge ${DEMO_PREFIX}-senzing-package
    helm repo remove senzing
    helm repo remove bitnami
    ```
