# Demo preparation

These are the pre-reqs I used to demonstrate my sample App Connect Enterprise application. My App Connect application connects to a PostgreSQL database - so I need to set up a PostgreSQL database to demo it. My App Connect application receives messages from Kafka - so I need to create a Kafka cluster to demo it. And so on.

I'm keeping this here as it'll be convenient when I need to recreate this demo from scratch, but as you'll be building and deploying your own App Connect Enterprise application, **you will have different pre-reqs to me**.

If you just follow these instructions on your existing OpenShift cluster, you will likely find some of this clashes with what you already have set up on your cluster.

This sample is based on Cloud Pak for Integration 2023.2.1. The versions of each components used are:

- IBM Integration Platform Navigator operator channel v7.1, defined in the file [ibm-integration.yaml](./operators/ibm-integration.yaml)
- IBM App Connect operator channel v9.1, defined in the file [ibm-appconnect.yaml](./operators/ibm-appconnect.yaml)
- IBM Event Streams operator channel v3.2, defined in the file [ibm-eventstreams.yaml](./operators/ibm-eventstreams.yaml)
- Plaform Navigator instance v2023.2.1, defined in the [platform-navigator.yaml](./cp4i/platform-navigator.yaml)
- App Connect Dashboard instance v12.0.9.0-r1, defined in the file [dashboard.yaml](./appconnect/dashboard.yaml)
- App Connect Integration Server instance v12.0.9.0-r1, defined the files [simple-pipelinerun.yaml](../simple-pipelinerun.yaml) and [complex-pipelinerun.yaml](../complex-pipelinerun.yaml). You need to update the parameters `app-connect-enterprise-version`, `app-connect-enterprise-license` and `app-connect-enterprise-base-image`. Refer to the documentation on the values needed [Licensing reference for App Connect](https://www.ibm.com/docs/en/app-connect/containers_cd?topic=resources-licensing-reference-app-connect-operator) and [App Connect server image](https://www.ibm.com/docs/en/app-connect/containers_cd?topic=obtaining-app-connect-enterprise-server-image-from-cloud-container-registry)

## A. Base installation for Simple pipeline run

### 1. Export IBM entitlement key
```sh
% export IBM_ENTITLEMENT_KEY=<ibm-entitlment-key-you-copied>
```

### 2. Setup storage classes
```sh
% oc get sc
NAME                          PROVISIONER                             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
localblock                    kubernetes.io/no-provisioner            Delete          WaitForFirstConsumer   false                  109m
ocs-storagecluster-ceph-rbd   openshift-storage.rbd.csi.ceph.com      Delete          Immediate              true                   103m
ocs-storagecluster-ceph-rgw   openshift-storage.ceph.rook.io/bucket   Delete          Immediate              false                  106m
ocs-storagecluster-cephfs     openshift-storage.cephfs.csi.ceph.com   Delete          Immediate              true                   103m
openshift-storage.noobaa.io   openshift-storage.noobaa.io/obc         Delete          Immediate              false                  101m

% export FILE_STORAGECLASS=ocs-storagecluster-cephfs
% export BLOCK_STORAGECLASS=ocs-storagecluster-ceph-rbd
```

### 2. Add IBM software to Operator Hub
```sh
% oc apply -f ibm-catalog-source.yaml
catalogsource.operators.coreos.com/ibm-operator-catalog created
```

### 3. Install operators needed for the demo
```sh
% oc apply -f operators
subscription.operators.coreos.com/ibm-appconnect created
subscription.operators.coreos.com/ibm-eventstreams created
subscription.operators.coreos.com/ibm-integration-platform-navigator created
subscription.operators.coreos.com/postgresql created
subscription.operators.coreos.com/openshift-pipelines-operator-rh created
```

### 4. Deploy Platform UI

#### a. Create `integration` project and setup ibm entitlement key
```sh
% oc new-project integration
Now using project "integration" on server "https://api.64e01bd0fa254600179b97b4.cloud.techzone.ibm.com:6443".

% oc create secret docker-registry ibm-entitlement-key --docker-username=cp --docker-password=${IBM_ENTITLEMENT_KEY} --docker-server=cp.icr.io -n integration
secret/ibm-entitlement-key created
```

#### b. Replace the ${FILE_STORAGECLASS} variable and create `./cp4i/platform-navigator.yaml`
```sh
% envsubst < cp4i/platform-navigator.yaml.tmpl > cp4i/platform-navigator.yaml
```

#### c. Install Platform UI
```sh
% oc apply -f ./cp4i
platformnavigator.integration.ibm.com/navigator created
```

#### d. Wait for Platform UI to be ready. This takes 30-45 mins because it also installs Cloud Pak for Integration (foundational services).
```sh
% oc get platformnavigator navigator -n integration
NAME        REPLICAS   VERSION   STATUS    READY   LASTUPDATE   AGE     MESSAGE
navigator   1                    Pending           2m16s        4m15s   Waiting for Zen to install. This can take up to 30 minutes, please wait. The ZenService object in the [integration] namespace is being fulfilled. The ZenService status is [Running reconciliation].
```

#### e. When installation is completed, you should see the following:
```sh
% oc get platformnavigator navigator -n integration 
NAME        REPLICAS   VERSION      STATUS   READY   LASTUPDATE   AGE   MESSAGE
navigator   1          2023.2.1-1   Ready    True    51m          85m   Platform UI has been provisioned.
```

#### f. Access Platform UI

##### i. Get URL
```sh
% oc get route -n integration navigator-pn -ojson | jq -r .spec.host
navigator-pn-integration.apps.<clusterID>.<domainName>
```

##### ii. Get credentials
```sh
% oc get secret -n ibm-common-services platform-auth-idp-credentials -ojson | jq -r .data.admin_password | base64 -d
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

##### iii. Go to private window of a browser (incognito mode) and login to Platform UI
![Platform UI](./images/platform-ui.png)

### 7. Deploy ACE Dashboard UI

#### a. Create `ace-demo` project and setup ibm entitlement key
```sh
% oc new-project ace-demo
Now using project "ace-demo" on server "https://api.64e01bd0fa254600179b97b4.cloud.techzone.ibm.com:6443".

% oc create secret docker-registry ibm-entitlement-key --docker-username=cp --docker-password=${IBM_ENTITLEMENT_KEY} --docker-server=cp.icr.io -n ace-demo
secret/ibm-entitlement-key created
```

#### b. Replace the ${FILE_STORAGECLASS} variable and create `./appconnect/dashboard.yaml`
```sh
% envsubst < appconnect/dashboard.yaml.tmpl > appconnect/dashboard.yaml
```

#### c. Install ACE Dashboard UI
```sh
% oc apply -f ./appconnect
dashboard.appconnect.ibm.com/ace-dashboard created
```

#### d. Wait for ACE Dashboard UI to be ready. This takes 5-10 mins.
```sh
% oc get dashboard ace-dashboard -n ace-demo
NAME            RESOLVEDVERSION   REPLICAS   CUSTOMIMAGES   STATUS    URL   AGE
ace-dashboard   12.0.9.0-r1       1          false          Pending         67s
```

#### e. When installation is completed, you should see the following
```sh
% oc get dashboard ace-dashboard -n ace-demo 
NAME            RESOLVEDVERSION   REPLICAS   CUSTOMIMAGES   STATUS   URL                                                                                                                                 AGE
ace-dashboard   12.0.9.0-r1       1          false          Ready    https://cpd-integration.apps.64e2d402c503c400176b4ae1.cloud.techzone.ibm.com/integration/run/integrations/ace-demo/ace-dashboard/   5m47s
```

#### f. Access ACE Dashboard UI

##### i. Get URL
```sh
% oc get route -n ace-demo ace-dashboard-ui -ojson | jq -r .spec.host
ace-dashboard-ui-ace-demo.apps.<clusterID>.<domainName>
```

##### ii. Go to private window of a browser (incognito mode) and login to ACE Dashboard UI

![](./images/ace-dashboard-ui.png)

### 8. Setup the namespace where the sample ACE demo will run
```sh
% oc new-project ace-demo
Now using project "ace-demo" on server "https://api.64e01bd0fa254600179b97b4.cloud.techzone.ibm.com:6443".

% oc create secret docker-registry ibm-entitlement-key --docker-username=cp --docker-password=${IBM_ENTITLEMENT_KEY} --docker-server=cp.icr.io -n ace-demo
secret/ibm-entitlement-key created
```

## B. Additional installation for Complex pipeline run

### 1. Deploy Event Streams

#### a. Create `eventstreams` project and setup ibm entitlement key
```sh
% oc new-project eventstreams
Now using project "eventstreams" on server "https://api.64e01bd0fa254600179b97b4.cloud.techzone.ibm.com:6443".

% oc create secret docker-registry ibm-entitlement-key --docker-username=cp --docker-password=${IBM_ENTITLEMENT_KEY} --docker-server=cp.icr.io -n eventstreams
secret/ibm-entitlement-key created
```

#### b. Install Event Streams
```sh
% oc apply -f ./kafka
kafkauser.eventstreams.ibm.com/appconnect-kafka-user created
eventstreams.eventstreams.ibm.com/event-backbone created
secret/kafka-connection-info created
kafkatopic.eventstreams.ibm.com/todo.updates created
```

#### b. Check installation status for EventStreams
```sh
% oc get eventstreams event-backbone -n eventstreams
NAME             STATUS
event-backbone   Ready
```

#### c. Access Event Stream UI

##### i. Get URL
```sh
% oc get route -n eventstreams event-backbone-ibm-es-ui -ojson | jq -r .spec.host
event-backbone-ibm-es-ui-eventstreams.apps.<clusterID>.<domainName>
```

##### ii. Go to private window of a browser (incognito mode) and login to Event Stream UI

![](./images/eventstreams-ui.png)

### 2. Deploy PostgreSQL (not needed for Simple pipeline, only for Complex pipeline)

#### a. Create `eventstreams` project and setup ibm entitlement key
```sh
% oc new-project postgresql
Now using project "postgresql" on server "https://api.64e01bd0fa254600179b97b4.cloud.techzone.ibm.com:6443".
```
#### b. Set RWO block storageclass as default storageclass

```sh
oc patch storageclass ${BLOCK_STORAGECLASS} -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

#### c. Install PostgresSQL
```sh
% oc apply -f ./postgresql/db-data.yaml
configmap/pg-initial-data-cm created

% oc apply -f ./postgresql/database.yaml
postgrescluster.postgres-operator.crunchydata.com/store created
```

#### d. Check installation status for PostgresSQL, make sure all the pods are Running (or Completed)

```sh
% oc get po -n postgresql
NAME                               READY   STATUS      RESTARTS   AGE
store-00-pknl-0                    4/4     Running     0          6m28s
store-backup-t9bc-d2w25            0/1     Completed   0          5m40s
store-pgbouncer-7487d969cf-6mqgd   2/2     Running     0          6m27s
store-repo-host-0                  2/2     Running     0          6m28s
```
#### e. Access the database

```sh
% oc exec -it -n postgresql -c database \
  $(oc get pods -n postgresql --selector='postgres-operator.crunchydata.com/cluster=store,postgres-operator.crunchydata.com/role=master' -o name) \
  -- psql -d store
psql (15.3)
Type "help" for help.

store=# select * from todos;
 id | user_id | title | encoded_title | is_completed 
----+---------+-------+---------------+--------------
(0 rows)
```

### 3. Submit an HTTP request to the simple ACE flow

```sh
curl "http://$(oc get route -nace-demo hello-world-http -o jsonpath='{.spec.host}')/hello"
```

### 4. Produce a message to the Kafka topic that will trigger the complex ACE flow

```sh
BOOTSTRAP=$(oc get eventstreams event-backbone -neventstreams -ojsonpath='{.status.kafkaListeners[1].bootstrapServers}')
PASSWORD=$(oc get secret -neventstreams appconnect-kafka-user -ojsonpath='{.data.password}' | base64 -d)
oc get secret -neventstreams event-backbone-cluster-ca-cert -ojsonpath='{.data.ca\.p12}' | base64 -d > ca.p12
CA_PASSWORD=$(oc get secret -neventstreams event-backbone-cluster-ca-cert -ojsonpath='{.data.ca\.password}' | base64 -d)

echo '{"id": 1, "message": "quick test"}' | ./kafka-console-producer.sh \
    --bootstrap-server $BOOTSTRAP \
    --topic TODO.UPDATES \
    --producer-property "security.protocol=SASL_SSL" \
    --producer-property "sasl.mechanism=SCRAM-SHA-512" \
    --producer-property "sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="appconnect-kafka-user" password="$PASSWORD";" \
    --producer-property "ssl.truststore.location=ca.p12" \
    --producer-property "ssl.truststore.type=PKCS12" \
    --producer-property "ssl.truststore.password=$CA_PASSWORD"
```

### 5. Check that the ACE flow put something in PostgreSQL

```sh
oc exec -it -n postgresql -c database \
  $(oc get pods -n postgresql --selector='postgres-operator.crunchydata.com/cluster=store,postgres-operator.crunchydata.com/role=master' -o name) \
  -- psql -d store
```

```sql
store=# select * from todos;
 id | user_id |       title        |            encoded_title             | is_completed
----+---------+--------------------+--------------------------------------+--------------
  1 |       1 | delectus aut autem | RU5DT0RFRDogZGVsZWN0dXMgYXV0IGF1dGVt | f
(1 row)
```