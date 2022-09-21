# flux-local-dev

[Flux](https://fluxcd.io) local dev environment with Docker and Kubernetes KIND.

## Who is this for?

- **Flux users** who want to test Flux configs locally, without having to push changes to a Git repository. 
  Config changes are pushed to a local registry and synced on the cluster by Flux automatically.
- **Flux contributors** who want to test their changes to Flux controllers locally,
  without having to push the container images to an external registry.
- **Flux maintainers** who want to test Flux prereleases on various Kubernetes versions and configurations.

## How does it work?

This project spins up a Docker Registry container named `kind-registry` and a Kubernetes Kind cluster
named `flux` under the same Docker network. Then it installs Flux and configures it to upgrade itself
from the latest OCI artifact published at `ghcr.io/fluxcd/flux-manifests`.

| Component                                                                                                | Role                            | Host                        |
|----------------------------------------------------------------------------------------------------------|---------------------------------|-----------------------------|
| [Kubernetes KIND](https://kind.sigs.k8s.io/)                                                             | Local cluster                   | Binds to port 80 and 443    |
| [Docker Registry](https://docs.docker.com/registry/)                                                     | Local registry                  | Binds to port 5050          |
| [Flux](https://fluxcd.io)                                                                                | Cluster reconciler              | -                           |
| [ingress-nginx](https://github.com/kubernetes/ingress-nginx)                                             | Ingress for `*.flux.local`      | -                           |
| [cert-manager](https://github.com/cert-manager/cert-manager)                                             | Self-signed ingress certs       | -                           |
| [metrics-server](https://github.com/kubernetes-sigs/metrics-server)                                      | Container resource metrics      | -                           |
| [kube-prometheus-stack](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack) | Prometheus Operator and Grafana | Binds to grafana.flux.local |
| [weave-gitops](https://github.com/weaveworks/weave-gitops)                                               | Flux UI                         | Binds to ui.flux.local      |
| [podinfo](https://github.com/stefanprodan/podinfo)                                                       | Demo app                        | Binds to podinfo.flux.local |

The Docker registry is exposed on the local machine on `localhost:5050` and inside the cluster
on `kind-registry:5000`. The registry servers two purposes:
- hosts container images e.g. `docker push localhost:5050/podinfo:test1`
- hosts OCI artifacts e.g. `flux push artifact oci://localhost:5050/podinfo-manifests:test1`

To facilitate ingress access to the Flux UI and any other
application running inside the cluster, the Kubernetes Kind container
binds to port `80` and `443` on localhost.
Ingress is handled by Kubernetes ingress-nginx and self-signed TLS certs
are provided by cert-manager.

To monitor how the deployed applications perform on the cluster,
the kube-prometheus-stack and metrics-server Helm charts are installed at
bootstrap along with the Flux Grafana dashboards.

To monitor and debug Flux using a Web UI, the Weave GitOps Helm chart is
installed at bootstrap.


## How to get started?

### Prerequisites 

Install Kubernetes kind, kubectl, flux and other CLI tools with Homebrew:

```shell
make tools
```

The complete list of tools can be found in the `Brewfile`.

### Bootstrap

Start the dev environment with:

```shell
make up
```

The `make up` command performs the following steps:
- creates the Docker registry container if it's not already running
- creates the Kubernetes Kind cluster if it's not already running
- pushes the Kubernetes manifests as OCI artifacts to the local registry
  - `locahost:5050/flux-cluster-sync` is generated from `kubernetes/clusters/local`
  - `locahost:5050/flux-infra-sync` is generated from `kubernetes/infra`
  - `locahost:5050/flux-apps-sync` is generated from `kubernetes/apps`
- installs Flux on the clusters and configures it to self upgrade from `oci://ghcr.io/fluxcd/flux-manifests`
- waits for Flux to reconcile the cluster addons from `oci://kind-registry:5000/flux-infra-sync`
- waits for Flux to reconcile the demo apps from `oci://kind-registry:5000/flux-apps-sync`

### Access Flux UI

Add the following domains to `/etc/hosts`:

```txt
127.0.0.1 podinfo.flux.local
127.0.0.1 grafana.flux.local
127.0.0.1 ui.flux.local
```

Verify that the NGINX ingress self-signed TLS works:

```shell
make check
```

Access the Flux UI and Grafana using the username `admin` and password `flux`:

- [http://ui.flux.local/applications](http://ui.flux.local/applications)
- [http://grafana.flux.local/d/flux-control-plane](http://grafana.flux.local/d/flux-control-plane/flux-control-plane?orgId=1&refresh=10s)
- [http://grafana.flux.local/d/flux-cluster](http://grafana.flux.local/d/flux-cluster/flux-cluster-stats?orgId=1&refresh=10s)

Access the demo application on [http://podinfo.flux.local](http://ui.flux.local/).

### Sync local changes

Add a label to the `apps` namespace definition:

```shell
yq eval '.metadata.labels.env="dev"' -i ./kubernetes/apps/namespace.yaml
```

Push the changes to the local registry with:

```shell
make sync
```

Verify that Flux has reconciled the namespace:

```shell
kubectl get ns apps --show-labels
```

### Teardown

Delete the registry and the Kubernetes cluster with:

```shell
make down
```

## How to test Flux controllers?

Assuming you are contributing a change to kustomize-controller,
and you want to run a series of end-to-end tests before opening a PR.
Most importantly, you want to make sure your changes don't break Flux 
capability to upgrade itself.

### Build and push the controller image

From within the kustomize-controller local clone, run `make docker-build` to build the controller image.
If your local machine is an Apple M1, set the arch to `linux/arm64` and run:

```shell
BUILD_PLATFORMS=linux/arm64 make docker-build
```

Tag and push the image to your local registry:

```shell
docker tag fluxcd/kustomize-controller:latest localhost:5050/kustomize-controller:test1
docker push localhost:5050/kustomize-controller:test1
```

### Deploy the controller

From within the flux-local-dev clone, open the `kubernetes/clusters/local/flux-system/flux-sync.yaml` file
and add an image patch:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: flux-sync
  namespace: flux-system
spec:
  images:
    - name: ghcr.io/fluxcd/kustomize-controller
      newName: localhost:5050/kustomize-controller
      newTag: test1
```

Sync the changes on the cluster with `make sync` and verify that the image is being rolled out:

```shell
kubectl -n flux-system get deploy/kustomize-controller --watch
```

Finally, verify that the upgrade was successful with:

```console
$ flux check 

✔ kustomize-controller: deployment ready
► localhost:5050/kustomize-controller:test1
```

## How to test Flux prereleases?

Assuming you are maintainer, and you want to test the Flux controller suite
before a release.

### Build and push the manifests

From within the flux2 local clone, run `make build-dev` to build the Flux CLI
binary that embeds the install manifests.

Extract the manifests to a directory with:

```shell
mkdir -p flux-vnext

./bin/flux install --components-extra=image-reflector-controller,image-automation-controller \
--export > ./flux-vnext/install.yaml
```

Push the manifests to your local registry:

```shell
./bin/flux push artifact oci://localhost:5050/flux:latest --path ./flux-vnext \
--source="$(git config --get remote.origin.url)" \
--revision="$(git rev-parse HEAD)"
```

### Deploy the controllers

From within the flux-local-dev clone, open the `kubernetes/clusters/local/flux-system/flux-source.yaml` file,
change the URL to point to your local registry and enable the insecure flag:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: OCIRepository
metadata:
  name: flux-source
  namespace: flux-system
spec:
  url: oci://kind-registry:5000/flux
  insecure: true
```

Sync the changes on the cluster with `make sync` and wait for the new version to rollout:

```shell
flux reconcile ks flux-sync --with-source
```

Finally, verify that the upgrade was successful with:

```shell
flux check 
```
