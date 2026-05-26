**Here are complete, ready-to-use manifests** for deploying **Elastic Agent as a DaemonSet** using a **ConfigMap** for `elastic-agent.yml`.

### 1. `configmap.yaml` (with Elasticsearch Root CA)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: elastic-agent-config
  namespace: kube-system
  labels:
    app.kubernetes.io/name: elastic-agent
data:
  elastic-agent.yml: |-
    # ================================= Fleet =====================================
    fleet:
      enabled: true
      url: "https://your-fleet-server:8220"          # Change to your Fleet Server URL
      enrollment_token: "YOUR_ENROLLMENT_TOKEN_HERE" # Get from Fleet UI

    # ============================== Outputs =====================================
    outputs:
      default:
        type: elasticsearch
        hosts: ["https://your-elasticsearch:9200"]   # Change to your ES hosts
        username: "elastic"
        password: "your-password"                    # Use API Key in production (recommended)

        ssl:
          # === Option 1: Inline Root CA (recommended) ===
          certificate_authorities:
            - |
              -----BEGIN CERTIFICATE-----
              MIID...your-full-root-ca-pem-content-here...
              -----END CERTIFICATE-----

          # === Option 2: Use fingerprint instead (Elastic 8.0+) ===
          # ca_trusted_fingerprint: "1a2b3c4d5e6f..."   # SHA-256 fingerprint of root CA

    # Optional: Reduce verbosity
    logging:
      level: info
```

> **Replace**:
> - `your-fleet-server:8220`
> - `YOUR_ENROLLMENT_TOKEN_HERE`
> - Elasticsearch hosts, credentials
> - Paste your actual **root CA certificate** (usually `http_ca.crt` from Elasticsearch)

---

### 2. `daemonset.yaml`

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: elastic-agent
  namespace: kube-system
  labels:
    app.kubernetes.io/name: elastic-agent
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: elastic-agent
  template:
    metadata:
      labels:
        app.kubernetes.io/name: elastic-agent
    spec:
      serviceAccountName: elastic-agent
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: elastic-agent
        image: docker.elastic.co/beats/elastic-agent:8.17.0   # Change version as needed
        env:
        - name: FLEET_ENROLL
          value: "1"
        - name: FLEET_URL
          value: "https://your-fleet-server:8220"
        - name: FLEET_ENROLLMENT_TOKEN
          value: "YOUR_ENROLLMENT_TOKEN_HERE"
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          runAsUser: 0
          privileged: true
        resources:
          limits:
            memory: 1Gi
          requests:
            cpu: 200m
            memory: 512Mi
        volumeMounts:
        - name: config
          mountPath: /usr/share/elastic-agent/elastic-agent.yml
          subPath: elastic-agent.yml
          readOnly: true
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: etc-full
          mountPath: /hostfs/etc
          readOnly: true
        - name: proc
          mountPath: /hostfs/proc
          readOnly: true
        - name: sys
          mountPath: /hostfs/sys
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: elastic-agent-config
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: varlog
        hostPath:
          path: /var/log
      - name: etc-full
        hostPath:
          path: /etc
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
```

---

### 3. Required RBAC (ServiceAccount + ClusterRole)

You also need these (create them first):

**rbac.yaml**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: elastic-agent
  namespace: kube-system
  labels:
    app.kubernetes.io/name: elastic-agent

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: elastic-agent
subjects:
- kind: ServiceAccount
  name: elastic-agent
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: elastic-agent
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: elastic-agent
  labels:
    app.kubernetes.io/name: elastic-agent
rules:
- apiGroups: [""]
  resources:
  - nodes
  - namespaces
  - pods
  - events
  - services
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["replicasets", "deployments", "daemonsets", "statefulsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "watch"]
```

---

### How to Deploy

```bash
# 1. Apply RBAC
kubectl apply -f rbac.yaml

# 2. Apply ConfigMap
kubectl apply -f configmap.yaml

# 3. Apply DaemonSet
kubectl apply -f daemonset.yaml
```

Check status:

```bash
kubectl -n kube-system get pods -l app.kubernetes.io/name=elastic-agent
kubectl -n kube-system logs -f daemonset/elastic-agent
```

Would you like me to adjust anything? For example:
- Use **Secret** for the CA instead of inlining
- Switch to **standalone mode** (no Fleet)
- Add specific integrations
- Use a different Elastic Agent version

Just let me know your Elasticsearch version and whether you're using Fleet or standalone.
