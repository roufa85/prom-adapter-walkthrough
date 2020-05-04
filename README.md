# prom-adapter-walkthrough

```
kubectl apply -f sample-app.deploy.yaml
curl http://$(minikube ip):$(kubectl get service sample-app -o jsonpath='{ .spec.ports[0].nodePort }')/metrics 
```

```
kubectl apply -f 00-prom-crd/
kubectl apply -Rf 01-prom-setup/
kubectl apply -f service-monitor.yaml
```
