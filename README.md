# Webinar_dataflow

### we will build next pipeline:
#### Step 1. Create k8s cluster and spin up Kafka, MySQL, KafkaConnect cluster.
#### Step 2. Create PostgreSQL as DBaaS.
#### Step 3. Create JupyterHub as a Service inside MCS. Test some code using Jupyter.
#### Step 4. Create NiFi cluster. Write some ETL.
#### Step 5. Create Big Data cluster, start Superset, built several fancy charts.

## Prerequisites


### Create K8s cluster in mcs and download kubeconfig
Instruction: https://mcs.mail.ru/help/kubernetes/clusterfast

Kubernetes as a Service: https://mcs.mail.ru/app/services/containers/add/


### Install Docker if you want to build your own images (optional)
#### You can skip this step and use our own images
https://docs.docker.com/engine/install/ubuntu/

### Install kubectl
https://mcs.mail.ru/help/kubernetes/local-clients#section-6

### Set path to kubeсonfig for kubectl
```console
export KUBECONFIG=/replace_with_path/to_your_kubeconfig.yaml
```

### also it will be easier to work with kubectl while enabling autocomplete and using alias
```console
alias k=kubectl
source <(kubectl completion bash)
complete -F __start_kubectl k
```


## Kafka and Debezium installation

### create namespace
```console
kubectl create namespace kafka
```

### Then install the cluster operator and associated resources:

```console
curl -L https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.16.1/strimzi-cluster-operator-0.16.1.yaml \
  | sed 's/namespace: .*/namespace: kafka/' \
  | kubectl apply -f - -n kafka
```

### Spin up a Kafka cluster  
```console
kubectl -n kafka \
    apply -f https://raw.githubusercontent.com/strimzi/strimzi-kafka-operator/0.16.1/examples/kafka/kafka-persistent-single.yaml \
  && kubectl wait kafka/my-cluster --for=condition=Ready --timeout=300s -n kafka  
```

### Create a Strimzi Kafka Connect image which includes the Debezium MySQL connector and its dependencies
### You can use our image or built your own from the ground

#### Using our image
```console 
export DOCKER_ORG=mcscloud
``` 

#### Building your own image (optional)
##### First download and extract the Debezium MySQL connector archive

```console
curl https://repo1.maven.org/maven2/io/debezium/debezium-connector-mysql/1.0.0.Final/debezium-connector-mysql-1.0.0.Final-plugin.tar.gz \
| tar xvz
```

##### Then Prepare a Dockerfile which adds those connector files to the Strimzi Kafka Connect image (optional)
```console
cat <<EOF >Dockerfile
FROM strimzi/kafka:0.16.1-kafka-2.4.0
USER root:root
RUN mkdir -p /opt/kafka/plugins/debezium
COPY ./debezium-connector-mysql/ /opt/kafka/plugins/debezium/
USER 1001
EOF
```

##### You should use your own dockerhub organization (optional)
```console
export DOCKER_ORG=REPLACE_THIS_WITH_YOUR_DOCKERHUB_OR_USE_OUR_IMAGE
docker build . -t ${DOCKER_ORG}/connect-debezium
docker push ${DOCKER_ORG}/connect-debezium
```

## Starting up MySQL in K8s

### Create MySQL
```console
cat | kubectl -n kafka apply -f - << 'EOF'
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: debezium/example-mysql:1.0
        name: mysql
        env:
          # Use secret in real usage
        - name: MYSQL_ROOT_PASSWORD
          value: debezium
        - name: MYSQL_USER
          value: mysqluser
        - name: MYSQL_PASSWORD
          value: mysqlpw
        ports:
        - containerPort: 3306
          name: mysql
EOF
```


### create Services for MySQL
```console
cat | kubectl -n kafka apply -f - << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
EOF
```

```console
cat | kubectl -n kafka apply -f - << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: mysql-externallb
spec:
  ports:
  - port: 3306
    targetPort: 3306
  selector:
    app: mysql
  type: LoadBalancer
EOF
```

### run mysql client to check our db instance, create new table, insert test records
```console
kubectl run -n kafka -it --rm --image=mysql:5.7 --restart=Never mysql-client -- mysql -h mysql -pdebezium

use inventory;

CREATE TABLE sales (client_id INT,
region VARCHAR (100),
product_name VARCHAR(100),
quantity INT,
price  INT
);

INSERT INTO inventory.sales(client_id, region, product_name, quantity, price)
VALUES
(1,'Moscow', 'Milk', 3, 85),
(2,'Piter', 'Umbrella', 3, 385),
(3,'Kazan', 'Ichpochmak', 3, 123),
(4,'Saratov', 'Bread', 3, 8),
(5,'Voronezh', 'Juice', 3, 42)
;
```

### Secure the database credentials

