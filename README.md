# Kafka Deployment

Helm umbrella chart that deploys a Kafka cluster through the Strimzi operator, together with the Confluent
Schema Registry and the AKHQ web UI.

## Overview

The chart `steadops-kafka` packages the [AKHQ](https://akhq.io/) Helm chart as a dependency and adds its own
templates for:

- a Strimzi `Kafka` resource named `kafka` in KRaft mode with node pools enabled, with an internal plain listener
  on port 9092 and an external NodePort listener on port 9094
- two Strimzi `KafkaNodePool` resources, one for the brokers and one for the KRaft controllers
- the Confluent Schema Registry as `Deployment` and `Service`, plus the `KafkaTopic` it stores its schemas in
- Istio `VirtualService` and Forecastle `ForecastleApp` resources for AKHQ and the Schema Registry
- the `Namespace` of the release

The Strimzi, Istio, and Forecastle resources are only rendered when the cluster serves the matching API
(`kafka.strimzi.io/v1`, `networking.istio.io/v1`, `forecastle.stakater.com/v1alpha1`). The Strimzi operator
itself is not part of this chart and must already run in the cluster.

The service names in the values (for example `kafka-kafka-bootstrap.kafka.svc.cluster.local` and `kafka-akhq`)
assume the release name `kafka` in the namespace `kafka`.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) for the containerized commands, or the
  `SteadOps-Steadies-K8s-Workplace` workbench, which provides `helm`, `yq`, `kubectl`, `hetzner-k3s`, and `act`.
- All commands run from the repository root.

## Repository Layout

| Path                              | Purpose                                                              |
| --------------------------------- | -------------------------------------------------------------------- |
| `Chart.yaml`                      | Chart metadata and the `akhq` dependency                             |
| `Chart.lock`                      | Pinned dependency version, committed                                 |
| `charts/`                         | Committed `akhq` chart archive                                       |
| `templates/`                      | Kafka, node pool, Schema Registry, AKHQ routing, and namespace       |
| `values.yaml`                     | Defaults for `global`, `entityOperator`, `kafka`, `schemaRegistry`   |
| `values-subchart-overrides.yaml`  | Values for the `akhq` subchart                                       |
| `values-local.yaml`               | Local cluster: single replicas, reduced resources                    |
| `values-development.yaml`         | Development cluster domain                                           |
| `values-production.yaml`          | Production cluster domain                                            |
| `tests/`                          | helm-unittest suites; snapshots in `tests/__snapshot__/` are ignored |
| `.github/workflows/`              | Helm unittest and Trufflehog CI workflows                            |
| `renovate.json`                   | Renovate configuration                                               |

`values.yaml` sets the top-level keys `global` (ACME domain, Forecastle group, Istio ingress gateway namespace),
`entityOperator` (topic and user operator resources), `kafka` (cluster and controller replicas, resources, storage,
and Kafka version), `nodeAffinityLabel`, and `schemaRegistry` (image, bootstrap servers, topic, resources).

### Values for Subcharts

`values-subchart-overrides.yaml` holds the values for the subchart(s) used by this chart. They are kept apart
from the values for the main chart to be able to unit test for incompatible changes in the values of the
subcharts. This is necessary because Helm does not allow switching off the usage of `values.yaml`. Now it is
possible to test, for example, whether we use the same registry and repository for images as the subcharts do.

## Setup

`charts/` contains the committed `akhq` archive. After cloning, and after any pull that changes `Chart.lock`,
install the dependency pinned by `Chart.lock`.

Containerized flow (`alpine/helm` ships `yq`):

```sh
 docker run \
   -e HOME=/tmp \
   --entrypoint sh \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm -c '
     yq "explode(.) | .dependencies[] | select(.repository == \"http*\") | .name + \" \" + .repository" Chart.yaml |
       while read -r name repo; do helm repo add --force-update "$name" "$repo"; done &&
     helm dependency build .
   '
```

Workbench flow:

```sh
 yq 'explode(.) | .dependencies[] | select(.repository == "http*") | .name + " " + .repository' Chart.yaml |
   while read -r name repo; do helm repo add --force-update "$name" "$repo"; done
 helm dependency build .
```

## Rendering

Render the manifests for an environment into `_local/<environment>`, which is gitignored. Set `env` to `local`,
`development`, or `production`. The `-a` flags announce the APIs that gate the Strimzi, Istio, and Forecastle
templates.

```sh
 env=local
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm template \
   -a forecastle.stakater.com/v1alpha1/ForecastleApp \
   -a kafka.strimzi.io/v1/Kafka \
   -a kafka.strimzi.io/v1/KafkaNodePool \
   -a kafka.strimzi.io/v1/KafkaTopic \
   -a networking.istio.io/v1/VirtualService \
   -f values-subchart-overrides.yaml \
   -f "values-$env.yaml" \
   --include-crds \
   -n kafka \
   --output-dir "_local/$env" \
   --release-name \
   kafka \
   .
```

