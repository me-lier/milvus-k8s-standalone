# milvus-k8s-standalone

> Production-ready **Milvus v2.6.0 standalone** on Kubernetes — persistent storage, Attu web UI, Ingress, and one-flag switch between Minikube and GKE.

Clone it, run three commands, and you have a fully working vector database inside your cluster — with data that survives pod restarts, redeployments, and accidental deletions.

---

## What's inside

```
┌──────────────────────────────── namespace: milvus ──────────────────────────────────┐
│                                                                                      │
│  ┌──────────┐     ┌───────────────┐     ┌──────────────────────┐     ┌──────────┐  │
│  │   etcd   │     │     minio     │     │   milvus standalone  │     │   attu   │  │
│  │  :2379   │◄────│  api  :9000   │◄────│   gRPC     :19530    │◄────│  :3000   │  │
│  │  1Gi PVC │     │  ui   :9001   │     │   metrics  :9091     │     │  web UI  │  │
│  └──────────┘     │  10Gi PVC     │     │   3Gi PVC            │     └──────────┘  │
│                   └───────────────┘     └──────────────────────┘                   │
│                                                    ▲                                │
└────────────────────────────────────────────────────│────────────────────────────────┘
                                                     │
                         any pod in any namespace connects via:
                         milvus.milvus.svc.cluster.local:19530
```

**Persistent by design** — all three stateful components use `reclaimPolicy: Retain`:
- Pod crash → data survives
- `kubectl delete deployment` → data survives  
- Accidental PVC deletion → underlying disk is **not** deleted

---

## Files

```
milvus/
├── namespace.yaml                     # milvus namespace
├── milvus-storage-class.yaml          # Retain policy StorageClass (swap 1 line for GKE)
│
├── etcd-pvc.yaml                      # 1Gi  — collection schemas, session data
├── etcd-deployment.yaml               # etcd v3.5.15
├── etcd-service.yaml                  # ClusterIP :2379
│
├── minio-pvc.yaml                     # 10Gi — vector index files, segment data
├── minio-deployment.yaml              # MinIO object storage
├── minio-service.yaml                 # ClusterIP :9000 (API)  :9001 (console)
│
├── milvus-pvc.yaml                    # 3Gi  — WAL and local buffers
├── milvus-standalone-deployment.yaml  # Milvus v2.6.0
├── milvus-standalone-service.yaml     # NodePort :19530 (gRPC)  :9091 (metrics)
│
├── attu-deployment.yaml               # Attu v2.4 — visual admin UI
├── attu-service.yaml                  # NodePort :3000
└── attu-ingress.yaml                  # Ingress → attu.milvus.local + minio.milvus.local
```

---

## Prerequisites

| Requirement | Minikube | GKE |
|---|---|---|
| kubectl | ✓ | ✓ |
| Cluster RAM | 4 GB (`minikube start --memory=4096`) | n1-standard-2 or higher |
| Disk | 14 GB available | 14 GB on node |
| Ingress | `minikube addons enable ingress` | nginx-ingress controller |

---

## Quick start

```bash
# Clone
git clone https://github.com/YOUR_USERNAME/milvus-k8s-standalone.git
cd milvus-k8s-standalone

# Deploy everything (initContainers handle startup ordering)
kubectl apply -f namespace.yaml
kubectl apply -f milvus-storage-class.yaml
kubectl apply -f .

# Watch pods come up (takes ~60s)
kubectl get pods -n milvus -w
```

**Expected output once ready:**
```
NAME                     READY   STATUS    RESTARTS   AGE
attu-xxx                 1/1     Running   0          2m
etcd-xxx                 1/1     Running   0          3m
milvus-xxx               1/1     Running   0          2m
minio-xxx                1/1     Running   0          3m
```

```bash
# Confirm all PVCs are Bound
kubectl get pvc -n milvus
```

---

## Access

### Attu — visual admin UI

