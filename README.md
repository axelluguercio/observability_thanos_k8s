# observability thanos

### Create monitoring and prod namespaces

`kubectl --context=kind-cluster1 create namespace prod`
`kubectl --context=kind-cluster1 create namespace monitoring`
`kubectl --context=kind-cluster2 create namespace prod`
`kubectl --context=kind-cluster2 create namespace monitoring`

### Deploy the lamp_app 

```
kubectl --context=kind-cluster1 apply -f lamp_app/mysql/ -n prod
kubectl --context=kind-cluster1 apply -f lamp_app/php/ -n prod
kubectl --context=kind-cluster1 apply -f lamp_app/phpmyadmin/ -n prod
 
kubectl --context=cluster2 apply -f lamp_app/mysql/ -n prod
kubectl --context=cluster2 apply -f lamp_app/php/ -n prod
kubectl --context=cluster2 apply -f lamp_app/phpmyadmin/ -n prod
```

### Expose the application with an ingress controller

`kubectl --context=kind-cluster2 apply -f lamp_app/ingress_controller.yaml -n prod`
`kubectl --context=kind-cluster2 apply -f lamp_app/ingress_controller.yaml -n prod`


### Setup prometheus

Install prometheus with helm chart

###### add repos

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo update
```

###### install chart

`helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring`

###### Expose Prometheus and Grafana

`kubectl --context=kind-cluster1 apply -f monitoring/ingress-monitoring-controller.yml -n monitoring`
`kubectl --context=kind-cluster2 apply -f monitoring/ingress-monitoring-controller.yml -n monitoring`

### Setup Service monitoring to monitor mysql database deployment

A Prometheus exporter for  metrics.

This chart bootstraps a Prometheus  deployment on a  cluster using the  package manager.

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

###### configure values

Scrap the values for the mysqld-exporter

`helm show values prometheus-community/prometheus-mysql-exporter > values.yml`

Modify matchlabels to be sure that service discovery can reach the service monitor.

```
# Default values for prometheus-mysql-exporter.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.
serviceMonitor:
  enabled: true
  additionalLabels:
    release: prometheus 

# mysql connection params which build the DATA_SOURCE_NAME env var of the docker container
mysql:
  # secret only containing the password
  existingPasswordSecret:
    name: "[NAME_SECRET_DB_PASS]"
    key: "mysql-password"
```

Install exporter
`helm install mysqld-exporter prometheus-community/prometheus-mysql-exporter -f values.yml --namespace prod`

###### Link to chart
[https://github.com/prometheus/mysqld_exporter]

Now prometheus will start to scrap metrics from mysql service.

### Generate secret for GCP bucket to store metrics 

Now we can deploy a better storage solution for store all metrics to a long term storage such as GCP buckets using thanos.

Configure thanos-objstore to match with an existing GCP bucket.

`vim monitoring/thanos-objstore.yaml`

```
type: GCS
config:
  bucket: "observability-for-kubernetes-thanos"
  service_account: |-
  {
    "type": "service_account",
    "project_id": "[YOUR_PROJECT_ID]",
    "private_key_id": "",
    "private_key": "-----BEGIN PRIVATE KEY-----\n\n-----END PRIVATE KEY-----\n",
    "client_email": "observability-for-kubernetes@[YOUR_PROJECT_ID].iam.gserviceaccount.com",
    "client_id": "",
    "auth_uri": "https://accounts.google.com/o/oauth2/auth",
    "token_uri": "https://oauth2.googleapis.com/token",
    "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
    "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/observability-for-kubernetes%40[YOUR_PROJECT_ID].iam.gserviceaccount.com"
  }  
```

### Encoding base64 and generate kube secret

`cat monitoring/thanos-objstore.yaml | base64`

### kubernetes secret

`vim monitoring/thanos-objstore-secret.yaml`

```
apiVersion: v1
kind: Secret
metadata:
  name: thanos-objectstorage
  namespace: monitoring
  labels:
    app: thanos
type: Opaque
data:
  thanos.yaml: xxxxxxxxx==<====
```

### Apply secret

```
kubectl --context cluster1 apply -f monitoring/thanos-objstore-secret.yaml
kubectl --context cluster2 apply -f monitoring/thanos-objstore-secret.yaml
```
### Create service for thanos-sidecars

```
kubectl --context=cluster1 -n monitoring apply -f monitoring/01-thanos-sidecar-svc.yaml
kubectl --context=cluster2 -n monitoring apply -f monitoring/01-thanos-sidecar-svc.yaml
```

### Install Thanos-store + Thanos-query

```
kubectl --context cluster1 apply -f monitoring/02-thanos-querier.yaml
```