In the workbench, run the same `helm template` command without the `docker run` wrapper.

### Value Files Used by Argo CD

Argo CD lists all value files it may use in the `PARAMETERS` tab of the `kafka` application. Argo CD ignores
missing value files, so drop the ones that do not exist before passing them to `helm template`. To print the list
from a running cluster in the workbench:

```sh
 kubectl -n argocd get applications.argoproj.io kafka -o yaml |
   yq '.spec.source.helm.valueFiles[]'
```

Containerized flow:

```sh
 docker run \
   -e KUBECONFIG=/tmp/kubeconfig \
   --entrypoint sh \
   --rm \
   -u $(id -u) \
   -v "$KUBECONFIG:/tmp/kubeconfig:ro" \
   ghcr.io/steadforce/steadops/workbenches/k8s:main -c '
     kubectl -n argocd get applications.argoproj.io kafka -o yaml |
       yq ".spec.source.helm.valueFiles[]"
   '
```

## Testing

Run the helm-unittest suites in `tests/` after [Setup](#setup). The suites for the `akhq` subchart need the
archive in `charts/`.

```sh
 docker run \
   -e HELM_CACHE_HOME=/tmp/helm/.config \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   helmunittest/helm-unittest \
   .
```

To write a JUnit report like the pipeline does, add `-t JUnit -o test-output.xml`. Without `-t`, helm-unittest
writes XUnit. `test-output.xml` is gitignored.

The pipeline also lints the chart:

```sh
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm lint \
   .
```

> [!NOTE]
> The workbench image does not ship the helm-unittest plugin, so use the containerized command there as well.

## CI/CD

| Workflow                   | Trigger                                          | Reusable workflow                    |
| -------------------------- | ------------------------------------------------ | ------------------------------------ |
| `helm-unittest.yaml`       | every push                                       | `helm-unittest.yaml@v4.2.0`          |
| `trufflehog.yaml`          | push and pull request to `main`, manual dispatch | `trufflehog-oss.yaml@v3.0.0`         |

Both reusable workflows live in
[`steadforce/steadops-workflows`](https://github.com/steadforce/steadops-workflows).

The Helm unittest workflow:

1. installs Helm and the helm-unittest plugin
2. registers the HTTP(S) dependency repositories from `Chart.yaml` and runs `helm dependency build`, which fails
   when `Chart.lock` and `Chart.yaml` disagree
3. runs `helm unittest --with-subchart=true -t JUnit -o test-output.xml` and publishes the results as a check run
4. runs `helm lint`
5. on branches starting with `renovate/`, posts the result to Microsoft Teams

The Teams notification uses two optional secrets:

| Workflow secret                                   | Filled from repository secret                     |
| ------------------------------------------------- | ------------------------------------------------- |
| `steadops-helm-renovation-ms-teams-webhook`       | `STEADOPS_HELM_RENOVATION_MS_TEAMS_WEBHOOK`       |
| `steadops-helm-renovation-ms-teams-error-webhook` | `STEADOPS_HELM_RENOVATION_ERROR_MS_TEAMS_WEBHOOK` |

Successes go to the regular webhook. Renovate update failures go to the error webhook, which points to a separate
Teams channel. Failures fall back to the regular webhook when the error secret is not set. Without
either secret, no notification is sent. Both must be Microsoft Teams Workflows webhook URLs; legacy Office 365
connector URLs no longer work.

The Trufflehog workflow scans the commit range of the push or pull request for leaked secrets and fails when it
finds one.

### Run the Pipeline Locally

To run the pipeline locally, start the workbench and run `act` from the repository root. On the first run, `act`
asks which flavour of its image to use; the default `medium` is a good starting point. `act` skips publishing the
test results and the Teams notification.

```sh
 act
```

## Dependency Updates

[Renovate](https://docs.renovatebot.com/) uses `config:base` with the dependency dashboard, so it opens pull
requests for updates without automerging them, for example for the `akhq` chart, the Schema Registry image, and the
pinned `steadops-workflows` versions. `helmUpdateSubChartArchives` makes Renovate also update the archive in `charts/`.

To change the `akhq` version by hand, edit `Chart.yaml` and regenerate `Chart.lock` and the archive in `charts/`:

```sh
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm dependency update \
   .
```

Commit the new `Chart.lock` and the new archive in `charts/` together with `Chart.yaml`. See the
[Helm docs](https://helm.sh/docs/topics/charts/#chart-dependencies) for details.
