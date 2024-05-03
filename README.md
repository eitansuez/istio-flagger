2023.02.13

# Streamlining canary deployments with [Flagger](https://flagger.app/) and Istio

## Introduction

Flagger is an open-source tool for progressive delivery.

Read about Flagger at https://docs.flagger.app/.

This document helps you understand and learn Flagger by excercising it using Istio's [`bookinfo`](https://istio.io/latest/docs/examples/bookinfo/) sample application.

What follows is the general recipe.

## Prerequisites

- A running Kubernetes cluster, with the ability to configure ingress

## Install Istio

```shell
istioctl install
```

Note that the above will use the "default" profile.

## Install Istio observability addons

The addons are located in the `samples/addons` folder of the Istio distribution, and include:

- Prometheus
- Grafana
- Jaeger
- Kiali

Feel free to deploy only Prometheus and Grafana, or the entire set.

Assuming your [Istio distribution](https://istio.io/latest/docs/setup/getting-started/#download) is located in the path denoted by the variable `$ISTIO_DISTRIBUTION`:

```shell
kubectl apply $ISTIO_DISTRIBUTION/samples/addons/prometheus.yaml
kubectl apply $ISTIO_DISTRIBUTION/samples/addons/grafana.yaml
```

etc..

> ### Question
>
> The Flagger documentation mentions their own Grafana dashboards to expose RED metrics.
> As far as I can tell this is not necessary.  Perhaps their instructions predate the introduction of Istio's observability dashboard samples.


## Install Flagger

See https://docs.flagger.app/install/flagger-install-on-kubernetes

```shell
helm repo add flagger https://flagger.app
helm repo update
```

Apply the CRD:

```shell
kubectl apply -f https://raw.githubusercontent.com/fluxcd/flagger/main/artifacts/flagger/crd.yaml
```

Install Flagger:

```shell
helm upgrade -i flagger flagger/flagger \
  --namespace=istio-system \
  --set crd.create=false \
  --set meshProvider=istio \
  --set metricsServer=http://prometheus:9090
```

## Install the Flagger load tester

It's necessary to drive traffic to the application in order to generate the metrics necessary for Flagger to determine the state of the application during a progressive rollout.

See https://docs.flagger.app/usage/webhooks

```shell
kubectl create ns test
kubectl label ns test istio-injection=enabled
helm upgrade -i flagger-loadtester flagger/loadtester \
  --namespace=test \
  --set cmd.timeout=1h \
  --set cmd.namespaceRegexp=''
```

## The scenario

The `bookinfo` demo application comes with three versions of the `reviews` service.
The idea is to deploy `bookinfo` with initial version 1, and to use Flagger to canary-upgrade `reviews` to subsequent versions.

## How does it work?  A general outline

- You define a "canary" Custom Resource (CR)
- Flagger creates two clusterIP services: `reviews-canary` and `reviews-primary`, backed by two deployments: `reviews` and `reviews-primary`.
- When the application is not being upgraded, the "primary" version receives all of the traffic.  The other workload is scaled to zero replicas.
- During progressive delivery, we progressively promote the new version (`reviews`) to be `primary`.
- Flagger watches for changes made to the `reviews` deployment.  When a change is made, it triggers a promotion.

## Deploy `bookinfo`

The manifests for the initial deployment of the `bookinfo` application are in [`bookinfo.yaml`](bookinfo.yaml).

This manifest differs from the original version published on istio.io as follows:

- The `reviews` Deployment is named `reviews`, not `reviews-v1` or `reviews-v2` etc..
- The `version` label has been removed.
- We do not initially deploy versions 2 and 3.  We will let Flagger do that.  The manifests for those deployments have been separated out to [`reviews-v2.yaml`](reviews-v2.yaml) and [`reviews-v3.yaml`](reviews-v3.yaml).  The only difference between these deployments and the original is the container image name they reference.


1. Create the `bookinfo` namespace

    ```shell
    kubectl create ns bookinfo
    ```

1. Label it for sidecar injection:
 
    ```shell
    kubectl label ns bookinfo istio-injection=enabled
    ```

1. Deploy `bookinfo`:

    ```shell
    kubectl apply -f bookinfo.yaml -n bookinfo
    ```

1. Configure ingress for the `productpage` service:

    ```shell
    kubectl apply -f bookinfo-gateway.yaml -n bookinfo
    ```

## Draft a Canary custom resource

See [`reviews-canary.yaml`](reviews-canary.yaml).

This file configures the parameters that become a part of the canary deployment algorithm that Flagger performs.

## Apply the Canary resource:

```shell
kubectl apply -f reviews-canary.yaml -n bookinfo
```

The status of the resource will initially be "Initializing".  When flagger has finished initializing, the status will change to "Initialized."

Describe the reviews canary:

```shell
kubectl -n bookinfo describe canary/reviews
```

- Note the events section at the end of the output.  After the canary status switches from "Initializing" to "Initialized", you should see an event stating "Initialization done! reviews.bookinfo".

Two new services `reviews-canary` and `reviews-primary` will have been created by Flagger:

```shell
kubectl get svc -n bookinfo
```

> ### Question
> 
> What does this message mean?:
> 
> "reviews-primary.bookinfo not ready: waiting for rollout to finish: observed deployment generation less than desired generation"

Also note that the reviews deployment is set to 0 replicas and replaced with "reviews-primary".
That's because that deployment will become the canary version when changes are applied.
So it should not be receiving any traffic during stable (non-promotion) periods.

At this point, test that the `productpage` comes up and that it shows served reviews: "reviews served by reviews-primary-...".

## Turn on Envoy sidecar console logging

Before we proceed, we need to ensure that we can see or detect when requests are made against the canary workload.

We deployed Istio with the default profile, which does not turn on console logging for Envoy sidecars (the demo profile does).

This can be done using with the [Telemetry](https://istio.io/latest/docs/tasks/observability/logs/access-log/#enable-envoys-access-logging) resource.

1. Apply the [`telemetry.yaml`](telemetry.yaml) Telemetry resource:

    ```shell
    kubectl apply -f telemetry.yaml
    ```

1. Tail the sidecar logs:

    ```shell
    kubectl logs --follow -n bookinfo -l app=reviews -c istio-proxy
    ```

The canary is configured to load-test the new version of the app using the Flagger loadtester.  You can see the requests come through by tailing those logs.

## Trigger a canary upgrade

Deploy an update to the `reviews` deployment:

```shell
kubectl apply -f reviews-v2.yaml -n bookinfo
```

## Observe the progress

Watch the events section of the canary resource:

```shell
watch "kubectl describe canary -n bookinfo reviews | tail -20"
```

You should see the progression go from 20% of traffic siphoned to the canary, to 40%, then 60%, and ultimately the promotion of the canary to "primary".

The last events should state "Routing all traffic to primary" followed by "Promotion completed".

The `reviews-primary` pod id will change after promotion, since it represents a newer version.

The product page should now show ratings using black colored stars.

## Repeat for `reviews-v3`

```shell
kubectl apply -f reviews-v3.yaml -n bookinfo
```

You may see a warning-type event that states:

> _Halt advancement no values found for istio metric request-success-rate probably reviews.bookinfo is not receiving traffic: running query failed: no values found_

I suspect what happens here is that there's a delay in getting the metrics needed to ascertain whether to proceed.  So the algorithm waits.  Once Flagger knows the new version of the app meets the specified metrics thresholds, we proceed (or rollback).

Ultimately, the promotion succeeds.

End-to-end the process (as configured) takes 8-9 minutes.

## Thoughts/Questions

- Why do I need to remove the `version` label on the deployment?
- The contract is the deployment, which feels odd.
- The documentation doesn't clearly explain what the user's contract actually is.
  I'd like to see an explanation of how Flagger expects the app to be initially deployed before control over the application is transferred over to Flagger.
  
