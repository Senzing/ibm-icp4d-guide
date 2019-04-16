# ibm-icp4d-guide

## Overview

This repository illustrates a reference implementation of Senzing on the IBM Cloud Private for Data

The following diagram shows the relationship of the Helm charts, docker containers, and code in this IBM Cloud Private for Data demonstration.

![Image of architecture](docs/architecture.png)

### Contents

1. [Expectations](#expectations)
    1. [Space](#space)
    1. [Time](#time)
    1. [Background knowledge](#background-knowledge)
1. [Prerequisites](#prerequisites)
    1. [Clone repository](#clone-repository)
    1. [Docker images](#docker-images)
    1. [Kafka initialization](#kafka-initialization)
    1. [Database initialization](#database-initialization)
    1. [Database connection information](#database-connection-information)
1. [Demonstrate](#demonstrate)
    1. [Set environment variables](#set-environment-variables)
    1. [Add helm repositories](#add-helm-repositories)
    1. [Deploy Senzing_API.tgz package](#deploy-senzing_apitgz-package)
    1. [Install mock-data-generator Helm chart](#install-mock-data-generator-helm-chart)
    1. [Install stream-loader Helm chart](#install-stream-loader-helm-chart)
    1. [Install senzing-api-server Helm chart](#install-senzing-api-server-helm-chart)
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

## Prerequisites

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

### Docker images

1. **FIXME:**  Describe how to accept terms and conditions for the senzing/senzing-package docker image.

### Kafka initialization

1. Find pod running kafka. Example:

    ```console
    kubectl get pods -n zen | grep kafka
    ```

1. Enter the pod. Example:

    ```console
    kubectl exec -it kafka-0 -n zen bash
    ```

1. Within the Kafka pod, create Kafka topic for Senzing.  Example:

    ```console
    /opt/kafka/bin/kafka-topics.sh \
      --create \
      --zookeeper zookeeper:2181/kafka \
      --replication-factor 1 \
      --partitions 15 \
      --topic senzing-kafka-topic
    ```

1. Within the Kafka pod, verify the Kafka topic has been created.  Example:

    ```console
    /opt/kafka/bin/kafka-topics.sh \
      --list \
      --zookeeper zookeeper:2181/kafka
    ```

1. Outside of the Kafka pod, find the running Kafka services. Example:

    ```console
    kubectl get service --namespace zen | grep kafka
    ```

1. Identify the service host with type of `NodePort`.  Example:

    ```console
    export KAFKA_HOST=kafka-external

    echo ${KAFKA_HOST}
    ```

### Database initialization

#### Obtain correct SQL file

1. For **IBM Db2** use one of these techniques:
    1. In local git clone at `${GIT_REPOSITORY_DIR}/sql/g2core-schema-db2-create.sql`
    1. [On GitHub](https://github.com/Senzing/ibm-icp4d-guide/blob/issue-1.dockter.1/sql/g2core-schema-db2-create.sql)
    1. Using `curl`. Example:

        ```console
        curl -X GET \
          --output /tmp/g2core-schema-db2-create.sql \
          https://raw.githubusercontent.com/Senzing/ibm-icp4d-guide/issue-1.dockter.1/sql/g2core-schema-db2-create.sql
        ```

1. For **IBM Db2 BLU** use one of these techniques:
    1. In local git clone at `${GIT_REPOSITORY_DIR}/sql/g2core-schema-db2-BLU-create.sql`
    1. [On GitHub](https://github.com/Senzing/ibm-icp4d-guide/blob/issue-1.dockter.1/sql/g2core-schema-db2-BLU-create.sql)
    1. Using `curl`. Example:

        ```console
        curl -X GET \
          --output /tmp/g2core-schema-db2-BLU-create.sql \
          https://raw.githubusercontent.com/Senzing/ibm-icp4d-guide/issue-1.dockter.1/sql/g2core-schema-db2-BLU-create.sql
        ```

#### Run SQL file 

1. Variation #1. Create tables in the database using command line. Example:

    1. If needed, create a database for Senzing data. Example:

        ```console
        su - db2inst1
        db2 create database g2 using codeset utf-8 territory us
        ```

    1. Create tables in schema.  Example:

        ```console
        su - db2inst1
        db2 connect to g2
        db2 -tf g2core-schema-db2-create.sql
        db2 connect reset
        ```

1. Variation #2.  (FIXME:) Using the IBM Cloud Private for Data console with DB2 Advanced ...
    1. Home > My data > Databases
        1. Open tile for desired database
        1. Click on the ellipse, click on "Open"
        1. Menu > Run SQL
        1. Click on plus sign ("+") to add
        1. Select appropriate Schema (check the box)
        1. From file
        1. In file browser, navigate to SQL file
        1. Click "Run all" button

### Database connection information

1. Craft the `SENZING_DATABASE_URL`.  It will be used in "helm values" files.

    Components of the URL:

    ```console
    export DATABASE_USERNAME=<my-username>
    export DATABASE_PASSWORD=<my-password>
    export DATABASE_HOST=<hostname>
    export DATABASE_PORT=<db2-connnection-port>
    export DATABASE_DATABASE=<database-name>
    ```

    Example:

    ```console
    export DATABASE_USERNAME=johnsmith
    export DATABASE_PASSWORD=secret
    export DATABASE_HOST=my.database.com
    export DATABASE_PORT=50000
    export DATABASE_DATABASE=G2

    export SENZING_DATABASE_URL="db2://${DATABASE_USERNAME}:${DATABASE_PASSWORD}@${DATABASE_HOST}:${DATABASE_PORT}/${DATABASE_DATABASE}"

    echo ${SENZING_DATABASE_URL}
    ```

1. FIXME: In the user interface, (show how to find database credentials)
    1. Details
    1. Bottom
    1. "Access Information" section

## Demonstrate

### Set environment variables

1. Environment variables that need customization.  Example:

    ```console
    export DEMO_NAMESPACE=zen
    ```

1. If using Transport Layer Security (TLS), then set the following environment variable:

    ```console
    export HELM_TLS="--tls"
    ```

### Add helm repositories

1. Add Senzing Helm repository.  Example:

    ```console
    helm repo add senzing https://senzing.github.io/charts/
    ```

1. Optional:  View Senzing Helm charts in repository. Example:

    ```console
    helm search senzing/
    ```

1. References:
    1. [Senzing charts](https://github.com/Senzing/charts)

### Deploy Senzing_API.tgz package

1. Deploy the contents of
   [Senzing_API.tgz](https://github.com/Senzing/knowledge-base/blob/master/WHATIS/senzing-api-tgz.md)
   onto a Kubernetes Persistent Volume.

   References:
    1. [GitHub repository](https://github.com/Senzing/senzing-package)
    1. [Helm chart](https://github.com/Senzing/charts/tree/master/charts/senzing-package)
    1. [Docker](https://hub.docker.com/r/senzing/senzing-package)

1. Review helm values in `${GIT_REPOSITORY_DIR}/helm-values/senzing-package.yaml`.
    1. `senzing.optSenzingClaim` is the Persistent Volume Claim for use by Senzing as `/opt/senzing`.

1. Perform Helm install. Example:

    ```console
    helm install ${HELM_TLS} \
      --name senzing-package \
      --namespace ${DEMO_NAMESPACE} \
      --values ${GIT_REPOSITORY_DIR}/helm-values/senzing-package.yaml \
      senzing/senzing-package
    ```

1.  **Important:** Wait until job runs to completion before continuing.
    This will take a few minutes.
    Example of completed job:
    
    ```console
    # kubectl get pods --namespace ${DEMO_NAMESPACE}  | grep senzing
    
    senzing-package-r6z86                                             0/1     Completed   0          4m29s    
    ```

    Note that the job took four and a half minutes to complete.

1. Optional: To inspect the `/opt/senzing` volume, run a Senzing debug image.

    1. Review helm values in `${GIT_REPOSITORY_DIR}/helm-values/senzing-debug.yaml`.
        1. `senzing.databaseUrl` is the value of ${SENZING_DATABASE_URL}.
        1. `senzing.optSenzingClaim` is the Persistent Volume Claim for use by Senzing as `/opt/senzing`.

    1. Install chart.  Example:

        ```console
        helm install ${HELM_TLS} \
          --name senzing-debug \
          --namespace ${DEMO_NAMESPACE} \
          --values ${GIT_REPOSITORY_DIR}/helm-values/senzing-debug.yaml \
          senzing/senzing-debug
        ```

    1. Find and enter pod.  Example:

        ```console
        kubectl get pods --namespace ${DEMO_NAMESPACE}

        export POD_NAME=senzing-debug-XXXXXX
        kubectl exec -it --namespace ${DEMO_NAMESPACE} ${POD_NAME} -- /bin/bash
        ```

### Install Senzing license

This is an optional step.
Senzing comes with a trial license that supports 10,000 entities.


1. If working with more entities,
   [obtain a Senzing license](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/obtain-senzing-license.md)

1. With the `g2.lic` file, ...

1. FIXME: 




### Install mock-data-generator Helm chart

:warning:  **FIXME:**  This is a **mock** data generator.
In production, this component is replaced by
different components that feed Kafka.

1. This component reads JSON LINES from a URL-addressable file and pushes to Kafka.

   References:
    1. [GitHub repository](https://github.com/Senzing/mock-data-generator)
    1. [Helm chart](https://github.com/Senzing/charts/tree/master/charts/senzing-mock-data-generator)
    1. [Docker](https://hub.docker.com/r/senzing/mock-data-generator)

1. Review helm values in `${GIT_REPOSITORY_DIR}/helm-values/mock-data-generator.yaml`.
    1. `senzing.kafkaBootstrapServerHost` is the value of ${KAFKA_HOST}.
    1. `senzing.inputUrl` is a URL addressable file of JSON LINES. (e.g. `file://`, `http://`).
    1. `senzing.recordMax` is the maximum number of JSON LINES to read from the file.  Remove or 0 to read all lines.

1. Perform Helm install. Example:

    ```console
    helm install ${HELM_TLS} \
      --name senzing-mock-data-generator \
      --namespace ${DEMO_NAMESPACE} \
      --values ${GIT_REPOSITORY_DIR}/helm-values/mock-data-generator.yaml \
      senzing/senzing-mock-data-generator
    ```

### Install stream-loader Helm chart

1. This component reads messages from a Kafka topic and sends them to Senzing which populates the DB2 database.

   References:
    1. [GitHub repository](https://github.com/Senzing/stream-loader)
    1. [Helm chart](https://github.com/Senzing/charts/tree/master/charts/senzing-stream-loader)
    1. [Docker](https://hub.docker.com/r/senzing/stream-loader)

1. Review helm values in `${GIT_REPOSITORY_DIR}/helm-values/stream-loader.yaml`.
    1. `senzing.databaseUrl` is the value of ${SENZING_DATABASE_URL}.
    1. `senzing.kafkaBootstrapServerHost` is the value of ${KAFKA_HOST}.
    1. `senzing.optSenzingClaim` is the Persistent Volume Claim for use by Senzing as `/opt/senzing`.

1. Perform Helm install. Example:

    ```console
    helm install ${HELM_TLS} \
      --name senzing-stream-loader \
      --namespace ${DEMO_NAMESPACE} \
      --values ${GIT_REPOSITORY_DIR}/helm-values/stream-loader.yaml \
      senzing/senzing-stream-loader
    ```

### Install senzing-api-server Helm chart

1. This component creates an HTTP service that implements the
[Senzing REST API](https://github.com/Senzing/senzing-rest-api).

   References:
    1. [GitHub repository](https://github.com/Senzing/senzing-api-server)
    1. [Helm chart](https://github.com/Senzing/charts/tree/master/charts/senzing-api-server)
    1. [Docker](https://hub.docker.com/r/senzing/senzing-api-server)

1. Review helm values in `${GIT_REPOSITORY_DIR}/helm-values/senzing-api-server`.
    1. `senzing.databaseUrl` is the value of ${SENZING_DATABASE_URL}.
    1. `senzing.optSenzingClaim` is the Persistent Volume Claim for use by Senzing as `/opt/senzing`.

1. Perform Helm install. Example:

    ```console
    helm install ${HELM_TLS} \
      --name senzing-api-server \
      --namespace ${DEMO_NAMESPACE} \
      --values ${GIT_REPOSITORY_DIR}/helm-values/senzing-api-server.yaml \
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
      --namespace ${DEMO_NAMESPACE} \
      svc/senzing-api-server 8889:80
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


#### Delete Helm charts

1. Example:

    ```console
    helm delete ${HELM_TLS} --purge senzing-api-server
    helm delete ${HELM_TLS} --purge senzing-stream-loader
    helm delete ${HELM_TLS} --purge senzing-mock-data-generator
    helm delete ${HELM_TLS} --purge senzing-debug
    helm delete ${HELM_TLS} --purge senzing-package
    helm repo remove senzing
    ```

#### Delete Kafka topic

1. FIXME: Example:

1. Find pod running kafka. Example:

    ```console
    kubectl get pods -n zen | grep kafka
    ```

1. Enter the pod. Example:

    ```console
    kubectl exec -it kafka-0 -n zen bash
    ```

1. Within the Kafka pod, create Kafka topic for Senzing.  Example:

    ```console
    /opt/kafka/bin/kafka-topics.sh \
      --delete \
      --zookeeper zookeeper:2181/kafka \
      --topic senzing-kafka-topic
    ```

#### FIXME: Delete database.

1. Example:
