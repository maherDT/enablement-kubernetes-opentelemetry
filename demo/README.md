# OpenTelemetry to Bindplane demo

This walkthrough follows telemetry from the Astronomy Shop demo application,
through its OpenTelemetry Collector, to the Bindplane gateway running in the
cluster.

## Data flow

```text
Astronomy Shop services
        |
        | OTLP
        v
astronomy-shop-otelcol
        |
        | OTLP/HTTP (4319)
        v
bindplane-gateway-agent.bindplane-agent.svc.cluster.local
        |
        | Bindplane-managed pipeline
        v
Configured destination
```

The in-cluster gateway is the receiving hop. Its receiver and downstream
exporter pipeline must also be configured in Bindplane; the Kubernetes
manifest alone does not specify the final destination.

## 1. Configure the Bindplane gateway

The gateway resources are defined in [`bindplane-agent.yaml`](./bindplane-agent.yaml):

- The `bindplane-agent` namespace and service account are created.
- The `bindplane-gateway-agent` Deployment runs two gateway replicas, with an
  autoscaler configured for 2–10 replicas.
- The `bindplane-gateway-agent` Service exposes OTLP/gRPC on port `4320` and
  OTLP/HTTP on port `4319`. It also exposes the other ports declared in the
  manifest.
- The agents connect to the Bindplane management plane over OpAMP and receive
  their collector configuration there.

Before applying the manifest, configure the gateway in Bindplane with an
OTLP/HTTP receiver listening on port `4319` and a pipeline that exports to the
desired destination.

The demo manifest references the gateway's OpAMP secret from a Kubernetes
Secret; no token is stored in this folder. Set `OPAMP_SECRET_KEY` in your shell
to the value provided by your Bindplane administrator, then create the Secret
before applying the gateway manifest:

```sh
set -o pipefail
kubectl create namespace bindplane-agent --dry-run=client -o yaml | kubectl apply -f -
read -r -s -p "Bindplane OpAMP secret: " OPAMP_SECRET_KEY
printf '\n'
: "${OPAMP_SECRET_KEY:?No secret entered}"
printf '%s' "${OPAMP_SECRET_KEY}" | kubectl create secret generic bindplane-opamp \
  --namespace bindplane-agent \
  --from-file=OPAMP_SECRET_KEY=/dev/stdin \
  --dry-run=client -o yaml | kubectl apply -f -
unset OPAMP_SECRET_KEY
```

From the repository root, apply the gateway and wait for it to be ready:

```sh
kubectl apply -f demo/bindplane-agent.yaml
kubectl rollout status deployment/bindplane-gateway-agent -n bindplane-agent
kubectl get pods,svc -n bindplane-agent
```

Check that the gateway Service has endpoints:

```sh
kubectl get endpoints bindplane-gateway-agent -n bindplane-agent
```

## 2. Route the demo Collector to the gateway

The extracted Helm values overlay is in
[`otel-collector-values.yaml`](./otel-collector-values.yaml). It contains the
demo's `opentelemetry-collector` configuration:

- The `otlphttp/bindplane` exporter targets
  `http://bindplane-gateway-agent.bindplane-agent.svc.cluster.local:4319`.
  This is the gateway Service's cluster-local DNS name and OTLP/HTTP port.
- The traces, metrics, collector-health metrics, and logs pipelines export
  through `otlphttp/bindplane`.
- The Collector enriches telemetry with Kubernetes metadata and resource
  attributes before exporting it. The `transform/dynatrace-k8s` processor name
  refers to metadata mapping; it is not the exporter or the destination.

The exporter replaces the previous `otlphttp/dynatrace` exporter. Check that
the gateway and its OTLP/HTTP receiver are available before upgrading the
Astronomy Shop release.

## 3. Apply the demo Collector configuration

Run these commands from the repository root. Set `NAME` to the namespace or
service-namespace label you want the demo telemetry to use; the substitution fills
the `NAME_TO_REPLACE` placeholders in the values file.

```sh
export NAME=astronomy-shop
sed "s,NAME_TO_REPLACE,${NAME},g" \
  demo/otel-collector-values.yaml \
  > /tmp/astronomy-shop-collector-values.yaml

helm upgrade --install astronomy-shop open-telemetry/opentelemetry-demo \
  --version 0.31.0 \
  --namespace astronomy-shop \
  --create-namespace \
  --reuse-values \
  --values /tmp/astronomy-shop-collector-values.yaml
```

The `open-telemetry` Helm repository must already be configured. If it is not:

```sh
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
```

## 4. Verify the route

Confirm both workloads are ready and that the gateway Service has endpoints:

```sh
kubectl get pods -n astronomy-shop
kubectl get pods,svc,endpoints -n bindplane-agent
```

Inspect the Astronomy Shop Collector logs for exporter connection or retry
errors:

```sh
kubectl logs deployment/astronomy-shop-otelcol \
  -n astronomy-shop --since=10m
```

Then confirm in Bindplane that the gateway is connected, its assigned
configuration is healthy, and telemetry is reaching the configured destination.
Successful delivery to the Kubernetes Service only verifies the first hop; it
does not by itself verify the downstream Bindplane pipeline.
