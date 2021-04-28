# Grant admin access

Limited access to a specific namespaced application can be granted for developers and other project contributors to use via a **Service Account**.

## Prerequisites

- An operational [shared cluster](./create-cluster.md)

## Uses

A service account and associated role can be used, for example, to:

- Grant read-only access to list pods within a namespace
- Grant access to execute commands on running pods
- Grant access to forward network ports to access internal services securely

## Manifest template

In the following template, replace `MY_NAMESPACE` and `MY_ADMIN` with the namespace you're granting access to and a chosen name for the admin account. This template grants any user of the declared service account the three capabilities described above:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: MY_NAMESPACE

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: MY_ADMIN
  namespace: MY_NAMESPACE

---

kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: MY_ADMIN
  namespace: MY_NAMESPACE
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
- apiGroups: [""]
  resources: ["pods/exec", "pods/portforward"]
  verbs: ["create"]

---

kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: MY_ADMIN
  namespace: MY_NAMESPACE
subjects:
- kind: ServiceAccount
  name: MY_ADMIN
  namespace: MY_NAMESPACE
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: MY_ADMIN
```

## Generate KUBECONFIG

After applying the manifest to your cluster, replacing `MY_NAMESPACE` and `MY_ADMIN` again with the same values:

1. Install [`mkkubeconfig`](https://github.com/JarvusInnovations/mkkubeconfig) command (if needed):

    ```bash
    sudo hab pkg install --binlink jarvus/mkkubeconfig
    ```

2. Use `mkkubeconfig` to generate and save a `KUBECONFIG` to disk:

    ```bash
    mkkubeconfig MY_NAMESPACE MY_ADMIN > ~/.kube/MY_ADMIN.yaml
    ```

## Usage

1. Share the generated KUBECONFIG via a team credential vault or other means
2. Activate the downloaded `KUBECONFIG` in your current terminal session:

    ```bash
    export KUBECONFIG=~/.kube/MY_ADMIN.yaml
    ```

3. Get the name of interesting pods, selecting by appropriate label(s), and store them in shell variables:

    ```bash
    APP_POD=$(kubectl get pod -l component=app -o jsonpath='{.items[0].metadata.name}')
    DB_POD=$(kubectl get pod -l component=database -o jsonpath='{.items[0].metadata.name}')
    ```

### Open interactive app shell

```bash
kubectl exec -it $APP_POD -- bash
```

### Open interactive database shell

```bash
kubectl exec -it $DB_POD -- psql -U admin laravel
```

### Run an artisan command

```bash
kubectl exec -it $APP_POD -- php artisan migrate
```

### Run an SQL query

```bash
echo 'SELECT * FROM users' | kubectl exec -i $DB_POD -- psql -U admin laravel
```

### Forward PostgreSQL port

```bash
kubectl port-forward pods/$DB_POD 5432:5432
```

!!! tip "Database logins"
    Default database credentials can usually be found in `helm-chart/values.yaml`
