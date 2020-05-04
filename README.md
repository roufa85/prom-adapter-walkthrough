# prom-adapter-walkthrough

## sample-app Deployment

```
kubectl apply -f sample-app.deploy.yaml
curl http://$(minikube ip):$(kubectl get service sample-app -o jsonpath='{ .spec.ports[0].nodePort }')/metrics 
```

## Prometheus Setup

```
kubectl apply -f 00-prom-crd/
kubectl apply -Rf 01-prom-setup/
kubectl apply -f service-monitor.yaml
```

Access the Prom UI: 

```
kubectl port-forward svc/prometheus-operated 9090 9090
```

http://localhost:9090/graph?g0.range_input=1h&g0.expr=http_requests_total&g0.tab=1

## Launching the Adapter

The k8s metrics server should be enabled, for minikube enable the addon:

```
minikube addons enable metrics-server
```

Now that you've got a running copy of Prometheus that's monitoring your application, you'll need to deploy the adapter, which knows how to communicate with both Kubernetes and Prometheus, acting as a translator between the two.

The `adapter` directory contains the appropriate files for creating the Kubernetes objects to deploy the adapter.

As a prerequisite, we need to create a secret called `cm-adapter-serving-certs` with two values: `serving.crt` and `serving.key`:

```
cd $GOPATH/src/github.com/directxman12/k8s-prometheus-adapter
export PURPOSE=serving
openssl req -x509 -sha256 -new -nodes -days 365 -newkey rsa:2048 -keyout ${PURPOSE}-ca.key -out ${PURPOSE}-ca.crt -subj "/CN=ca"
echo '{"signing":{"default":{"expiry":"43800h","usages":["signing","key encipherment","'${PURPOSE}'"]}}}' > "${PURPOSE}-ca-config.json"
kubectl create secret tls cm-adapter-serving-certs --cert=PATH_TO/serving-ca.crt --key=PATH_TO/serving-ca.key
```

Apply the secret:
```
kubectl create secret tls cm-adapter-serving-certs --cert=certs/serving-ca.crt --key=certs/serving-ca.key
```

```
kubectl apply -f adapter/
```

```
kubectl get --raw '/apis/custom.metrics.k8s.io/v1beta1/' | jq .

{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "custom.metrics.k8s.io/v1beta1",
  "resources": [
    {
      "name": "namespaces/http_requests_per_second",
      "singularName": "",
      "namespaced": false,
      "kind": "MetricValueList",
      "verbs": [
        "get"
      ]
    },
    {
      "name": "pods/http_requests_per_second",
      "singularName": "",
      "namespaced": true,
      "kind": "MetricValueList",
      "verbs": [
        "get"
      ]
    }
  ]
}
```

```
kubectl get --raw '/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests_per_second' | jq .

{
  "kind": "MetricValueList",
  "apiVersion": "custom.metrics.k8s.io/v1beta1",
  "metadata": {
    "selfLink": "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/%2A/http_requests_per_second"
  },
  "items": [
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "sample-app-847c9b7cdf-lp4gm",
        "apiVersion": "/v1"
      },
      "metricName": "http_requests_per_second",
      "timestamp": "2020-05-04T13:33:14Z",
      "value": "29865m",
      "selector": null
    }
  ]
}
```
Notice that the API uses Kubernetes-style quantities to describe metric values. These quantities use SI suffixes instead of decimal points. The most common to see in the metrics API is the m suffix, which means milli-units, or 1000ths of a unit. If your metric is exactly a whole number of units on the nose, you might not see a suffix. Otherwise, you'll probably see an m suffix to represent fractions of a unit.

For example, here, 500m would be half a request per second, 10 would be 10 requests per second, and 10500m would be 10.5 requests per second.

## Experimenting

```
hey -n 10000 -q 10 -c 5 http://$(minikube i
p):$(kubectl get service sample-app -o jsonpath='{ .spec.ports[0].nodePort }')/ 
```

```
kubectl get --raw '/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests_per_second?selector=app%3Dsample-app' | jq .
```

## Horizontal-Pod AutoScaling

