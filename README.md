# Kafka

Kafka deployment using the Strimzi operator.

## Dependencies

This chart pulls in istio `akhq` as a dependency. The version
used is specified in `Chart.yaml` in the `dependencies` section.
If you change the version in there, you need to then run

    $ helm dependency update

in order to have the chart downloaded to the `charts` directory
and then also commit that new version alongside with the altered
`Chart.yaml` file.

See the [Helm docs](https://helm.sh/docs/topics/charts/#chart-dependencies)
for details.

# Testing

## values-subchart-overrides.yaml

The `values-subchart-overrides.yaml` file is used to override values in the subchart(s) used by this chart.
We have to separate the values for the subcharts from the values for the main chart, to be able to
unit test for incompatible changes in values of the subcharts. This is necessary because helm does not allow
switching off the usage of values.yaml. Now it's possible to test if we use the same registry and repository
for images as the subcharts are using.

## run helm unittests

```shell
 docker run --pull=always -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest .
```

Or with output in JUnit format:

```shell
 docker run --pull=always -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest -o test-output.xml .
```

## Rendering locally

To get the param-files which argocd uses take a look into argo-cd `applications-root` application details.
In the tab `PARAMETERS` all potentially used value files for argo-cd templating are listed
there. You only have to remove the blanks after `,` and the non-existing value files, since we have set
the argo-cd parameter ignore-missing-value-files.

Get argo-cd's used list of value files from a running cluster:

```shell
 VALUE_FILES_CSV=$(kubectl -n argocd get applications.argoproj.io kafka -o yaml | \
   yq '.spec.source.helm.valueFiles | @csv'); \
   for i in ${VALUE_FILES_CSV//,/ }
   do
    # call your procedure/other scripts here below
      echo "      -f $i \\"
   done
```

This prints all configured value files as needed for helm templating.
Since argo-cd ignores non-existent value files, they have to be removed when used in the statements below.

## Render resources local

```bash
 helm template \
  --output-dir _local/local \
  --release-name kafka \
  -a forecastle.stakater.com/v1alpha1/ForecastleApp \
  -a kafka.strimzi.io/v1beta2/Kafka \
  -a kafka.strimzi.io/v1beta2/KafkaNodePool \
  -a kafka.strimzi.io/v1beta2/KafkaTopic \
  -a networking.istio.io/v1/VirtualService \
  -f values-subchart-overrides.yaml \
  -f values-local.yaml \
  -n kafka \
  .
```

## Run act pipeline local

To run the pipeline in local environment, startup the workbench, cd into the folder containing this
`README.md` and execute the following command:

```shell
  act
```

On first execution you're asked which flavour of the act image should be used. Using the default `medium`
is a good starting point.