For production, creating a **dedicated ServiceAccount with limited permissions** and generating a kubeconfig for it is much safer than using `/etc/kubernetes/admin.conf`.

## 1. Create ServiceAccount

```bash
kubectl create serviceaccount ai-k8s-manager -n kube-system
```

Verify:

```bash
kubectl get sa ai-k8s-manager -n kube-system
```

---

## 2. Create Read-Only ClusterRole

Create `readonly-clusterrole.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ai-k8s-readonly
rules:
- apiGroups: [""]
  resources:
  - pods
  - services
  - endpoints
  - namespaces
  - nodes
  - events
  - persistentvolumeclaims
  - persistentvolumes
  verbs:
  - get
  - list
  - watch

- apiGroups: ["apps"]
  resources:
  - deployments
  - daemonsets
  - statefulsets
  - replicasets
  verbs:
  - get
  - list
  - watch

- apiGroups: ["batch"]
  resources:
  - jobs
  - cronjobs
  verbs:
  - get
  - list
  - watch
```

Apply:

```bash
kubectl apply -f readonly-clusterrole.yaml
```

---

## 3. Bind Role to ServiceAccount

```bash
kubectl create clusterrolebinding ai-k8s-readonly-binding \
  --clusterrole=ai-k8s-readonly \
  --serviceaccount=kube-system:ai-k8s-manager
```

Verify:

```bash
kubectl get clusterrolebinding ai-k8s-readonly-binding
```

---

## 4. Generate ServiceAccount Token

For Kubernetes v1.24+:

```bash
kubectl create token ai-k8s-manager -n kube-system
```

Example output:

```text
eyJhbGciOiJSUzI1NiIsImtpZCI6...
```

Save it:

```bash
TOKEN=$(kubectl create token ai-k8s-manager -n kube-system)
echo $TOKEN
```

---

## 5. Get Cluster Information

### Cluster Name

```bash
kubectl config current-context
```

Example:

```text
kubernetes-admin@kubernetes
```

### API Server

```bash
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'
```

Example:

```text
https://10.10.10.100:6443
```

### CA Certificate

```bash
kubectl config view --raw --minify \
-o jsonpath='{.clusters[0].cluster.certificate-authority-data}'
```

Copy the output.

---

## 6. Create Kubeconfig

Create `ai-k8s-manager.kubeconfig`:

```yaml
apiVersion: v1
kind: Config

clusters:
- name: kubernetes
  cluster:
    server: https://10.10.10.100:6443
    certificate-authority-data: <PASTE_CA_DATA>

users:
- name: ai-k8s-manager
  user:
    token: <PASTE_TOKEN>

contexts:
- name: ai-k8s-manager-context
  context:
    cluster: kubernetes
    user: ai-k8s-manager

current-context: ai-k8s-manager-context
```

---

## 7. Test the Kubeconfig

```bash
kubectl \
  --kubeconfig ai-k8s-manager.kubeconfig \
  get nodes
```

Test pods:

```bash
kubectl \
  --kubeconfig ai-k8s-manager.kubeconfig \
  get pods -A
```

Test deployments:

```bash
kubectl \
  --kubeconfig ai-k8s-manager.kubeconfig \
  get deploy -A
```

---

## 8. Copy to Jumpbox

```bash
scp ai-k8s-manager.kubeconfig \
user@jumpbox:/home/user/.kube/dev-cluster-config
```

Then test from jumpbox:

```bash
kubectl \
  --kubeconfig ~/.kube/dev-cluster-config \
  get nodes
```

---

## 9. Verify Permissions

This should work:

```bash
kubectl --kubeconfig ai-k8s-manager.kubeconfig get pods -A
```

This should fail:

```bash
kubectl --kubeconfig ai-k8s-manager.kubeconfig delete pod nginx
```

Expected:

```text
Error from server (Forbidden)
```

That confirms the ServiceAccount has read-only access.

---

## For Your AI Project

I recommend **two ServiceAccounts per cluster**:

1. **ai-k8s-readonly** (default)

   * List pods
   * Read logs
   * Read events
   * Read deployments
   * AI diagnostics

2. **ai-k8s-operator** (optional)

   * Restart deployments
   * Scale deployments
   * Rollout restart

Use the read-only kubeconfig for AI analysis and reserve the operator kubeconfig for explicit user-approved actions. This follows the principle of least privilege and is safer for production clusters.