```console
cat <<EOF > debezium-mysql-credentials.properties
mysql_username: debezium
mysql_password: dbz
EOF


kubectl -n kafka create secret generic my-sql-credentials \
  --from-file=debezium-mysql-credentials.properties
  
rm debezium-mysql-credentials.properties
```

## Create KafkaConnect Cluster
```console
cat <<EOF | kubectl -n kafka apply -f -
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaConnect
metadata:
  name: my-connect-cluster
  annotations:
  # use-connector-resources configures this KafkaConnect
  # to use KafkaConnector resources to avoid
  # needing to call the Connect REST API directly
    strimzi.io/use-connector-resources: "true"
spec:
  image: ${DOCKER_ORG}/connect-debezium
  replicas: 1
  bootstrapServers: my-cluster-kafka-bootstrap:9093
  tls:
    trustedCertificates:
      - secretName: my-cluster-cluster-ca-cert
        certificate: ca.crt
  config:
    config.storage.replication.factor: 1
    offset.storage.replication.factor: 1
    status.storage.replication.factor: 1
    config.providers: file
    config.providers.file.class: org.apache.kafka.common.config.provider.FileConfigProvider
  externalConfiguration:
    volumes:
      - name: connector-config
        secret:
          secretName: my-sql-credentials

EOF
```


## Create the KafkaConnector to connect to our “inventory” database in MySQL

```console
cat | kubectl -n kafka apply -f - << 'EOF'
apiVersion: "kafka.strimzi.io/v1alpha1"
kind: "KafkaConnector"
metadata:
  name: "inventory-connector"
  labels:
    strimzi.io/cluster: my-connect-cluster
spec:
  class: io.debezium.connector.mysql.MySqlConnector
  tasksMax: 1
  config:
    database.hostname: mysql
    database.port: "3306"
    database.user: "${file:/opt/kafka/external-configuration/connector-config/debezium-mysql-credentials.properties:mysql_username}"
    database.password: "${file:/opt/kafka/external-configuration/connector-config/debezium-mysql-credentials.properties:mysql_password}"
    database.server.id: "184054"
    database.server.name: "dbserver1"
    database.whitelist: "inventory"
    database.history.kafka.bootstrap.servers: "my-cluster-kafka-bootstrap:9092"
    database.history.kafka.topic: "schema-changes.inventory"
    include.schema.changes: "true" 
EOF
```

### Enabling external access to Kafka Cluster
#### you can use nano as editor or any favorite tool
##### example with nano

```console
export KUBE_EDITOR="nano"
kubectl edit kafka -n kafka

Then find section with listeners and add next config

    listeners:
      external:
        type: loadbalancer
        tls: false
```

#### Load balancer with external ip will be created after several minutes
#### Then you should list LB and write down external-ip values for accessing Kafka and MySQL
#### Services: my-cluster-kafka-external-bootstrap and mysql-externallb

```console
kubectl get service -n kafka
or
kubectl get service mysql-externallb -n kafka
kubectl get service my-cluster-kafka-external-bootstrap -n kafka
```

#### Test Kafka Cluster
```console
kubectl -n kafka exec my-cluster-kafka-0 -c kafka -i -t -- bin/kafka-topics.sh --bootstrap-server ENTER_IP_OF_KAFKA_EXTERNAL_BOOTSTRAP:9094 --list

kubectl -n kafka exec my-cluster-kafka-0 -c kafka -i -t -- bin/kafka-console-consumer.sh --bootstrap-server ENTER_IP_OF_KAFKA_EXTERNAL_BOOTSTRAP:9094 --topic dbserver1.inventory.sales  --from-beginning
```

## Next Steps

### Step 2
#### Create PostgreSQL as DBaaS.
##### https://mcs.mail.ru/app/services/databases/add/
##### https://mcs.mail.ru/help/instance-databases/launch-databases

### Step 3
#### Create Machine learning workplace
#### https://mcs.mail.ru/app/services/infra/servers/add/?service=learning
#### https://mcs.mail.ru/help/machinelearning/start-machine-learning?kb_language=ru_RU
##### get ipynb template from repository


### Step 4. Create NiFi (DataFlow) cluster. Write some ETL.
#### https://mcs.mail.ru/app/services/bigdata/add/?templateName
#### https://mcs.mail.ru/help/start-bigdata/launch-cluster-bigdata?kb_language=ru_RU
#### https://mcs.mail.ru/help/howto/port
##### NiFi port 9090
##### get NiFi template from repository
##### you will need to upload postgres drivers to NiFi cluster head node
##### example: scp -i your_ssh_key.pem C:\mcs\postgresql-42.2.17.jar admin@public_ip_of_NiFi_cluster:/nifi_drivers/

#### Step 5. Create Big Data cluster, start Superset, built several fancy charts.
#### https://mcs.mail.ru/app/services/bigdata/add/?templateName
##### Superset port 9088
##### connect Superset to Postgres: postgresql+pygresql://username:passwordk@public_ip_db:5432/db_name

