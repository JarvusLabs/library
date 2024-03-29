# Create a cluster

## Examples

- [cluster-template](https://github.com/JarvusInnovations/cluster-template): Jarvus' public template for lightweight, self-sufficient, multi-project Kubernetes clusters.
- [jarvus-sandbox-cluster](https://github.com/JarvusInnovations/jarvus-sandbox-cluster): If you have access, this repository demonstrates the setup documented in this section.

## Set up cluster repository

1. Create a new repo (e.g. `cluster-live`) and configure it with hologit, for example:

    ```bash
    git init cluster-live && cd cluster-live
    touch README.md && git add . && git commit -m "wip: initial commit"
    git holo init && git commit -m "feat: configure holo workspace"
    ```

2. Add `cluster-template` as holosource in `.holo/sources`

    === ".holo/sources/cluster-template.toml"

        ```toml
        [holosource]
        url = "https://github.com/JarvusInnovations/cluster-template"
        ref = "refs/tags/v0.4.0"
        ```

3. Add holobranch for `cluster-template` using the `k8s-blueprint` holomapping in `.holo/branches`

    === ".holo/branches/k8s-manifests/_cluster-template.toml"

        ```toml
        [holomapping]
        holosource = "=>k8s-blueprint"
        files = "**"
        before = "*"
        ```

4. Add your own k8s manifests/settings to the repo e.g. Cluster Issuers for **cert-manager** certificates

    === "cert-manager.issuers.yaml"

        ```yaml
        apiVersion: cert-manager.io/v1
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

        apiVersion: cert-manager.io/v1
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

    ```bash
    git holo project --working k8s-manifests --commit-to=k8s/manifests
    ```

6. Checkout the branch, and diff/apply the changes in the repo to the cluster.

    ```bash
    git checkout k8s/manifests

    kubectl diff -Rf ./

    kubectl apply -Rf ./
    ```

## Create GitHub token for deploy workflows

The GitHub Actions deployment workflow requires a GitHub bot user to write to the cluster repository with. This is necessary so that a branch pushed by the GitHub Actions workflow can trigger subsequent workflows. The authentication tokens that are automatically provided to all GitHub Actions workflow runs are able to push to the host repository, but their pushes are prevented from triggering any workflows.

1. Create or reuse GitHub bot user with write access to the cluster repository
2. Grant the GitHub bot user the write access to the cluster repository, and ensure that its invitation is accepted
3. [Create a Personal Access Token](https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/creating-a-personal-access-token) under the GitHub bot user with the `repo` and `workflow` scopes
4. Save the generated Personal Access Token to a secret called `BOT_GITHUB_TOKEN` under the cluster repository

## Create service account

The GitHub Actions Workflows driving deployments will need a service account with read/write access to all namespaces. Add this manifest to the cluster's GitHub repository under e.g. `deployers/cluster.yaml` where it will become part of the automated deployment, but you will need to apply it to the cluster manually ahead of the first automated deployment:

=== "github-actions.serviceaccount.yaml"

    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: github-actions
      namespace: kube-system

    ---

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: github-actions
    rules:
    - apiGroups: ["*"]
      resources: ["*"]
      verbs: ["*"]
    - apiGroups: ["apiextensions.k8s.io"]
      resources: ["customresourcedefinitions"]
      verbs: ["*"]
    - nonResourceURLs: ["*"]
      verbs: ["*"]

    ---

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: github-actions-cluster-admin-binding
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: github-actions
    subjects:
    - kind: ServiceAccount
      name: github-actions
      namespace: kube-system
    ```

1. Install [`mkkubeconfig`](https://github.com/JarvusInnovations/mkkubeconfig) command (if needed):

    ```bash
    sudo hab pkg install --binlink jarvus/mkkubeconfig
    ```

2. Apply manifests to create service account:

    ```bash
    kubectl apply -f github-actions.serviceaccount.yaml
    ```

3. Use `mkkubeconfig` to generate a base64-encoded KUBECONFIG for GitHub:

    ```bash
    mkkubeconfig kube-system github-actions | base64
    ```

4. Save the base64 blob output above to a secret called `KUBECONFIG_BASE64` under the cluster repository

## Create GitHub token for cluster image pulls

If deployments will utilize Docker container images uploaded to GitHub's Packages registry, the deployment workflow can handle injecting these into any desired namespace.

1. Create or reuse GitHub bot user with write access to the cluster repository
2. Grant the GitHub bot user write access to the cluster repository, and ensure that its invitation is accepted
3. [Create a Personal Access Token](https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/creating-a-personal-access-token) under the GitHub bot user with the `read:packages` scope
4. Generate a base64-encoded `.docker/config.json` file:

    ```bash
    echo -n 'Username: ' && read github_username
    echo -n 'Password: ' && read -s github_token
    echo && echo "{ \"auths\": { \"ghcr.io\": { \"auth\": \"$(echo -n "${github_username}:${github_token}" | base64)\" }, \"ghcr.io\": { \"auth\": \"$(echo -n "${github_username}:${github_token}" | base64)\" } } }" | base64
    ```

5. Save the generated base64 blob to a secret called `DOCKER_CONFIG_BASE64` under the cluster repository

If no private image pulls will be needed, create `DOCKER_CONFIG_BASE64` with the value `e30K` (an empty JSON document)
