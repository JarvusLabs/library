# Create a cluster

*Coming soon*

For now, see:

- [cluster-template](https://github.com/JarvusInnovations/cluster-template): Jarvus' public template for lightweight, self-sufficient, multi-project Kubernetes clusters.
- [jarvus-sandbox-cluster](https://github.com/JarvusInnovations/jarvus-sandbox-cluster): If you have access, this repository demonstrates the setup documented in this section.

## Set up cluster repository

1. Create a new repo (e.g. `cluster-live`) and configure it with hologit, for example:
```
mkdir -p cluster-live/.holo && cd cluster-live && git init
touch README.md && git add . && git commit -m "wip: initial commit"
git holo init && git add . && git commit -m "feat: add .holo cfg"
```

2. Add `cluster-template` as holosource in `.holo/sources`

`.holo/sources/cluster-template.toml`
```
[holosource]
url = "https://github.com/JarvusInnovations/cluster-template"
ref = "refs/tags/v0.4.0"

```
3. Add holobranch for `cluster-template` using the `k8s-blueprint` holomapping in `.holo/branches`

`.holo/branches/k8s-manifests/_cluster-template.toml`
```
[holomapping]
holosource = "=>k8s-blueprint"
files = "**"
before = "*"
```

4. Add your own k8s manifests/settings to the repo e.g. Cluster Issuers for **cert-manager** certificates

`cert-manager/cert-manager.issuers.yaml`

```
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: email@example.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx
---

apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: email@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx

```

5. Project manifests into `k8s/manifests` branch
```
git holo project --working k8s-manifests --commit-to=k8s/manifests
```

6. Checkout the branch, and diff/apply the changes in the repo to the cluster.
```
git checkout k8s/manifests

kubectl diff -Rf ./

kubectl apply -Rf ./

```

## Create GitHub token for deploy workflows

The GitHub Actions deployment workflow requires a GitHub bot user to write to the cluster repository with. This is necessary so that a branch pushed by the GitHub Actions workflow can trigger subsequent workflows. The authentication tokens that are automatically provided to all GitHub Actions workflow runs are able to push to the host repository, but their pushes are prevented from triggering any workflows.

1. Create or reuse GitHub bot user with write access to the cluster repository
2. Grant the GitHub bot user the write access to the cluster repository, and ensure that its invitation is accepted
3. [Create a Personal Access Token](https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/creating-a-personal-access-token) under the GitHub bot user with `repo` scope
4. Save the generated Personal Access Token to a secret called `BOT_GITHUB_TOKEN` under the cluster repository

## Create service account

The GitHub Actions Workflows driving deployments will need a service account with read/write access to all namespaces. Add this manifest to the cluster's GitHub repository under e.g. `deployers/cluster.yaml` where it will become part of the automated deployment, but you will need to apply it to the cluster manually ahead of the first automated deployment:

=== "deployers/cluster.yaml"

    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
    name: cluster-deployer
    namespace: kube-system

    ---

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
    name: cluster-deployer
    rules:
    - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]

    ---

    kind: RoleBinding
    apiVersion: rbac.authorization.k8s.io/v1beta1
    metadata:
    name: cluster-deployer
    namespace: kube-system
    roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: cluster-deployer
    subjects:
    - kind: ServiceAccount
    name: cluster-deployer
    namespace: kube-system
    ```

=== "mkkubeconfig"

    ```bash
    #!/bin/bash
    # adapted from: https://gist.github.com/ericchiang/d2a838ddad3f44436ae001a342e1001e

    TEMPDIR=$( mktemp -d )

    trap "{ rm -rf $TEMPDIR ; exit 255; }" EXIT

    SA_SECRET=$( kubectl get sa -n $1 $2 -o jsonpath='{.secrets[0].name}' )

    # Pull the bearer token and cluster CA from the service account secret.
    BEARER_TOKEN=$( kubectl get secrets -n $1 $SA_SECRET -o jsonpath='{.data.token}' | base64 -d )
    kubectl get secrets -n $1 $SA_SECRET -o jsonpath='{.data.ca\.crt}' | base64 -d > $TEMPDIR/ca.crt

    CLUSTER_URL=$( kubectl config view -o jsonpath='{.clusters[0].cluster.server}' )

    KUBECONFIG=$TEMPDIR/kubeconfig.yaml

    1>&2 kubectl config --kubeconfig=$KUBECONFIG \
        set-cluster \
        $CLUSTER_URL \
        --server=$CLUSTER_URL \
        --certificate-authority=$TEMPDIR/ca.crt \
        --embed-certs=true

    1>&2 kubectl config --kubeconfig=$KUBECONFIG \
        set-credentials $2 --token=$BEARER_TOKEN

    1>&2 kubectl config --kubeconfig=$KUBECONFIG \
        set-context registry \
        --cluster=$CLUSTER_URL \
        --user=$2

    1>&2 kubectl config --kubeconfig=$KUBECONFIG \
        use-context registry

    cat $KUBECONFIG
    ```

1. Apply manifests to create service account:

    ```bash
    kubectl apply -f deployers/cluster.yaml
    ```

2. Use provided script to generate a base64-encoded KUBECONFIG for GitHub:

    ```bash
    ./mkkubeconfig kube-system cluster-deployer | base64
    ```

3. Save the base64 blob output above to a secret called `KUBECONFIG_BASE64` under the cluster repository

## Create GitHub token for cluster image pulls

If deployments will utilize Docker container images uploaded to GitHub's Packages registry, the deployment workflow can handle injecting these into any desired namespace.

1. Create or reuse GitHub bot user with write access to the cluster repository
2. Grant the GitHub bot user write access to the cluster repository, and ensure that its invitation is accepted
3. [Create a Personal Access Token](https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/creating-a-personal-access-token) under the GitHub bot user with the `read:packages` scope
4. Generate a base64-encoded `.docker/config.json` file:

    ```bash
    echo -n 'Username: ' && read github_username
    echo -n 'Password: ' && read -s github_token
    echo && echo "{ \"auths\": { \"docker.pkg.github.com\": { \"auth\": \"$(echo -n "${github_username}:${github_token}" | base64)\" } } }" | base64
    ```

5. Save the generated base64 blob to a secret called `DOCKER_CONFIG_BASE64` under the cluster repository
