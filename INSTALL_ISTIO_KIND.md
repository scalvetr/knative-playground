# Install Istio KinD


Create the cluster:

```shell
cat <<EOF | kind create cluster --name knative-playground --wait 5m --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30080
    hostPort: 80
    listenAddress: "0.0.0.0" # Optional, defaults to "0.0.0.0"
    protocol: TCP
  - containerPort: 30443
    hostPort: 443
    listenAddress: "0.0.0.0" # Optional, defaults to "0.0.0.0"
    protocol: TCP
EOF
```
Verify the cluster

```shell
kind get clusters
kubectl config use-context kind-istio-playground
kubectl cluster-info
```

Create the namespaces:

```shell
kubectl create namespace istio-system
kubectl create namespace istio-ingress
kubectl label namespace istio-ingress istio-injection=enabled
```

Install the istio-base, istiod and istio-ingress packages.

```shell
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update

helm install istio-base istio/base --version 1.13.2 -n istio-system
helm install istiod istio/istiod --version 1.13.2 -n istio-system --wait
helm install istio-ingress istio/gateway --version 1.13.2 -n istio-ingress --wait --set service.type=NodePort
```

Check out the ports exposed by this service:

```shell
kubectl get services -n istio-ingress istio-ingress -o json | jq -r '.spec.ports'
```

Let’s now modify the node ports to match the ones configured earlier in the cluster extraPortMappings: 30080 and 30443.

```shell
kubectl -n istio-ingress patch svc istio-ingress --patch \
'{"spec": { "ports": [ { "port": 80, "nodePort": 30080 }, { "port": 443, "nodePort": 30443 } ] } }'
# service/istio-ingress patched
```