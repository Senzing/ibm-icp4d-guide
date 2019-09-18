# ibm-icp4d-guide-helm-rabbitmq-db2

## Overview

This repository illustrates a reference implementation of Senzing on the IBM Cloud Pak for Data.

The instructions show how to set up a system that:

1. Reads JSON lines from a file on the internet.
1. Sends each JSON line to a message queue.
    1. In this implementation, the queue is RabbitMQ.
1. Reads messages from the queue and inserts into Senzing.
    1. In this implementation, Senzing keeps its data in an IBM Db2 database.
1. Reads information from Senzing via [Senzing REST API](https://github.com/Senzing/senzing-rest-api) server.

The following diagram shows the relationship of the Helm charts, docker containers, and code in this IBM Cloud Pak for Data reference implementation.

![Image of architecture](architecture.png)

### Contents

1. [Expectations](#expectations)
    1. [Space](#space)
    1. [Time](#time)
    1. [Background knowledge](#background-knowledge)
1. [Prerequisites](#prerequisites)
    1. [Clone repository](#clone-repository)
    1. [EULA](#eula)
    1. [Set environment variables](#set-environment-variables)
    1. [Database connection information](#database-connection-information)
    1. [Create custom Helm values files](#create-custom-helm-values-files)
    1. [Create custom kubernetes configuration files](#create-custom-kubernetes-configuration-files)
1. [Demonstrate](#demonstrate)
    1. [Create namespace](#create-namespace)
    1. [Create Persistent Volume Claims](#create-persistent-volume-claims)
    1. [Add Helm repositories](#add-helm-repositories)
    1. [Registry authorization](#registry-authorization)
    1. [Deploy Senzing RPM](#deploy-senzing-rpm)
    1. [Install IBM Db2 Driver Helm chart](#install-ibm-db2-driver-helm-chart)
    1. [Install senzing-debug Helm Chart](#install-senzing-debug-helm-chart)
    1. [Install Senzing license](#install-senzing-license)
    1. [Get Senzing schema sql for Db2](#get-senzing-schema-sql-for-db2)
    1. [Create Senzing schema on Db2](#create-senzing-schema-on-db2)
    1. [Database tuning](#database-tuning)
    1. [Optional TLS enablement](#optional-tls-enablement)
    1. [Install RabbitMQ Helm Chart](#install-rabbitmq-helm-chart)
    1. [Install mock-data-generator Helm chart](#install-mock-data-generator-helm-chart)
    1. [Install init-container Helm chart](#install-init-container-helm-chart)
    1. [Install configurator Helm chart](#install-configurator-helm-chart)
    1. [Install stream-loader Helm chart](#install-stream-loader-helm-chart)
    1. [Install senzing-api-server Helm chart](#install-senzing-api-server-helm-chart)
    1. [Install senzing-entity-search-web-app Helm chart](#install-senzing-entity-search-web-app-helm-chart)
    1. [View data](#view-data)
1. [Cleanup](#cleanup)
    1. [Delete Helm charts](#delete-helm-charts)
    1. [Delete database tables](#delete-database-tables)
    1. [Delete git repository](#delete-git-repository)

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
    export GIT_ACCOUNT_DIR=~/${GIT_ACCOUNT}.git
    export GIT_REPOSITORY_DIR="${GIT_ACCOUNT_DIR}/${GIT_REPOSITORY}"
    ```

1. Follow steps in [clone-repository](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/clone-repository.md) to install the Git repository.

### EULA

To use the Senzing code, you must agree to the End User License Agreement (EULA).

1. :warning: This step is intentionally tricky and not simply copy/paste.
   This ensures that you make a conscious effort to accept the EULA.
   See
   [SENZING_ACCEPT_EULA](https://github.com/Senzing/knowledge-base/blob/master/lists/environment-variables.md#senzing_accept_eula)
   for the correct value.
   Replace the double-quote character in the example with the correct value.
   The use of the double-quote character is intentional to prevent simple copy/paste.
   Example:

    ```console
    export SENZING_ACCEPT_EULA="
    ```

### Set environment variables

1. :pencil2: Environment variables that need customization.
   Example:

    ```console
    export DEMO_PREFIX=my
    export DEMO_NAMESPACE=senzing

    export DOCKER_REGISTRY_URL=docker.io
    export DOCKER_REGISTRY_SECRET=${DOCKER_REGISTRY_URL}-secret
    ```

1. :thinking: **Optional:** If using Transport Layer Security (TLS),
   then set the following environment variable:

    ```console
    export HELM_TLS="--tls"
    ```

1. Set environment variables listed in "[Clone repository](#clone-repository)".

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

    :pencil2: Set environment variables.  Example:

    ```console
    export DATABASE_USERNAME=johnsmith
    export DATABASE_PASSWORD=secret
    export DATABASE_HOST=my.database.com
    export DATABASE_PORT=50000
    export DATABASE_DATABASE=G2
    ```

    Construct database URL.  Example:

    ```console
    export SENZING_DATABASE_URL="db2://${DATABASE_USERNAME}:${DATABASE_PASSWORD}@${DATABASE_HOST}:${DATABASE_PORT}/${DATABASE_DATABASE}"

    echo ${SENZING_DATABASE_URL}
    ```

### Create custom Helm values files

:thinking: In this step, Helm template files are populated with actual values.
There are two methods of accomplishing this.
Only one method needs to be performed.

1. **Method #1:** Quick method using `envsubst`.
   Example:

    ```console
    export HELM_VALUES_DIR=${GIT_REPOSITORY_DIR}/helm-values
    mkdir -p ${HELM_VALUES_DIR}

    for file in ${GIT_REPOSITORY_DIR}/helm-values-templates/*.yaml; \
    do \
      envsubst < "${file}" > "${HELM_VALUES_DIR}/$(basename ${file})";
    done
    ```

1. **Method #2:** Copy and manually modify files method.
   Example:

    ```console
    export HELM_VALUES_DIR=${GIT_REPOSITORY_DIR}/helm-values
    mkdir -p ${HELM_VALUES_DIR}

    cp ${GIT_REPOSITORY_DIR}/helm-values-templates/* ${HELM_VALUES_DIR}
    ```

    :pencil2: Edit files in ${HELM_VALUES_DIR} replacing the following variables with actual values.

    1. `${DEMO_PREFIX}`
    1. `${DOCKER_REGISTRY_SECRET}`
    1. `${DOCKER_REGISTRY_URL}`
    1. `${SENZING_ACCEPT_EULA}`
    1. `${SENZING_DATABASE_URL}`

### Create custom kubernetes configuration files

:thinking: In this step, Kubernetes template files are populated with actual values.
There are two methods of accomplishing this.
Only one method needs to be performed.

1. **Method #1:** Quick method using `envsubst`.
   Example:

    ```console
    export KUBERNETES_DIR=${GIT_REPOSITORY_DIR}/kubernetes
    mkdir -p ${KUBERNETES_DIR}

    for file in ${GIT_REPOSITORY_DIR}/kubernetes-templates/*; \
    do \
      envsubst < "${file}" > "${KUBERNETES_DIR}/$(basename ${file})";
    done
    ```

1. **Method #2:** Copy and manually modify files method.
   Example:

    ```console
    export KUBERNETES_DIR=${GIT_REPOSITORY_DIR}/kubernetes
    mkdir -p ${KUBERNETES_DIR}

    cp ${GIT_REPOSITORY_DIR}/kubernetes-templates/* ${KUBERNETES_DIR}
    ```

    :pencil2: Edit files in ${KUBERNETES_DIR} replacing the following variables with actual values.

    1. `${DEMO_NAMESPACE}`

## Demonstrate

### Create namespace

1. Create namespace.
   Example:

    ```console
    kubectl create -f ${KUBERNETES_DIR}/namespace.yaml
    ```

1. :thinking: **Optional:**
   Review namespaces.

    ```console
    kubectl get namespaces
    ```

### Create Persistent Volume Claims

1. Create Persistent Volume Claim (PVC).
   Example:

    ```console
    kubectl create \
      --filename ${KUBERNETES_DIR}/persistent-volume-claim-senzing.yaml

    kubectl create \
      --filename ${KUBERNETES_DIR}/persistent-volume-claim-rabbitmq.yaml
    ```

1. :thinking: **Optional:**
   Review persistent volumes and claims.

    ```console
    kubectl get persistentvolumes \
      --namespace ${DEMO_NAMESPACE}

    kubectl get persistentvolumeClaims \
      --namespace ${DEMO_NAMESPACE}
    ```

### Add Helm repositories

1. Add Senzing repository.
   Example:

    ```console
    helm repo add senzing https://senzing.github.io/charts/
    ```

1. Update repositories.
   Example:

    ```console
    helm repo update
    ```

1. :thinking: **Optional:**
   Review repositories.
   Example:

    ```console
    helm repo list
    ```

1. Reference: [helm repo](https://helm.sh/docs/helm/#helm-repo)

### Registry authorization

1. To enable the ICP4D image policy that allows pulling from `docker.io`,
   visit the "Customizing your policy (post installation)" section of
   [Enforcing container image security](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.0/manage_images/image_security.html).

1. :pencil2: Review and potentially modify `${KUBERNETES_DIR}/enable-docker-io.yaml`

1. Apply policy.  Example:

    ```console
    kubectl apply \
      --namespace ${DEMO_NAMESPACE} \
      --filename ${KUBERNETES_DIR}/enable-docker-io.yaml
    ```

### Deploy Senzing RPM

This deployment initializes the Persistent Volume with Senzing code and data.

1. Install chart.
   Example:

    ```console
    helm install ${HELM_TLS} \
      --name ${DEMO_PREFIX}-senzing-yum \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/senzing-yum.yaml \
      senzing/senzing-yum
    ```

### Install IBM Db2 Driver Helm chart

This deployment adds the IBM Db2 Client driver code to the Persistent Volume.

1. Install chart.
   Example:

    ```console
    helm install ${HELM_TLS} \
      --name ${DEMO_PREFIX}-ibm-db2-driver-installer \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/ibm-db2-driver-installer.yaml \
      senzing/ibm-db2-driver-installer
    ```

### Install senzing-debug Helm chart

This deployment will be used later to:

- Inspect mounted volumes
- Copy files onto the Persistent Volume
- Debug issues

1. Install chart.  Example:

    ```console
    helm install ${HELM_TLS} \
      --name senzing-debug \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/senzing-debug.yaml \
       senzing/senzing-debug
    ```

1. Wait for pods to run.
   Example:

    ```console
    kubectl get pods \
      --namespace ${DEMO_NAMESPACE} \
      --watch
    ```

1. In a separate terminal window, log into debug pod.

    :pencil2:  Set environment variables.  Example:

    ```console
    export DEMO_NAMESPACE=senzing
    ```

    Log into pod.  Example:

    ```console
    export DEBUG_POD_NAME=$(kubectl get pods \
      --namespace ${DEMO_NAMESPACE} \
      --output jsonpath="{.items[0].metadata.name}" \
      --selector "app.kubernetes.io/name=senzing-debug, \
                  app.kubernetes.io/instance=senzing-debug" \
      )

    kubectl exec -it --namespace ${DEMO_NAMESPACE} ${DEBUG_POD_NAME} -- /bin/bash
    ```

1. To use senzing-debug pod, see [View Senzing Debug pod](#view-senzing-debug-pod).

### Install Senzing license

:thinking: **Optional:**
Senzing comes with a trial license that supports 10,000 records.
If this is sufficient, there is no need to install a new license
and this step may be skipped.

1. If working with more than 10,000 records,
   [obtain a Senzing license](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/obtain-senzing-license.md).

1. Be sure the `senzing-debug` Helm Chart has been installed and is running.
   See "[Install senzing-debug Helm Chart](#install-senzing-debug-helm-chart)".

1. Copy the `g2.lic` file to the `senzing-debug` pod
   at `/opt/senzing/g2/data/g2.lic`.

    :pencil2: Identify location of `g2.lic` on local workstation.
    Example:

    ```console
    export G2_LICENSE_PATH=/path/to/local/g2.lic
    ```

    Copy file to debug pod.
    Example:

    ```console
    kubectl cp \
      --namespace ${DEMO_NAMESPACE} \
      ${G2_LICENSE_PATH} \
      ${DEMO_NAMESPACE}/${DEBUG_POD_NAME}:/etc/opt/senzing/g2.lic
    ```

1. Note: `/etc/opt/senzing` is attached as a Kubernetes Persistent Volume Claim (PVC),
   so the license will be seen by all pods that attach to the PVC.

### Get Senzing schema sql for Db2

This step copies the SQL file used to create the Senzing database schema onto the local workstation.

1. Be sure the `senzing-debug` Helm Chart has been installed and is runnning.
   See "[Install senzing-debug Helm Chart](#install-senzing-debug-helm-chart)".

1. Copy the `/opt/senzing/g2/resources/schema/g2core-schema-db2-create.sql`
   file from the `senzing-debug` pod.

    :pencil2: Identify location to place `g2core-schema-db2-create.sql` on local workstation.
    Example:

    ```console
    export SENZING_LOCAL_SQL_PATH=/path/to/local/g2core-schema-db2-create.sql
    ```

    Copy file from pod to local workstation.
    Example:

    ```console
    kubectl cp \
      --namespace ${DEMO_NAMESPACE} \
      ${DEMO_NAMESPACE}/${DEBUG_POD_NAME}:/opt/senzing/g2/resources/schema/g2core-schema-db2-create.sql \
      ${SENZING_LOCAL_SQL_PATH}
    ```

### Create Senzing schema on Db2

1. Copy `g2core-schema-db2-create.sql` to FIXME:
   Example:

   ```console
   FIXME:
   ```

1. Logon to FIXME:

1. If needed, create a database for Senzing data.
   Example:

    ```console
    su - db2inst1
    export DB2_DATABASE=G2

    source sqllib/db2profile
    db2 create database ${DB2_DATABASE} using codeset utf-8 territory us
    ```

1. Connect to `DB2_DATABASE`.
   Example:

    ```console
    su - db2inst1
    export DB2_DATABASE=G2
    export DB2_USER=db2inst1

    source sqllib/db2profile
    db2 connect to ${DB2_DATABASE} user ${DB2_USER}
    ```

    When requested, supply password.

1. Create tables in schema.
   Example:

    ```console
    db2 -tvf g2core-schema-db2-create.sql
    db2 terminate
    ```

1. **Alternative:**  **FIXME:** Using the IBM Cloud Pak for Data console with DB2 Advanced ...
    1. Home > My data > Databases
        1. Open tile for desired database
        1. Click on the ellipse, click on "Open"
        1. Menu > Run SQL
        1. Click on plus sign ("+") to add
        1. Select appropriate Schema (check the box)
        1. From file
        1. In file browser, navigate to SQL file
        1. Click "Run all" button

### Database tuning

**FIXME:** Continue to improve.

1. For information on tuning the database for optimum performance, see
   [Tuning your Database](https://senzing.zendesk.com/hc/en-us/articles/360016288254-Tuning-your-Database).

1. Additional tuning parameters to try:

    ```console
    db2set DB2_USE_ALTERNATE_PAGE_CLEANING=ON
    db2set DB2_APPENDERS_PER_PAGE=1
    db2set DB2_INLIST_TO_NLJN=YES
    db2set DB2_LOGGER_NON_BUFFERED_IO=ON
    db2set DB2_SKIP_LOG_WAIT=YES
    db2set DB2_APM_PERFORMANCE=off
    db2set DB2_SKIPLOCKED_GRAMMAR=YES
    ```

1. Additional tuning parameters to try:

    ```console
    db2 connect to ${DB2_DATABASE} user ${DB2_USER}

    db2 UPDATE SYS_SEQUENCE SET CACHE_SIZE=100000
    db2 commit
    ```

1. **FIXME:** In the user interface, (show how to find database credentials)
    1. Details
    1. Bottom
    1. "Access Information" section

### Optional TLS enablement

:no_entry: **Not ready for prime time**  :no_entry:

If Db2 is not enabled for Transport Layer Security (TLS),
the "Optional TLS enablement" section my be skipped by proceeding to
"[Install RabbitMQ Helm Chart](#install-rabbitmq-helm-chart)."

If using Db2 with TLS, the `db2dsdriver.cfg` file needs to be modified.
Also, "key database" and "stash" files need to be added.

Example:

1. Be sure the `senzing-debug` Helm Chart has been installed.
   See "[Install senzing-debug Helm Chart](#install-senzing-debug-helm-chart)".

1. Generate "key database" and "stash" files.
    1. References:
        1. [Configuring Secure Sockets Layer (SSL) support in non-Java Db2 client](https://www.ibm.com/support/knowledgecenter/en/SSEPGG_11.1.0/com.ibm.db2.luw.admin.sec.doc/doc/t0053518.html)

1. Copy the "key database" file to the `senzing-debug` pod
   at `/opt/senzing/db2/clientstore.kdb`.

    :pencil2: Identify location of `.kbd` on local workstation.  Example:

    ```console
    export DB2_KEY_DATABASE_FILE=/path/to/local/mydbclient.kbd
    ```

    Copy file to debug pod.  Example:

    ```console
    kubectl cp \
      --namespace ${DEMO_NAMESPACE} \
      ${DB2_KEY_DATABASE_FILE} \
      ${DEBUG_POD_NAME}:/opt/senzing/db2/clientstore.kdb
    ```

1. Copy the "stash" file to the `senzing-debug` pod
   at `/opt/senzing/db2/clientstore.sth`.

    :pencil2: Identify location of `.sth` on local workstation.  Example:

    ```console
    export DB2_STASH_FILE=/path/to/local/mydbclient.sth
    ```

    Copy file to debug pod.  Example:

    ```console
    kubectl cp \
      --namespace ${DEMO_NAMESPACE} \
      ${DB2_STASH_FILE} \
      ${DEBUG_POD_NAME}:/opt/senzing/db2/clientstore.sth
    ```

1. Download current `db2dsdriver.cfg` from pod.

    :pencil2: Identify a location on the local workstation of where to put `db2dsdriver.cfg`.  Example:

    ```console
    export MY_DB2DSDRIVER_FILE=/path/to/local/db2dsdriver.cfg
    ```

    Copy file from debug pod.  Example:

    ```console
    kubectl cp \
      --namespace ${DEMO_NAMESPACE} \
      ${DEBUG_POD_NAME}:/opt/senzing/db2/clidriver/cfg/db2dsdriver.cfg \
      ${MY_DB2DSDRIVER_FILE}
    ```

1. Edit the `${MY_DB2DSDRIVER_FILE}` file (i.e. the local copy of `db2dsdriver.cfg`).
   Add the following lines:

    ```xml
    <parameter name="SecurityTransportMode" value="SSL"/>
    <parameter name="SSLClientKeystoredb"   value="/opt/senzing/db2/clientstore.kdb"/>
    <parameter name="SSLClientKeystash"     value="/opt/senzing/db2/clientstore.sth"/>
    ```

    Example:

    ```xml
    <configuration>
      <dsncollection>
        <dsn alias="{SCHEMA}" name="{SCHEMA}" host="{HOST}" port="{PORT}" />
      </dsncollection>
      <databases>
        <database name="{SCHEMA}" host="{HOST}" port="{PORT}">
          <parameter name="CommProtocol" value="TCPIP"/>
          <parameter name="SecurityTransportMode" value="SSL"/>
          <parameter name="SSLClientKeystoredb"   value="/opt/senzing/db2/clientstore.kdb"/>
          <parameter name="SSLClientKeystash"     value="/opt/senzing/db2/clientstore.sth"/>
        </database>
      </databases>
    </configuration>
    ```

1. Upload modified `db2dsdriver.cfg` to pod.

    ```console
    kubectl cp \
      --namespace ${DEMO_NAMESPACE} \
      ${MY_DB2DSDRIVER_FILE} \
      ${DEBUG_POD_NAME}:/opt/senzing/db2/clidriver/cfg/db2dsdriver.cfg
    ```

1. Note: `/opt/senzing` is attached as a Kubernetes Persistent Volume Claim (PVC),
   so the `clientstore.kdb`, `clientstore.sth`, and modified `db2dsdriver.cfg`
   files will be seen by all pods that attach to the PVC.

### Install RabbitMQ Helm chart

1. Install chart.
   Example:

    ```console
    helm install ${HELM_TLS} \
      --name ${DEMO_PREFIX}-rabbitmq \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/rabbitmq.yaml \
      stable/rabbitmq
    ```

1. Wait for pods to run.
   Example:

    ```console
    kubectl get pods \
      --namespace ${DEMO_NAMESPACE} \
      --watch
    ```

1. To view RabbitMQ, see [View RabbitMQ](#view-rabbitmq).

### Install mock-data-generator Helm chart

:warning:  **FIXME:**  This is a **mock** data generator.
In production, this component is replaced by
different components that feed RabbitMQ.

The mock data generator pulls JSON lines from a file and pushes them to RabbitMQ.

1. Install chart.
   Example:

    ```console
    helm install ${HELM_TLS} \
      --name ${DEMO_PREFIX}-senzing-mock-data-generator \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/mock-data-generator-rabbitmq.yaml \
      senzing/senzing-mock-data-generator
    ```

### Install init-container Helm chart

The init-container creates files from templates and initializes the G2 database.

1. Install chart.
   Example:

    ```console
    helm install ${HELM_TLS} \
      --name ${DEMO_PREFIX}-senzing-init-container \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/init-container-db2.yaml \
      senzing/senzing-init-container
    ```

1. Wait for pods to run.
   Example:

    ```console
    kubectl get pods \
      --namespace ${DEMO_NAMESPACE} \
      --watch
    ```

### Install configurator Helm chart

The Senzing Configurator is a micro-service for changing Senzing configuration.

1. Install chart.
   Example:

    ```console
    helm install ${HELM_TLS} \
      --name ${DEMO_PREFIX}-senzing-configurator \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/configurator.yaml \
      senzing/senzing-configurator
    ```

1. To view Senzing Configurator, see [View Senzing Configurator](#view-senzing-configurator).

### Install stream-loader Helm chart

The stream loader pulls messages from RabbitMQ and sends them to Senzing.

1. Install chart.
   Example:

    ```console
    helm install ${HELM_TLS} \
      --name ${DEMO_PREFIX}-senzing-stream-loader \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/stream-loader-rabbitmq-db2.yaml \
      senzing/senzing-stream-loader
    ```

### Install senzing-api-server Helm chart

The Senzing API server receives HTTP requests to read and modify Senzing data.

1. Install chart.
   Example:

    ```console
    helm install ${HELM_TLS} \
      --name ${DEMO_PREFIX}-senzing-api-server \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/senzing-api-server.yaml \
      senzing/senzing-api-server
    ```

1. Wait for pods to run.
   Example:

    ```console
    kubectl get pods \
      --namespace ${DEMO_NAMESPACE} \
      --watch
    ```

1. To view Senzing API server, see [View Senzing API Server](#view-senzing-api-server).

### Install senzing-entity-search-web-app Helm chart

The Senzing Entity Search WebApp is a light-weight WebApp demonstrating Senzing search capabilities.

1. Install chart.
   Example:

    ```console
    helm install ${HELM_TLS} \
      --name ${DEMO_PREFIX}-senzing-entity-search-web-app \
      --namespace ${DEMO_NAMESPACE} \
      --values ${HELM_VALUES_DIR}/entity-search-web-app.yaml \
      senzing/senzing-entity-search-web-app
    ```

1. Wait for pod to run.
   Example:

    ```console
    kubectl get pods \
      --namespace ${DEMO_NAMESPACE} \
      --watch
    ```

1. To view Senzing Entity Search WebApp, see [View Senzing Entity Search WebApp](#view-senzing-entity-search-webapp).

### View data

1. Username and password for the following sites are the values seen in the corresponding "values" YAML file located in
   [helm-values-templates](../helm-values-templates).
1. :pencil2: When using a separate terminal window in each of the examples below, set environment variables.
   Example:

    ```console
    export DEMO_PREFIX=my
    export DEMO_NAMESPACE=${DEMO_PREFIX}-namespace
    ```

#### View Senzing Debug pod

1. In a separate terminal window, log into debug pod.
   Example:

    ```console
    export DEBUG_POD_NAME=$(kubectl get pods \
      --namespace ${DEMO_NAMESPACE} \
      --output jsonpath="{.items[0].metadata.name}" \
      --selector "app.kubernetes.io/name=senzing-debug, \
                  app.kubernetes.io/instance=${DEMO_PREFIX}-senzing-debug" \
      )

    kubectl exec -it --namespace ${DEMO_NAMESPACE} ${DEBUG_POD_NAME} -- /bin/bash
    ```

#### View RabbitMQ

1. In a separate terminal window, port forward to local machine.
   Example:

    ```console
    kubectl port-forward \
      --address 0.0.0.0 \
      --namespace ${DEMO_NAMESPACE} \
      svc/${DEMO_PREFIX}-rabbitmq 15672:15672
    ```

1. RabbitMQ will be viewable at [localhost:15672](http://localhost:15672).
    1. Login
        1. See `helm-values/rabbitmq.yaml` for Username and password.

#### View Senzing Configurator

1. In a separate terminal window, port forward to local machine.
   Example:

    ```console
    kubectl port-forward \
      --address 0.0.0.0 \
      --namespace ${DEMO_NAMESPACE} \
      svc/${DEMO_PREFIX}-senzing-configurator 5001:5000
    ```

1. Make HTTP calls via `curl`.
   Example:

    ```console
    export SENZING_API_SERVICE=http://localhost:5001

    curl -X GET ${SENZING_API_SERVICE}/datasources
    ```

#### View Senzing API Server

1. In a separate terminal window, port forward to local machine.
   Example:

    ```console
    kubectl port-forward \
      --address 0.0.0.0 \
      --namespace ${DEMO_NAMESPACE} \
      svc/${DEMO_PREFIX}-senzing-api-server 8889:8080
    ```

1. Make HTTP calls via `curl`.
   Example:

    ```console
    export SENZING_API_SERVICE=http://localhost:8889

    curl -X GET ${SENZING_API_SERVICE}/heartbeat
    curl -X GET ${SENZING_API_SERVICE}/license
    curl -X GET ${SENZING_API_SERVICE}/entities/1
    ```

#### View Senzing Entity Search WebApp

1. In a separate terminal window, port forward to local machine.
   Example:

    ```console
    kubectl port-forward \
      --address 0.0.0.0 \
      --namespace ${DEMO_NAMESPACE} \
      svc/${DEMO_PREFIX}-senzing-entity-search-web-app 8888:80
    ```

1. Senzing Entity Search WebApp will be viewable at [localhost:8888](http://localhost:8888).
   The [demonstration](https://github.com/Senzing/knowledge-base/blob/master/demonstrations/docker-compose-web-app.md)
   instructions will give a tour of the Senzing web app.

## Cleanup

### Delete Helm charts

1. Example:

    ```console
    helm delete ${HELM_TLS} --purge ${DEMO_PREFIX}-senzing-entity-search-web-app
    helm delete ${HELM_TLS} --purge ${DEMO_PREFIX}-senzing-api-server
    helm delete ${HELM_TLS} --purge ${DEMO_PREFIX}-senzing-stream-loader
    helm delete ${HELM_TLS} --purge ${DEMO_PREFIX}-senzing-configurator
    helm delete ${HELM_TLS} --purge ${DEMO_PREFIX}-senzing-init-container
    helm delete ${HELM_TLS} --purge ${DEMO_PREFIX}-senzing-mock-data-generator
    helm delete ${HELM_TLS} --purge ${DEMO_PREFIX}-senzing-rabbitmq
    helm delete ${HELM_TLS} --purge ${DEMO_PREFIX}-senzing-ibm-db2
    helm delete ${HELM_TLS} --purge ${DEMO_PREFIX}-senzing-debug
    helm delete ${HELM_TLS} --purge ${DEMO_PREFIX}-ibm-db2-driver-installer
    helm delete ${HELM_TLS} --purge ${DEMO_PREFIX}-senzing-yum
    ```

### Delete database tables

1. **FIXME:** Example:

### Delete git repository

1. Delete git repository.  Example:

    ```console
    sudo rm -rf ${GIT_REPOSITORY_DIR}
    ```
