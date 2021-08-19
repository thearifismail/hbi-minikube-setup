# Host Inventory Deployment to Minikube
This project shows how to deploy "host inventory" to a kubernetes namespace.  Target audience are new team members or members with limited experience with Kubernetes and OpenShift.  Using this project as a guide, users can deploy their latest code to Kubernetes/Minikube before delivering the changes to code repo.  Experienced users may find it too basic and rudimentary.  The reality is that in real world there are so many pieces and integrating services in a kubernetes cluster that it is hard to quickly recall the required howtos to be efficient. e.g. a user may not know how to deploy kafka to write messages for creating hosts.  Users are expected to know kubernetes concepts.

It provides samples written from scratch for creating scratch deployments, services, and cron jobs from scratch.  Though the yamls are written for kubernetes and deployed to minikube, they are expected to work without any problems in OpenShift or Code Ready Containers.

# Setup Minikube
For this project, Install [Minikube](https://minikube.sigs.k8s.io/docs/start/) and set it to use the following default parameters.
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
Kafka is setup using [Strimzi's Getting Started](https://strimzi.io/quickstarts/).  Host inventory depends on it for posting messages for creating hosts and various other tasks.
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
Type anything in the producer terminal.  It should appear in the consumer terminal. Appearing messages in the consumer terminal verifies that Kafka is working as expected.

# Setup Host Inventory Resources

## Setup Secrets.
Create secrets for pulling images from image registries and checking out code from github
```
kubectl create secret docker-registry quay-secret --docker-server=quay.io --docker-username=<username> --docker-password=<password> --docker-email=<email_address>

kubectl create secret docker-registry quay-secret --docker-server=registry.redhat.io --docker-username=<username> --docker-password=<password> --docker-email=<email_address>

kubectl create secret generic github-bot --from-literal=user=<github_user> --from-literal=token=<github_token>
```

Attach secrets to service accounts, which provide identity for processes that run in a pod.
```
kubectl -n hbi patch serviceaccount default -p '{"imagePullSecrets": [{"name": "quay-secret"}]}'
kubectl -n hbi patch serviceaccount default -p '{"imagePullSecrets": [{"name": "rhreg-secret"}]}'
```

## Create Resources
The created resources include namespace, secrets, deploymengs, services, cronJobs/jobs.

### Create hbi namespace
```
kubectl create namespace hbi
```

### Create pvc needed for hbi database
```
kubectl -n hbi create -f hbi-pvc.yaml
```

### Deploy postgresql database
```
kubectl -n hbi create -f hbi-db.yaml
```

### Deploy mq-services. "hbi-mq-p1" initializes/migrates the database
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
1. 5432: access database.
1. 9092: needed for creating hosts and posting messages to Kafka.
1. 8080: provides access to the API server. e.g. getting hosts

## Create CronJobs
Deploy manual job "host-synchronizer"
```
kubectl -n hbi create -f hbi-job-synchronizer.yaml
```

Deploy cronjob for system profile validation
```
kubectl -n hbi create -f hbi-cj-sp-validator.yaml
```

# TODO
Deploy xjoin-search, RBAC, and may be prometheus.
