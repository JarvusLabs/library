# Deploying a project

## Prerequisites

- A working `Dockerfile` at the root of the project repository

## Setup

### Publish Docker container via GitHub

Within the project's GitHub repository:

- Build a Helm 3 chart at in a `helm-chart/` directory at the root of the project
- Define a holobranch named `helm-chart` that outputs the contents of the `helm-chart/` directory
- Create a GitHub Actions workflow triggered by `v*` tags to build and publish a Docker container image to the project's GitHub Packages

### Create project deployment directory

Within the shared cluster's GitHub repository, create a top-level directory naming this project deployment:

```bash
mkdir client-project
```

### Set up release values

Inside the project deployment directory created in the cluster's repository above, create a YAML file to hold deployment-specific overrides to the Helm chart's `values.yaml`, including at least the current image+tag:

=== "client-project/release.yaml"

    ```yaml
    image: docker.pkg.github.com/jarvusinnovations/client-project/client-project:v1.2.3"
    ```

This will be where the image version gets bumped to deploy new versions to the shared cluster. You can add any other instance-specific overrides like hostnames as well.

### Populate deployment directory with project's helm-chart projection

Still within the shared cluster's GitHub repository, define a holosource and holomapping to populate the project's directory created above with content from the project's `helm-chart` projection:

=== ".holo/sources/client-project.toml"

    ```toml
    [holosource]
    url = "https://github.com/JarvusInnovations/client-project.git"
    ref = "refs/heads/deploy"
    ```

=== ".holo/branches/k8s-manifests/client-project.toml"

    ```toml
    [holomapping]
    holosource = "=>helm-chart"
    files = "**"
    ```

During the projection of the cluster's `k8s-manifests` holobranch, the `client-project` directory will be populated from the remote `helm-chart` holobranch from the `client-project` holosource. The identically-named directory within the cluster's repository is then overlaid on top via an existing holomapping to copy all the cluster repository's overrides with `after = "*"` configured.

### Test composition

To test the set up so far, project the `k8s-manifests` holobranch without lensing and list the full contents of the resulting project deployment directory:

```bash
git ls-tree -r $(git holo project --working --no-lens k8s-manifests):client-project
```

The output should include `Chart.yaml`, `values.yaml`, and `templates/` from the project repository's `helm-chart` sources, plus the `release.yaml` file from the shared cluster repository's overrides directory:

```console
info: indexing working tree (this can take a while under Docker)...
...
info: writing final output tree...
info: projection ready
100644 blob 539bef2f8faea713a589ffabf97f094314faaf44    Chart.yaml
100644 blob dbf8c7a6ffc8edd87788ef885adf68cbf556f238    release.yaml
100644 blob dfcff8182969fe2db44037ed1c3a582b362bcdcc    templates/_helpers.tpl
100644 blob e7e1a8bbbb63dc859dbf283557c413ae3e7c776f    templates/cronjob.yaml
100644 blob 8fd6234ea1b1e2b04e469ef8ea31a860ec068901    templates/deployment.yaml
100644 blob b0c972120c53b742397d9032a8e14222b2aa14ef    templates/ingress.yaml
100644 blob 0b2daeff5d1416eb981d6fa03dba5eabbebb6e73    templates/pvc.yaml
100644 blob 51ecaa3981401a8f20f56c458d92a9e46a3b3b8c    templates/service.yaml
100644 blob 9a7fb93863e295ee8f059f3607840d7bf222b98d    values.yaml
```

### Define hololens to build chart

Finally, define a lens that uses `helm template` to transform the combined Helm chart content to deploy-ready Kubernetes manifests:

=== ".holo/lenses/client-project.toml"

    ```toml
    [hololens]
    package = "holo/lens-helm3/1.2"

    [hololens.input]
    root = "client-project"
    files = "**"

    [hololens.output]
    merge = "replace"

    [hololens.helm]
    release_name = "client-project"

    namespace = "client"
    namespace_fill = true

    chart_path = "."
    value_files = [
        "values.yaml",
        "release.yaml"
    ]
    ```

### Test composition+lensing

To test the full manifest generation process, first sync your local `k8s/deploy` branch to the current remote version:

```bash
git update-ref refs/heads/k8s/deploy origin/k8s/deploy
```

Then commit the projected `k8s-manifests` holobranch **with** lensing on top to preview how it will be extended:

```bash
git holo project k8s-manifests-github --working --commit-to=k8s/deploy
```

### Add GitHub credentials to new namespace

Authentication is always required to pull Docker images from GitHub Packages, even when the project repository is public. Wherever the project's Helm chart templates reference the project's Docker container image published on GitHub, `imagePullSecrets` should be configured to use a secret named `regcred`.

The shared cluster's `.github/workflows/deploy-manifests.yml` workflow must then be edited to add a line with the new project namespace to the `REGCRED_NAMESPACES` env value. With that, the workflow will handle generating a `regcred` secret with access to GitHub in each namespace to include in every deployment.

### Authenticate access to private project sources

If the project's repository is private, an additional step must be taken to grant fetch access to the GitHub Actions workflow running under the shared cluster's repository.

Edit the shared cluster's `.github/workflows/build-manifests.yml` workflow and add an environment variable to the manifests projection step overriding the named source with an authenticated Git remote URL:

```yaml
# ...
jobs:
  build-manifests:
    # ...
    steps:
    - name: 'Update holobranch: k8s/manifests'
      uses: JarvusInnovations/hologit@actions/projector/v1
      env:
        # ...
        HOLO_SOURCE_CLIENT_PROJECT: https://jarvus-bot:${{ secrets.BOT_GITHUB_TOKEN }}@github.com/JarvusInnovations/client-project.git
```

The bot user must be granted read access to the repository.
