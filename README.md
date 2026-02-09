# YouTrack + Strimzi Kafka — Dev Environment

## Prerequisites

- Kubernetes cluster (1.25+)
- `kubectl` configured
- A StorageClass available (or adjust `storageClassName` in PVCs)
- An ingress controller (nginx-ingress shown; adjust annotations for Traefik/etc.)

## Deploy

```bash
# 1. Install Strimzi operator (pick one — must be done before applying kustomization)
# Option A: Direct install
kubectl create namespace youtrack-dev
kubectl create -f 'https://strimzi.io/install/latest?namespace=youtrack-dev' -n youtrack-dev

# Option B: Helm
kubectl create namespace youtrack-dev
helm repo add strimzi https://strimzi.io/charts/
helm install strimzi-operator strimzi/strimzi-kafka-operator -n youtrack-dev

# 2. Wait for operator to be ready
kubectl wait deployment/strimzi-cluster-operator --for=condition=Available -n youtrack-dev --timeout=120s

# 3. Apply everything via kustomize
kubectl apply -k .

# 4. Wait for services to come up
kubectl wait kafka/youtrack-kafka --for=condition=Ready -n youtrack-dev --timeout=300s
kubectl wait kafkabridge/youtrack-bridge --for=condition=Ready -n youtrack-dev --timeout=120s
kubectl rollout status statefulset/youtrack -n youtrack-dev --timeout=600s
```

## Accessing YouTrack

- **Via Ingress**: Update `youtrack-dev.example.com` to your actual domain
- **Port-forward** (quick access): `kubectl port-forward svc/youtrack 8080:8080 -n youtrack-dev`
- First-time setup wizard runs on initial access — set admin credentials and configure license

## Kafka Bootstrap Server

From within the cluster, Kafka is reachable at:

```
youtrack-kafka-kafka-bootstrap.youtrack-dev.svc.cluster.local:9092  (plaintext)
youtrack-kafka-kafka-bootstrap.youtrack-dev.svc.cluster.local:9093  (TLS)
```

## Kafka HTTP Bridge

The Strimzi HTTP Bridge accepts REST calls and produces/consumes to Kafka:

```
http://youtrack-bridge-bridge-service.youtrack-dev.svc.cluster.local:8080
```

Produce to a topic:
```bash
curl -X POST http://youtrack-bridge-bridge-service:8080/topics/youtrack-events \
  -H 'Content-Type: application/vnd.kafka.json.v2+json' \
  -d '{"records": [{"key": "TEST-1", "value": {"eventType": "CREATED", "summary": "test"}}]}'
```

## What to Customize

| Item | Where | Notes |
|------|-------|-------|
| Domain / Base URL | `20-youtrack.yaml` Ingress + `YOUTRACK_BASE_URL` env | Match your DNS |
| Storage sizes | PVCs in `20-youtrack.yaml`, storage in `10-strimzi-kafka.yaml` | Scale for your data |
| Resource limits | Both files | Increase for heavier usage |
| TLS | Ingress TLS block + cert-manager annotation | Uncomment when ready |
| Kafka topics | `10-strimzi-kafka.yaml` KafkaTopic | Add topics your integrations need |
| YouTrack image tag | `20-youtrack.yaml` | Pin to the version matching your airgapped instance |

## Teardown

```bash
kubectl delete -k .
# Strimzi operator:
kubectl delete -f 'https://strimzi.io/install/latest?namespace=youtrack-dev' -n youtrack-dev
kubectl delete namespace youtrack-dev
```