Now that you finally have the metrics API set up, your HorizontalPodAutoscaler should be able to fetch the appropriate metric, and make decisions based on it:

```
kubectl apply -f sample-app-hpa.yaml
```

```
kubectl describe hpa sample-app

Name:                                  sample-app
Namespace:                             default
Labels:                                <none>
Annotations:                           CreationTimestamp:  Mon, 04 May 2020 15:40:17 +0200
Reference:                             Deployment/sample-app
Metrics:                               ( current / target )
  "http_requests_per_second" on pods:  33m / 500m
Min replicas:                          1
Max replicas:                          10
Deployment pods:                       1 current / 1 desired
Conditions:
  Type            Status  Reason              Message
  ----            ------  ------              -------
  AbleToScale     True    ReadyForNewScale    recommended size matches current size
  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate a replica count from pods metric http_requests_per_second
  ScalingLimited  False   DesiredWithinRange  the desired count is within the acceptable range
Events:           <none>
```

Generate more traffic:

```
hey -n 10000 -q 10 -c 5 http://$(minikube i
p):$(kubectl get service sample-app -o jsonpath='{ .spec.ports[0].nodePort }')/ 
```
If you describe the HPA again, you should see that the last observed metric value roughly corresponds to your rate of requests, and that the HPA has recently scaled your app.
Check again the custom HPA:

```
kubectl describe hpa sample-app

Name:                                  sample-app
Namespace:                             default
Labels:                                <none>
Annotations:                           CreationTimestamp:  Mon, 04 May 2020 15:40:17 +0200
Reference:                             Deployment/sample-app
Metrics:                               ( current / target )
  "http_requests_per_second" on pods:  16366m / 500m
Min replicas:                          1
Max replicas:                          10
Deployment pods:                       1 current / 4 desired
Conditions:
  Type            Status  Reason            Message
  ----            ------  ------            -------
  AbleToScale     True    SucceededRescale  the HPA controller was able to update the target scale to 4
  ScalingActive   True    ValidMetricFound  the HPA was able to successfully calculate a replica count from pods metric http_requests_per_second
  ScalingLimited  True    ScaleUpLimit      the desired replica count is increasing faster than the maximum scale rate
Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  SuccessfulRescale  11s   horizontal-pod-autoscaler  New size: 4; reason: pods metric http_requests_per_second above target
```

With more load and traffic the HPA will scale up to 10:

```
kubectl describe hpa sample-app

Name:                                  sample-app
Namespace:                             default
Labels:                                <none>
Annotations:                           CreationTimestamp:  Mon, 04 May 2020 15:40:17 +0200
Reference:                             Deployment/sample-app
Metrics:                               ( current / target )
  "http_requests_per_second" on pods:  5032m / 500m
Min replicas:                          1
Max replicas:                          10
Deployment pods:                       10 current / 10 desired
Conditions:
  Type            Status  Reason            Message
  ----            ------  ------            -------
  AbleToScale     True    ReadyForNewScale  recommended size matches current size
  ScalingActive   True    ValidMetricFound  the HPA was able to successfully calculate a replica count from pods metric http_requests_per_second
  ScalingLimited  True    TooManyReplicas   the desired replica count is more than the maximum replica count
Events:
  Type    Reason             Age    From                       Message
  ----    ------             ----   ----                       -------
  Normal  SuccessfulRescale  2m21s  horizontal-pod-autoscaler  New size: 4; reason: pods metric http_requests_per_second above target
  Normal  SuccessfulRescale  2m6s   horizontal-pod-autoscaler  New size: 8; reason: pods metric http_requests_per_second above target
  Normal  SuccessfulRescale  111s   horizontal-pod-autoscaler  New size: 10; reason: pods metric http_requests_per_second above target
```

Now that you've got your app autoscaling on the HTTP requests, you're all ready to launch! If you leave the app alone for a while, the HPA should scale it back down, so you can save precious budget for the launch party.

## Tear-Down

```
kubectl delete -Rf . --ignore-not-found=true
kubectl delete secret cm-adapter-serving-certs
```