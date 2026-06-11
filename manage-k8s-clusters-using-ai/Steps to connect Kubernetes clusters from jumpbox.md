To connect Kubernetes clusters from your jumpbox, the most common and secure approach is to use **kubeconfig files** for each cluster.

# Step 1: Verify Network Connectivity

From the jumpbox, verify you can reach the Kubernetes API server.

Example:

```bash
nc -zv <api-server-ip> 6443
```

or

```bash
curl -k https://<api-server-ip>:6443/version
```

Example:

```bash
curl -k https://10.10.10.100:6443/version
```

If this fails, check:

* Firewall rules
* Security groups
* VPN connectivity
* Routing between jumpbox and cluster

---

# Step 2: Copy Kubeconfig from Cluster

On the Kubernetes control plane:

```bash
sudo cat /etc/kubernetes/admin.conf
```

Copy the content to your jumpbox.

Create:

```bash
mkdir -p ~/.kube
nano ~/.kube/dev-config
```

Paste the content and save.

Set permissions:

```bash
chmod 600 ~/.kube/dev-config
```

---

# Step 3: Test Cluster Access

Use the kubeconfig directly:

```bash
kubectl --kubeconfig ~/.kube/dev-config get nodes
```

Expected output:

```text
NAME          STATUS   ROLES           AGE
master-01     Ready    control-plane   10d
worker-01     Ready    <none>          10d
worker-02     Ready    <none>          10d
```

---

# Step 4: Add Multiple Clusters

Example:

```text
~/.kube/
├── dev-config
├── test-config
├── prod-config
```

Verify:

```bash
kubectl --kubeconfig ~/.kube/dev-config get nodes
kubectl --kubeconfig ~/.kube/test-config get nodes
kubectl --kubeconfig ~/.kube/prod-config get nodes
```

---

# Step 5: Create a Combined Kubeconfig (Optional)

Merge all kubeconfigs:

```bash
export KUBECONFIG=~/.kube/dev-config:~/.kube/test-config:~/.kube/prod-config

kubectl config view --flatten > ~/.kube/config

unset KUBECONFIG
```

Check contexts:

```bash
kubectl config get-contexts
```

Example:

```text
CURRENT   NAME
*         dev
          test
          prod
```

Switch clusters:

```bash
kubectl config use-context dev
kubectl config use-context test
kubectl config use-context prod
```

---

# Step 6: Test Using Python

Install client:

```bash
pip install kubernetes
```

Create `test.py`:

```python
from kubernetes import client, config

config.load_kube_config(
    config_file="/root/.kube/dev-config"
)

v1 = client.CoreV1Api()

for node in v1.list_node().items:
    print(node.metadata.name)
```

Run:

```bash
python test.py
```

---

# Step 7: Store Cluster Definitions for Your Application

Create `config/clusters.yaml`

```yaml
clusters:
  - name: Dev Cluster
    kubeconfig: /root/.kube/dev-config

  - name: Test Cluster
    kubeconfig: /root/.kube/test-config

  - name: Prod Cluster
    kubeconfig: /root/.kube/prod-config
```

Load it in Python:

```python
import yaml

with open("config/clusters.yaml") as f:
    cfg = yaml.safe_load(f)

print(cfg)
```

---

# Step 8: Secure Access for Production

Avoid using `admin.conf` directly in production.

Create a dedicated service account:

```bash
kubectl create serviceaccount ai-k8s-manager -n kube-system
```

Create a read-only ClusterRole:

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
    - nodes
    - namespaces
    - events
  verbs:
    - get
    - list
    - watch
```

Bind it:

```bash
kubectl create clusterrolebinding ai-k8s-readonly-binding \
  --clusterrole=ai-k8s-readonly \
  --serviceaccount=kube-system:ai-k8s-manager
```

Then generate a kubeconfig using that service account instead of `admin.conf`.

---

# Step 9: Validate for Your AI Project

From the jumpbox, confirm these work:

```bash
kubectl --kubeconfig ~/.kube/dev-config get nodes
kubectl --kubeconfig ~/.kube/dev-config get pods -A
kubectl --kubeconfig ~/.kube/dev-config get events -A
kubectl --kubeconfig ~/.kube/dev-config top nodes
```

## Better approach for your AI K8s Manager

Instead of embedding a temporary token in `ai-k8s-manager.kubeconfig`, use one of these:

### Option 1: Use a ServiceAccount token Secret (long-lived)

Create a secret manually:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ai-k8s-manager-token
  namespace: kube-system
  annotations:
    kubernetes.io/service-account.name: ai-k8s-manager
type: kubernetes.io/service-account-token
```

Apply:

```bash
kubectl apply -f sa-token-secret.yaml
```

Get the token:

```bash
kubectl get secret ai-k8s-manager-token \
  -n kube-system \
  -o jsonpath='{.data.token}' | base64 -d
```

Use this token in your kubeconfig.

This token typically remains valid until:

* the ServiceAccount is deleted,
* the Secret is deleted,
* or cluster signing keys change.

---

### Option 2: Use a kubeconfig generated from a client certificate

For production automation, certificate-based authentication is generally preferred over static tokens.

---

### Option 3: Dynamic token generation in your Streamlit app

Since you're building an AI Kubernetes management platform, a common pattern is:

1. Store an admin kubeconfig securely on the jumpbox.
2. When the application starts:

   * Generate a fresh token:

     ```bash
     kubectl create token ai-k8s-manager -n kube-system
     ```
3. Build the Kubernetes client configuration dynamically.
4. Refresh the token before it expires.

This avoids manual kubeconfig updates.

---

For your **"manage-k8s-clusters-using-ai"** Streamlit project, I recommend:

* Create a dedicated ServiceAccount (`ai-k8s-manager`).
* Bind it to a least-privilege ClusterRole (your `ai-k8s-readonly` role is a good start).
* Create a `kubernetes.io/service-account-token` Secret and use that token in the application's kubeconfig.

That will prevent the recurring `Unauthorized` errors caused by expiring tokens.

kubectl --kubeconfig ~/.kube/dev-config top pods -A
```

Once these commands work, your Streamlit + Ollama application can use the same kubeconfig files through the Kubernetes Python client to collect cluster state and provide AI-assisted diagnostics.
