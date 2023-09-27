# mca-4.3-p2

Istio Canary Deployments with Flagger

## Generate Docker images in DockerHub [v1-4]

``` bash
mvn spring-boot:build-image -Dspring-boot.build-image.imageName=manloralm/mca-4.3-p2:v[1-4]
docker push manloralm/mca-4.3-p2:v[1-4]
```

Need to deploy first the database to allow pass the integration tests.

``` bash
docker run -d \
--name mysql \
-e MYSQL_DATABASE=test \
-e MYSQL_ROOT_PASSWORD=pass \
-p 3306:3306 \
mysql:8.0.23
```

## Setup MiniKube cluster with Istio 1.19 and Kubernetes Metrics Server

``` bash
minikube start --memory 8192 --cpus 4 \
--addons enable ingress \
--addons enable istio-provisioner \
--addons enable istio \
--addons enable metrics-server
```

## Install Flagger

[Link](https://docs.flagger.app/tutorials/istio-progressive-delivery)

## Install Istio 1.19 and metrics server

``` bash
istioctl install --set profile=default

kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.19/samples/addons/prometheus.yaml

```

## Add Flagger and Grafana to k8s cluster

``` bash
kubectl apply -f https://raw.githubusercontent.com/weaveworks/flagger/master/artifacts/flagger/crd.yaml

helm upgrade -i flagger flagger/flagger \
--namespace=istio-system \
--set crd.create=false \
--set meshProvider=istio \
--set metricsServer=http://prometheus:9090

helm upgrade -i flagger-grafana flagger/grafana \
--namespace=istio-system \
--set url=http://prometheus.istio-system:9090 \
--set user=admin \
--set password=change-me

```

## Run the port forward command and navigate to http://localhost:3000 - admin/change-me

``` bash
kubectl -n istio-system port-forward svc/flagger-grafana 3000:80
```

## Configure Flagger to use Istio

``` bash
kubectl create ns test

kubectl label namespace test istio-injection=enabled

```

## Deploy database and app

``` bash
kubectl apply -f k8s/mysql.yaml

kubectl apply -f k8s/library.yaml
```

## Deploy canary resource and gateway

``` bash
kubectl apply -f k8s/canary.yaml

kubectl apply -f k8s/gateway.yaml
```

## Deploy service to generate load

``` bash
kubectl apply -k github.com/weaveworks/flagger//kustomize/tester
```

## Apply changes to the canary

``` bash
kubectl -n test set image deployment/library library=manloralm/mca-4.3-p2:v<version>
```

Need to update all resources with the new version server port and image. Each app version has a different port.

``` bash
kubectl apply -k ./k8s
```

## Watch the canary progress

``` bash
watch -n 1 'kubectl describe canary.flagger.app library -n test | tail'
```
