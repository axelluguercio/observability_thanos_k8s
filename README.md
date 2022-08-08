# observability thanos

### Create 2 cluster lamp_app with prometheus 

```
kubectl --context=cluster1 apply -f lamp_app/mysql/
kubectl --context=cluster1 apply -f lamp_app/php/
kubectl --context=cluster1 apply -f lamp_app/phpmyadmin/
kubectl --context=cluster1 apply -f monitoring/prometheus/setup
kubectl --context=cluster1 apply -f monitoring/prometheus/

kubectl --context=cluster2 apply -f lamp_app/mysql/
kubectl --context=cluster2 apply -f lamp_app/php/
kubectl --context=cluster2 apply -f lamp_app/phpmyadmin/
kubectl --context=cluster2 apply -f monitoring/prometheus/setup
kubectl --context=cluster2 apply -f monitoring/prometheus/
```

### Expose the application with an ingress controller

`kubectl apply -f lamp_app/ingress_controller.yaml`

### Generar secret for GCP bucket to store metrics 

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