The easiest way to explore your collections, run queries, and manage users.

```bash
# Option A: port-forward
kubectl port-forward svc/attu 3000:3000 -n milvus
# open http://localhost:3000

# Option B: Minikube URL
minikube service attu -n milvus
```

Login: **root** / **Milvus**

What you can do in Attu:
- Browse and create collections (tables for vectors)
- Insert, query, and delete vectors visually
- Run ANN (Approximate Nearest Neighbour) searches with filters
- Manage users, roles, and permissions
- Monitor index status and collection stats

### Milvus gRPC — connect from your code

```bash
# Port-forward for local development
kubectl port-forward svc/milvus 19530:19530 -n milvus
# connect to localhost:19530
```

### MinIO console — inspect stored files

```bash
kubectl port-forward svc/minio 9001:9001 -n milvus
# open http://localhost:9001
```

Login: **minioadmin** / **minioadmin**

---

## Ingress — access via hostname (no port-forward)

### Minikube

```bash
# Enable ingress addon (one-time)
minikube addons enable ingress

# Get your minikube IP and add hostnames
echo "$(minikube ip)  attu.milvus.local minio.milvus.local" | sudo tee -a /etc/hosts

# Apply ingress
kubectl apply -f attu-ingress.yaml
```

Open in browser:
- **http://attu.milvus.local** — Attu UI
- **http://minio.milvus.local** — MinIO console

### GKE

1. Install nginx ingress controller:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml
   ```

2. Edit `attu-ingress.yaml` — replace hosts with your real domain:
   ```yaml
   - host: attu.yourdomain.com
   - host: minio.yourdomain.com
   ```

3. Apply and get the external IP:
   ```bash
   kubectl apply -f attu-ingress.yaml
   kubectl get ingress -n milvus   # copy EXTERNAL-IP → point your DNS there
   ```

---

## Connecting your service to Milvus

### From inside the cluster (recommended)

Any pod in any namespace can reach Milvus using the full service DNS:

```
URI:   http://milvus.milvus.svc.cluster.local:19530
Token: root:Milvus
```

### Python — pymilvus

```python
from pymilvus import MilvusClient

client = MilvusClient(
    uri="http://milvus.milvus.svc.cluster.local:19530",
    token="root:Milvus"
)

# Create a collection
client.create_collection(
    collection_name="my_vectors",
    dimension=384
)

# Insert vectors
client.insert("my_vectors", [
    {"id": 1, "vector": [0.1, 0.2, ...], "text": "hello world"},
])

# Search
results = client.search(
    collection_name="my_vectors",
    data=[[0.1, 0.2, ...]],   # your query embedding
    limit=5,
    output_fields=["text"]
)
```

### Node.js — @zilliz/milvus2-sdk-node

```js
const { MilvusClient } = require('@zilliz/milvus2-sdk-node')

const client = new MilvusClient({
  address: 'milvus.milvus.svc.cluster.local:19530',
  token: 'root:Milvus',
})
```

### Go — milvus-sdk-go

```go
client, err := gomilvus.NewClient(ctx, gomilvus.Config{
    Address: "milvus.milvus.svc.cluster.local:19530",
    APIKey:  "root:Milvus",
})
```

### Kubernetes Secret for your Deployment

Create this in your app's namespace — not in `milvus`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: milvus-credentials
  namespace: your-app-namespace
type: Opaque
stringData:
  MILVUS_URI: "http://milvus.milvus.svc.cluster.local:19530"
  MILVUS_TOKEN: "root:Milvus"
```

Reference it in your Deployment:

```yaml
containers:
- name: your-app
  envFrom:
  - secretRef:
      name: milvus-credentials
```

Your app then reads `os.environ["MILVUS_URI"]` and `os.environ["MILVUS_TOKEN"]` — no hardcoded addresses.

---

## Use cases

### 1. RAG — Retrieval-Augmented Generation

