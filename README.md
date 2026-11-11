# Kubernetes Learning Journey

This repo holds the manifests I wrote while learning Kubernetes during my internship. The files go roughly from the basics (Pods, labels, ReplicaSets) up to Ingress, TLS with cert-manager, and policy enforcement with Kyverno.

It ends with a working application on a local **kind** cluster: two services (`test` and `web-page`) behind an NGINX Ingress on `priceconverter.com`, served over HTTPS.

If you're learning Kubernetes too, you can follow the files in the order below and apply them yourself.

---

## Architecture of the deployed app

```
                    https://priceconverter.com
                               │
                    ┌──────────▼──────────┐
                    │ NGINX Ingress        │  (ingress-controller.yaml)
                    │ TLS secret: tls-cer  │  (certificate.yaml via cert-manager)
                    └──────┬────────┬─────┘
             /path/...     │        │   /play/...
                ┌──────────▼──┐  ┌──▼──────────┐
                │ Service:test│  │Service:     │
                │ port 80     │  │web-page :80 │
                └──────┬──────┘  └──────┬──────┘
                ┌──────▼──────┐  ┌──────▼──────┐
                │ Deployment: │  │ web-page    │
                │ test (:8080)│  │ pods        │
                └─────────────┘  └─────────────┘
```

The Ingress uses `rewrite-target: /$2`, so a request to `/path/foo` gets forwarded to the backend as `/foo`.

---

## Prerequisites

