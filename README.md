# gitlab-runner
step to install gitlab runner in kubernetes
# Deploying GitLab Runner on Kubernetes with Custom Certificate and RBAC

This guide documents the steps taken to deploy a **GitLab Runner** in a Kubernetes cluster (namespace `gitlab`) with:

- A custom SSL certificate (self-signed CA).
- A dedicated ServiceAccount `gitlab-runner` with required RBAC permissions.
- Deployment via Helm with a `values.yaml` configuration.

---

## 1. Retrieve GitLab TLS Certificate

```bash
kubectl -n gitlab get secret gitlab-gitlab-tls -o jsonpath="{.data.tls\.crt}" | base64 -d > gitlab.mirecloud.com.crt
```

Create a Kubernetes secret `gitlab-ca` containing the certificate:

```bash
kubectl -n gitlab create secret generic gitlab-ca   --from-file=gitlab.mirecloud.com.crt=gitlab.mirecloud.com.crt
```

---

## 2. Create ServiceAccount + RBAC

### ServiceAccount
```bash
kubectl -n gitlab create serviceaccount gitlab-runner
```

### ClusterRole
`clusterrole.yaml`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: gitlab-runner
rules:
- apiGroups: [""]
  resources: ["pods", "pods/exec", "secrets", "configmaps", "services", "persistentvolumeclaims", "pods/log", "pods/attach"]
  verbs: ["get", "list", "watch", "create", "delete", "update", "patch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "delete", "update", "patch"]
```

### ClusterRoleBinding
`clusterrolebinding.yaml`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gitlab-runner
subjects:
- kind: ServiceAccount
  name: gitlab-runner
  namespace: gitlab
roleRef:
  kind: ClusterRole
  name: gitlab-runner
  apiGroup: rbac.authorization.k8s.io
```

Apply:
```bash
kubectl apply -f clusterrole.yaml
kubectl apply -f clusterrolebinding.yaml
```

---

## 3. Deploy GitLab Runner with Helm

File `values.yaml`:

```yaml
gitlabUrl: https://gitlab.mirecloud.com
runnerRegistrationToken: "glrt-xxxxx"

certsSecretName: gitlab-ca

runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        namespace = "gitlab"
        image = "alpine:latest"
        service_account = "gitlab-runner"
```

Install / upgrade:
```bash
helm upgrade --install gitlab-runner gitlab/gitlab-runner -n gitlab -f values.yaml
```

---

## 4. Force ServiceAccount usage

Patch deployment to ensure the runner uses `gitlab-runner`:

```bash
kubectl -n gitlab patch deployment gitlab-runner   -p '{"spec":{"template":{"spec":{"serviceAccountName":"gitlab-runner"}}}}'
```

Verify:
```bash
kubectl -n gitlab get pod -l app=gitlab-runner -o jsonpath="{.items[0].spec.serviceAccountName}"
# should return "gitlab-runner"
```

---

## 5. Verification

Runner logs:
```bash
kubectl -n gitlab logs -f deploy/gitlab-runner
```

Run a test job → ✅ Runner registered and jobs executed in Kubernetes.

---

## Final Result

- Runner successfully registered with GitLab.  
- Custom SSL certificate recognized (`gitlab-ca`).  
- Runner uses a dedicated ServiceAccount with full RBAC.  
- GitLab CI/CD jobs are executed in the `gitlab` namespace.  