Build a chatbot or Q&A system that answers questions from your own documents (PDFs, wikis, support tickets).

**How it works:**
1. Split documents into chunks
2. Embed each chunk with a model (e.g. `BAAI/bge-m3`, `text-embedding-3-small`)
3. Store chunk + embedding in Milvus
4. At query time: embed the question → search Milvus for similar chunks → pass top-K chunks to your LLM as context

```python
# Store document chunks
client.insert("documents", [
    {"id": 1, "vector": embed("Kubernetes is an orchestration system..."), "text": "...", "source": "k8s-docs.pdf"},
])

# Retrieve relevant chunks for a question
results = client.search(
    collection_name="documents",
    data=[embed("How do I deploy a pod?")],
    limit=5,
    output_fields=["text", "source"]
)
# Pass results[0]["text"] as context to your LLM
```

**Best for:** customer support bots, internal knowledge bases, document Q&A, code assistant.

---

### 2. Semantic search

Search by meaning instead of keywords. "show me running shoes" finds "sneakers for jogging" even with zero keyword overlap.

```python
client.create_collection("products", dimension=384)

# Index your product catalog
client.insert("products", [
    {"id": 101, "vector": embed("red running shoes for marathon"), "name": "Nike Air Zoom", "price": 120},
])

# Search by natural language
results = client.search(
    collection_name="products",
    data=[embed("sneakers for long distance running")],
    limit=10,
    filter="price < 150",          # combine with metadata filters
    output_fields=["name", "price"]
)
```

**Best for:** e-commerce search, content discovery, job matching, legal document search.

---

### 3. Image / multimodal similarity

Find visually similar images, detect near-duplicate photos, or search by image + text together.

```python
# Store image embeddings (CLIP, ViT, etc.)
client.create_collection("images", dimension=512)

client.insert("images", [
    {"id": 1, "vector": clip_encode(image_bytes), "url": "s3://bucket/img1.jpg", "label": "cat"},
])

# Find similar images
results = client.search(
    collection_name="images",
    data=[clip_encode(query_image)],
    limit=20,
    output_fields=["url", "label"]
)
```

**Best for:** reverse image search, content moderation deduplication, medical imaging, fashion visual search.

---

### 4. Recommendation engine

Personalise feeds, suggest products, or recommend content based on user behaviour.

```python
# User and item embeddings from a model (e.g. two-tower network)
client.create_collection("items", dimension=256)

client.insert("items", [
    {"id": 42, "vector": item_embedding, "category": "electronics", "rating": 4.5},
])

# Get recommendations for a user
user_vec = get_user_embedding(user_id)
results = client.search(
    collection_name="items",
    data=[user_vec],
    limit=20,
    filter='category == "electronics" and rating > 4.0',
    output_fields=["id", "category"]
)
```

**Best for:** content feeds, e-commerce recommendations, music/video suggestions, social graph matching.

---

### 5. Anomaly detection

Embed time-series windows or log lines and flag anything that lands far from the normal cluster.

```python
client.create_collection("logs", dimension=128)

# During normal operation: store log embeddings as baseline
client.insert("logs", [
    {"id": ts, "vector": embed_log(log_line), "service": "payment-api", "level": "INFO"},
])

# At inference: if nearest-neighbour distance > threshold → anomaly
results = client.search("logs", data=[embed_log(new_log)], limit=1)
if results[0]["distance"] > THRESHOLD:
    alert("Anomaly detected!")
```

**Best for:** infrastructure monitoring, fraud detection, intrusion detection, log anomaly alerting.

---

### 6. Duplicate and near-duplicate detection

Find if a new piece of content (text, image, code) is a near-copy of something already stored.

```python
# Before storing a new article
results = client.search(
    collection_name="articles",
    data=[embed(new_article_text)],
    limit=1
)

if results[0]["distance"] > 0.95:   # cosine similarity threshold
    print(f"Near-duplicate of article {results[0]['id']} — skipping")
else:
    client.insert("articles", [{"vector": embed(new_article_text), ...}])
```

