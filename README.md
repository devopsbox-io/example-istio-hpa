# Prerequisites
- minikube (tested with v1.10.1, kubernetes v1.17.12)
- kvm (required by minikube)
- kubectl (tested with v1.17.12)
- helm (tested with v3.2.1)
- siege (tested with 4.0.4)
- istioctl (tested with 1.7.2)

You can use any other kubernetes distribution or load testing tool which supports HTTP/1.1 (e.g. hey). You can also probably use different versions and different virtualization solution.

# Steps
## Start minikube
```
minikube start --driver=kvm2 --kubernetes-version v1.17.12 --memory=8192 --cpus=4 && minikube tunnel
```

It will ask for your sudo password!

## Install Istio

In a separate terminal window:
```
istioctl install -y
```

## Create namespaces
```
kubectl create namespace monitoring
kubectl create namespace dev
```

## Enable Istio automatic sidecar injection
```
kubectl label namespace dev istio-injection=enabled
```

## Deploy the sample application
```
kubectl -n dev apply -f sample-app-with-istio.yaml
```

Wait for your application deployment (probably few minutes):
```
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
until curl -f http://$INGRESS_HOST; do echo "Waiting for the application to start..."; sleep 1; done
```

## Install prometheus
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo update

helm -n monitoring install prometheus prometheus-community/prometheus
```

## Install prometheus adapter
```
helm -n monitoring install prometheus-adapter prometheus-community/prometheus-adapter -f prometheus-adapter-values.yaml
```

## Deploy horizontal Pod autoscaling configuration
```
kubectl -n dev apply -f hpa.yaml
```

Wait for the metric availability:
```
until kubectl -n dev describe hpa | grep "\"istio_requests_per_second\" on pods:" | grep -v "<unknown> / 10"; do echo "Waiting for the metric availability..."; sleep 1; done
```

## Check how many Pods are running
```
watch kubectl -n dev get pod
```

## Run the load test

In a separate terminal window run (adjust number of concurrent users and test time to your environment):
```
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
siege -c 2 -t 5m http://$INGRESS_HOST
```

## Check

After few minutes you should see more pods in window with ```watch kubectl -n dev get pod```

You can also describe your horizontal pod autoscaler and see what is happening:
```
kubectl -n dev describe hpa httpbin
```

# Cleanup
```
minikube delete
```
