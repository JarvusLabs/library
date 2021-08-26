# Deploy Pull Requests

Enable a project GitHub repository to automatically deploy each Pull Request into a dedicated namespace hosted on an external shared cluster

## Prerequisites

- An operational [shared cluster](./create-cluster.md)
- A working `Dockerfile` at the root of the project repository

## Setup

### Add a service account

The GitHub Actions Workflows driving PR deployments will need a service account with read/write access to the dedicated namespace that all PR environments will be deployed to. Add this manifest to the cluster's GitHub repository under e.g. `deployers/myproject.yaml` where it will become part of the automated deployment:

=== "deployers/myproject.yaml"

    ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: myproject

    ---

    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: deployer
      namespace: myproject

    ---

    kind: Role
    apiVersion: rbac.authorization.k8s.io/v1beta1
    metadata:
      name: deployer
      namespace: myproject
    rules:
    - apiGroups: ["*"]
      resources: ["*"]
      verbs: ["*"]

    ---

    kind: RoleBinding
    apiVersion: rbac.authorization.k8s.io/v1beta1
    metadata:
      name: deployer
      namespace: myproject
    subjects:
    - kind: ServiceAccount
      name: deployer
      namespace: myproject
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: deployer
    ```

### Generate KUBECONFIG

After the service account manifest has been merged and deployed, you can generate a KUBECONFIG that client code can use to connect to the cluster under it:

1. Install [`mkkubeconfig`](https://github.com/JarvusInnovations/mkkubeconfig) command (if needed):

    ```bash
    sudo hab pkg install --binlink jarvus/mkkubeconfig
    ```

2. Use `mkkubeconfig` to generate a base64-encoded KUBECONFIG for GitHub:

    ```bash
    mkkubeconfig myproject deployer | base64
    ```

3. Copy the base64-encoded blob following the status log lines and paste it into a secret in the project GitHub repository named `KUBECONFIG_BASE64`
4. Install `k8s-deploy.yml` and `k8s-destroy.yml` into `.github/workflows/` within the project GitHub repository and tailor the environment variables at the top