**Best for:** content deduplication pipelines, plagiarism detection, social media moderation, patent search.

---

### 7. Multi-tenant vector search

Run one Milvus cluster for multiple teams or services — each gets its own collection with separate access controls.

```python
# Team A: support tickets
client.create_collection("team_a_tickets", dimension=384)

# Team B: product catalog
client.create_collection("team_b_products", dimension=512)

# Create a dedicated user for each team
from pymilvus import utility
utility.create_user("team_a_user", "secret", using="default")
utility.grant_privilege("team_a_user", "team_a_tickets", "Search")
```

**Best for:** SaaS platforms, internal platform teams, multi-product orgs.

---

## Switch to GKE

Only two files change:

**`milvus-storage-class.yaml`** — swap the provisioner line:
```yaml
# Minikube (comment out):
# provisioner: k8s.io/minikube-hostpath

# GKE (uncomment):
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-balanced          # pd-standard | pd-balanced | pd-ssd
```

**`milvus-standalone-service.yaml`** and **`attu-service.yaml`** — change service type:
```yaml
type: LoadBalancer            # was: NodePort
```

Everything else — PVCs, Deployments, environment variables, internal DNS — is identical.

---

## Storage reference

| Component | Claim | Size | Reclaim | Stores |
|---|---|---|---|---|
| etcd | `etcd-pvc` | 1 Gi | **Retain** | Collection schemas, indexes, user/role definitions, session IDs |
| MinIO | `minio-pvc` | 10 Gi | **Retain** | Vector index files (HNSW, IVF), segment binlogs, snapshots |
| Milvus | `milvus-pvc` | 3 Gi | **Retain** | Write-ahead log, local segment buffers |

> **Retain** = deleting the PVC does NOT delete the disk. To free storage you must also delete the PersistentVolume manually.

---

## Teardown

```bash
# Delete all Kubernetes resources — data on disk is KEPT (Retain policy)
kubectl delete namespace milvus
kubectl delete storageclass retain-pd

# ⚠️  Only run this if you want to permanently delete all vector data
kubectl get pv                        # find pv names bound to milvus claims
kubectl delete pv <pv-name-etcd>
kubectl delete pv <pv-name-minio>
kubectl delete pv <pv-name-milvus>
```

---

## Troubleshooting

**Milvus in CrashLoopBackOff after a redeploy**
etcd has stale session data from the previous crashed instance.
```bash
kubectl exec -n milvus deploy/etcd -- etcdctl del --prefix by-dev
kubectl delete pod -n milvus -l app=milvus
```

**PVC stuck in `Pending`**
The StorageClass provisioner doesn't match your environment.
Edit `milvus-storage-class.yaml` and switch the provisioner (`k8s.io/minikube-hostpath` ↔ `pd.csi.storage.gke.io`).

**Can't connect from another namespace**
Use the full DNS: `milvus.milvus.svc.cluster.local:19530`.  
Short names like `milvus:19530` only resolve within the same namespace.

**Attu shows "cannot connect to Milvus"**
Confirm Milvus is healthy first:
```bash
kubectl get pods -n milvus
kubectl logs -n milvus deploy/milvus --tail=30
```

**View logs**
```bash
kubectl logs -n milvus deploy/milvus  --tail=50 -f
kubectl logs -n milvus deploy/etcd    --tail=20
kubectl logs -n milvus deploy/minio   --tail=20
kubectl logs -n milvus deploy/attu    --tail=20
```

---

## Component versions

| Component | Image |
|---|---|
| Milvus | `milvusdb/milvus:v2.6.0` |
| etcd | `quay.io/coreos/etcd:v3.5.15` |
| MinIO | `minio/minio:RELEASE.2024-01-16T16-07-38Z` |
| Attu | `zilliz/attu:v2.4` |

---

## License

MIT
