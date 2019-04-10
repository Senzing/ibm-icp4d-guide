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
    1. [Install Kafka](#install-kafka)
    1. [Install Kafka test client](#install-kafka-test-client)
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

## Demonstrate

### Clone repository

1. Using these environment variable values:

    ```console
    export GIT_ACCOUNT=senzing
    export GIT_REPOSITORY=ibm-icp4d-guide
    ```

1. Then follow steps in [clone-repository](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/clone-repository.md).

1. After the repository has been cloned, be sure the following are set:

    ```console
    export GIT_ACCOUNT_DIR=~/${GIT_ACCOUNT}.git
    export GIT_REPOSITORY_DIR="${GIT_ACCOUNT_DIR}/${GIT_REPOSITORY}"
    ```

### Prerequisites

#### Docker images

1. **FIXME:**  Describe how to accept terms and conditions for the senzing/senzing-package docker image.

### Set environment variables

1. Environment variables that need customization.  Example:

    ```console
    export DEMO_NAMESPACE=zen
    ```

1. Set environment variables listed in "[Clone repository](#clone-repository)".

### Using Transport Layer Security

1. If you are using Transport Layer Security (TLS), then set the following environment variable:

    ```console
    export HELM_TLS="--tls"
    ```

### Add helm repositories

1. Add Bitnami repository. Example:

    ```console
    helm repo add bitnami https://charts.bitnami.com
    ```

1. Add Senzing repository.  Example:

    ```console
    helm repo add senzing https://senzing.github.io/charts/
    ```

1. Update repositories.

    ```console
    helm repo update
    ```

1. Review repositories

    ```console
    helm repo list
    ```

1. Reference: [helm repo](https://helm.sh/docs/helm/#helm-repo)

### Deploy Senzing_API.tgz package

1. Example:

    ```console
    helm install ${HELM_TLS} \
      --name ${DEMO_PREFIX}-senzing-package \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/senzing-package.yaml \
      senzing/senzing-package
    ```

1. To inspect the `/opt/senzing` volume, run a second copy in "sleep" mode. Example:

    ```console
    helm install ${HELM_TLS} \
      --name ${DEMO_PREFIX}-senzing-package-sleep \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/senzing-package-sleep.yaml \
      senzing/senzing-package
    ```

    ```console
    kubectl get pods --namespace ${DEMO_NAMESPACE}

    export POD_NAME=my-senzing-package-sleep-XXXXXX
    kubectl exec -it --namespace ${DEMO_NAMESPACE} ${POD_NAME} -- /bin/bash
    ```

### Initialize database

1. Using the directions shown in the output from the previous step,
   log into the IBM DB2 Express-C container.

1. In the IBM DB2 Express-C container, run the following:

    ```console
    su - db2inst1
    db2 create database g2 using codeset utf-8 territory us
    db2 connect to g2
    db2 -tf /opt/senzing/g2/data/g2core-schema-db2-create.sql
    db2 connect reset
    exit
    exit
    ```

### Install Kafka

1. Example:

    ```console
    helm install ${HELM_TLS} \
      --name ${DEMO_PREFIX}-kafka \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/kafka.yaml \
      bitnami/kafka
    ```

### Install Kafka test client

1. Install Kafka test client app. Example:

    ```console
    helm install ${HELM_TLS} \
      --name ${DEMO_PREFIX}-kafka-test-client \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/kafka-test-client.yaml \
      senzing/kafka-test-client
    ```

1. Wait for pods to run. Example:

    ```console
    watch -n 5 -d kubectl get pods --namespace ${DEMO_NAMESPACE}
    ```

1. Run the test client. Run in a separate terminal window. Example:

    ```console
    export DEMO_PREFIX=my
    export DEMO_NAMESPACE=${DEMO_PREFIX}-namespace
    export POD_NAME=$(kubectl get pods --namespace ${DEMO_NAMESPACE}-namespace -l "app.kubernetes.io/name=kafka-test-client,app.kubernetes.io/instance=${DEMO_NAMESPACE}-kafka-test-client" -o jsonpath="{.items[0].metadata.name}")

    kubectl exec \
      -it \
      --namespace ${DEMO_NAMESPACE} \
      ${POD_NAME} -- /usr/bin/kafka-console-consumer \
        --bootstrap-server ${DEMO_PREFIX}-kafka:9092 \
        --topic senzing-kafka-topic \
        --from-beginning
    ```  

### Install mock-data-generator

1. Example:

    ```console
    helm install ${HELM_TLS} \
      --name ${DEMO_PREFIX}-senzing-mock-data-generator \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/mock-data-generator.yaml \
      senzing/senzing-mock-data-generator
    ```

### Install stream-loader

1. Example:

    ```console
    helm install ${HELM_TLS} \
      --name ${DEMO_PREFIX}-senzing-stream-loader \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/stream-loader-db2.yaml \
      senzing/senzing-stream-loader
    ```

### Install senzing-api-server

1. Example:

    ```console
    helm install ${HELM_TLS} \
      --name ${DEMO_PREFIX}-senzing-api-server \
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
    export DEMO_PREFIX=my
    export DEMO_NAMESPACE=${DEMO_PREFIX}-namespace

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
    helm delete ${HELM_TLS} --purge ${DEMO_PREFIX}-kafka-test-client
    helm delete ${HELM_TLS} --purge ${DEMO_PREFIX}-senzing-package-sleep
    helm delete ${HELM_TLS} --purge ${DEMO_PREFIX}-senzing-package
    helm repo remove senzing
    helm repo remove bitnami
    ```