| Tool | Purpose |
|------|---------|
| [Docker](https://docs.docker.com/get-docker/) | Runs the kind nodes as containers |
| [kind](https://kind.sigs.k8s.io/) | Local Kubernetes cluster |
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | Talks to the cluster |
| [cert-manager](https://cert-manager.io/docs/installation/) | Issues the TLS certificates |
| [Kyverno](https://kyverno.io/docs/installation/) | Policy engine (only for the policy section) |
| [Trivy](https://trivy.dev/) | Image vulnerability scanning (optional) |

---

## Learning path (files in order)

### 1. Create the cluster

| File | What it does |
|------|--------------|
| [cluster.yml](cluster.yml) | kind config: 1 control-plane node and 1 worker node, both on `kindest/node:v1.37.0` |

```bash
kind create cluster --name learning-k8s --config cluster.yml
kubectl get nodes
```

### 2. Pods and labels

| File | What it does |
|------|--------------|
| [labels-test.yaml](labels-test.yaml) | A single nginx Pod with several labels (`environmet`, `release`, `tier`, `owner`) for practising label selectors |
| [test.yaml](test.yaml) | Output of `kubectl get pod test2 -o yaml`. It shows everything Kubernetes adds to a Pod on its own (status, tolerations, the service-account volume, QoS class, and so on). It's for reading, not for applying. |
| [testing_init.yaml](testing_init.yaml) | **Init containers**: an init container writes a file to a shared `emptyDir` volume, then the main container reads it |
| [resource_monitor.yaml](resource_monitor.yaml) | A Pod with `hostPID: true` that runs `top` in a loop to watch node processes, with resource requests and limits set |

```bash
kubectl apply -f labels-test.yaml
kubectl get pods --show-labels
kubectl get pods -l tier=kubecloud

kubectl apply -f testing_init.yaml
kubectl logs init-pod-test            # -> "Hello from the init container! ..."
```

### 3. Controllers: ReplicationController → ReplicaSet → Deployment

| File | What it does |
|------|--------------|
| [rc.yaml](rc.yaml) | ReplicationController, the older way to keep N replicas running. Note: it contains typos (`tempelate`, `lables`), so `kubectl apply` rejects it. Fixing those is a good exercise. |
| [rs.yaml](rs.yaml) | ReplicaSet with 3 nginx replicas, selected by `release: alphal.1` |
| [rollout-deploy.yaml](rollout-deploy.yaml) | Deployment with a **RollingUpdate** strategy (`maxSurge: 50%`, `maxUnavailable: 25%`) |

```bash
kubectl apply -f rs.yaml
kubectl delete pod <one-of-the-rs-pods>   # watch the ReplicaSet create a replacement
kubectl get pods -w

kubectl apply -f rollout-deploy.yaml
kubectl set image deployment/rollout-demo php-redis=nginx:1.27
kubectl rollout status deployment/rollout-demo
kubectl rollout history deployment/rollout-demo
kubectl rollout undo deployment/rollout-demo
```

### 4. Ingress controller

| File | What it does |
|------|--------------|
| [ingress-controller.yaml](ingress-controller.yaml) | The full manifest for the NGINX Ingress Controller (v1.15.1): namespace, RBAC, admission webhook, controller Deployment, and Service |

```bash
kubectl apply -f ingress-controller.yaml
kubectl get pods -n ingress-nginx -w
```

### 5. TLS with cert-manager

| File | What it does |
|------|--------------|
| [certmanager.yaml](certmanager.yaml) | `ClusterIssuer` named `letsencrypt-ci`, pointing at the Let's Encrypt **staging** ACME server with an HTTP-01 solver through nginx |
| [certificate.yaml](certificate.yaml) | `Certificate` for `priceconverter.com`, stored in the Secret `tls-cer` and issued by `selfsigned-issuer` |

Let's Encrypt can't reach a local kind cluster to complete the HTTP-01 challenge, so `certificate.yaml` uses a self-signed issuer instead. That issuer isn't in the repo yet. Create it like this:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
EOF

kubectl apply -f certificate.yaml
kubectl get certificate,secret tls-cer
```

### 6. The deployed application

| File | What it does |
|------|--------------|
| [ingress-test.yaml](ingress-test.yaml) | **test** app: a Deployment running `techwithprerit/k8s-hindi-app1:1.2-major` on port 8080, and a Service `test` that exposes it on port 80 |
| [ingress.yaml](ingress.yaml) | Ingress `ingress-kubecloud` for host `priceconverter.com` with TLS. It sends `/path` to the `test` service and `/play` to the `web-page` service |

The `web-page` Service and its Deployment aren't saved in this repo. To recreate an equivalent backend so the `/play` route works, run:

```bash
kubectl create deployment web-page --image=nginx
kubectl expose deployment web-page --port=80
```

Deploy and test:

```bash
kubectl apply -f ingress-test.yaml
kubectl apply -f ingress.yaml
kubectl get ingress

# Map the host name to the cluster (use your ingress/node IP)
echo "<INGRESS_IP> priceconverter.com" | sudo tee -a /etc/hosts

curl -k https://priceconverter.com/path/
curl -k https://priceconverter.com/play/
```

`-k` is needed because the certificate is self-signed.

> `ingress-test.yaml` sets `externalIPs: 172.21.0.3`, which was the worker node's IP in my kind network. Yours will probably differ. Check it with `docker inspect learning-k8s-worker`.

### 7. Policy as code with Kyverno

| File | What it does |
|------|--------------|
| [policy.yaml](policy.yaml) | `MutatingPolicy` **add-default-resources**: if a Pod has no CPU or memory requests, it sets them to `100m` and `100Mi`. Autogen also applies this to Deployments, Jobs, CronJobs, StatefulSets, and DaemonSets. |
| [policy1.yaml](policy1.yaml) | `MutatingPolicy` **add-labels**: adds `foo=bar` to every Pod, Service, ConfigMap, and Secret |
| [kata_policy.yaml](kata_policy.yaml) | `ClusterPolicy` **require-kata-runtime** (Enforce): rejects any Pod outside `kube-system` that doesn't set `runtimeClassName: kata` |

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm install kyverno kyverno/kyverno -n kyverno --create-namespace

kubectl apply -f policy.yaml -f policy1.yaml
kubectl run demo --image=nginx
kubectl get pod demo -o yaml | grep -A3 -E "labels|requests"   # foo=bar + default requests
```

> ⚠️ `kata_policy.yaml` blocks **every** Pod that doesn't use the Kata runtime. On a plain kind cluster without Kata installed, that includes the app above. Apply it only to see enforcement in action, then remove it with `kubectl delete -f kata_policy.yaml`.

### 8. Other files

| File | What it does |
|------|--------------|
| [cluster-state.yaml](cluster-state.yaml) | An empty `kubectl get ... -o yaml` export (`items: []`), kept as a snapshot |
| `trivy_*.deb` | Trivy installer used to scan container images for vulnerabilities (`trivy image nginx:latest`) |

---

## Concepts covered

- Cluster setup with kind (multi-node)
- Pods, labels, selectors, and init containers
- Resource requests and limits, and QoS classes
- ReplicationController vs ReplicaSet vs Deployment
- Rolling updates and rollbacks
- Services (ClusterIP, externalIPs)
- Ingress: host- and path-based routing and rewrite rules
- TLS: cert-manager, ClusterIssuers (ACME and self-signed), Certificates
- Admission control with Kyverno: mutating and validating policies
- Container image scanning with Trivy

## Cleanup

```bash
kind delete cluster --name learning-k8s
```
