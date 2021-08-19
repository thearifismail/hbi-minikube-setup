# Host Inventory Deployment to Minikube
This project shows how to deploy "host inventory" to a kubernetes namespace.   Target users are expected to be familiar with Kubernetes concepts but have limited experience with deploying services to Kubernetes.

Theoretically it sounds very simple to deploy a service to Kubernetes but gets complicated very quickly when the service integrates with other services.  Almost all the times, the integrating service "Getting Started" does fit the scenario in hand.  

It is recommended to use "Minikube" which is much faster than "Code Ready Containers".  Minikube may not be able to handle OpenShift specific features, like Templates, parameters, DeploymentConfigs etc.

The most painful process was getting started.  Almost all the times, the Google help provided conceptual clarification but the concepts could not be verified because the available cluster did not allow to verify Google steps because Google docs assumed "admin" privileges on the cluster.

This project uses "minikube" and sets up Kafka using "Strimzi Operator".

# Setup Minikube
For this project, minikube is set to use the following default parameters.
```
cat ~/.minikube/config/config.json 
{
    "cpus": 4,
    "disk-size": "36GB",
    "driver": "hyperkit", # for Linux use 'kvm2'
    "memory": "8192"
}
```
### Start minikube cluster
```
minikube start
```

# Setup Kafka
Kafka is setup using Strimzi "Getting Started" instrucations available at https://strimzi.io/quickstarts/
```
kubectl create namespace kafka
kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
kubectl apply -f https://strimzi.io/examples/latest/kafka/kafka-persistent-single.yaml -n kafka 
kubectl wait kafka/my-cluster --for=condition=Ready --timeout=300s -n kafka
```
## Verify Kafka
Open two terminal windows.  In the first window, type the following to produce messages:
```
kubectl -n kafka run kafka-producer -ti --image=quay.io/strimzi/kafka:0.24.0-kafka-2.8.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --broker-list my-cluster-kafka-bootstrap:9092 --topic my-topic
```
In the second terminal, type the following to cosume incoming messages.
```
kubectl -n kafka run kafka-consumer -ti --image=quay.io/strimzi/kafka:0.24.0-kafka-2.8.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic --from-beginning
```
Typing anything in the producer terminal.  It should appear in the consumer. Appearing messages in the consumer terminal verifies that Kafka is working as expected.

## Create Host Inventory Namespace and Resources.
The created resources include namespace, secrets, deploymengs, services, cronJobs/jobs.

### Create hbi namespace
```
kubectl create namespace hbi
```

### Create imagePullSecrets
```
kubectl -n hbi create -f hbi-secret-quay.yaml
kubectl -n hbi create -f hbi-secret-rhreg.yaml
kubectl -n hbi create -f hbi-secret-dippy-bot.yaml
```

```
kubectl -n hbi patch serviceaccount default -p '{"imagePullSecrets": [{"name": "quay-secret"}]}'
kubectl -n hbi patch serviceaccount default -p '{"imagePullSecrets": [{"name": "rhreg-secret"}]}'
```

### Create pvc needed for hbi database
```
kubectl -n hbi create -f hbi-pvc.yaml
```

### Deploy postgresql database
```
kubectl -n hbi create -f hbi-db.yaml
```

### Deploy mq-services; "hbi-mq-p1" initializes/migrates the DB
```
kubectl -n hbi create -f hbi-mq-p1.yaml
kubectl -n hbi create -f hbi-mq-pmin.yaml
kubectl -n hbi create -f hbi-mq-sp.yaml
```

### Deploy insights-inventory-service
```
kubectl -n hbi create -f hbi-api-server.yaml
```

###  Expose services for access from outside of the cluster
```
kubectl port-forward svc/inventory-db 5432:5432 -n hbi >/dev/null 2>&1 &
kubectl -n kafka port-forward svc/my-cluster-kafka-bootstrap 9092:9092 >/dev/null 2>&1 &
kubectl port-forward svc/insights-inventory 8080:8080 -n hbi >/dev/null 2>&1 &
```

### Deploy system_profile_validator
```
kubectl -n hbi create -f hbi-mq-sp.yaml
```

### Deploy manual job "host-synchronizer"
```
kubectl -n hbi create -f hbi-job-synchronizer.yaml
```

### Deploy cronjob for system profile validation
```
kubectl -n hbi create -f hbi-cj-sp-validator.yaml
```
