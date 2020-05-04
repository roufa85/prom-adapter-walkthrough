# prom-adapter-walkthrough


kubectl create service nodeport sample-app
curl http://$(minikube ip):$(kubectl get service sample-app -o jsonpath='{ .spec.ports[0].nodePort }')/metrics 

wget https://raw.githubusercontent.com/coreos/prometheus-operator/master/bundle.yaml
kubectl apply -f service-monitor.yaml

