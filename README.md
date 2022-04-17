# knative-playgroud

## Prerequisites
* [Install Istio](https://github.com/scalvetr/istio-playground/blob/main/README.md#istio-playground)

# Install Knative

More info [here](https://knative.dev/docs/install/operator/knative-with-operators/)

```shell
kubectl apply -f https://github.com/knative/operator/releases/download/knative-v1.3.1/operator.yaml

# Verify your Knative Operator installation
kubectl config set-context --current --namespace=default
# Check the Operator deployment status by running the command
kubectl get deployment knative-operator
```
Create the Knative Serving custom resource

```shell
export ISTIO_SYSTEM_NAMESPACE="istio-system";
export ISTIO_INGRESS_NAMESPACE="istio-ingress";
export KNATIVE_SERVING_NAMESPACE="knative-serving";

kubectl get svc -n ${ISTIO_INGRESS_NAMESPACE} istio-ingress -ojsonpath='{.metadata.labels}'


cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: ${KNATIVE_SERVING_NAMESPACE}
---
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: ${KNATIVE_SERVING_NAMESPACE}
spec:
  ingress:
    istio:
      enabled: true
      knative-ingress-gateway:
        selector:
          istio: ingress
  config:
    istio:
      gateway.knative-serving.knative-ingress-gateway: "istio-ingress.${ISTIO_INGRESS_NAMESPACE}.svc.cluster.local"
EOF
```
Configure the magic DNS
```shell
#kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.3.0/serving-default-domain.yaml

#kubectl patch configmap/config-domain \
#  --namespace knative-serving \
#  --type merge \
#  --patch '{"data":{"127.0.0.1.sslip.io":""}}'
  
kubectl patch configmap/config-domain \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"127.0.0.1.nip.io":""}}'
  
kubectl get configmap/config-domain \
  --namespace knative-serving
```

Test
```shell
kubectl label namespace default istio-injection=enabled

cat <<EOF | kubectl apply -f -
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: helloworld
  namespace: default
spec:
  template:
    spec:
      containers:
        - image: mendhak/http-https-echo
          ports:
            - containerPort: 80
          env:
              - name: HTTP_PORT
                value: "80"
EOF

kubectl get pods
kubectl get ksvc
kubectl describe revision helloworld-00001
kubectl get deployments helloworld-00001-deployment -o yaml

curl -vv http://helloworld.default.127.0.0.1.nip.io
# curl -k -v https://helloworld.default.127.0.0.1.sslip.io
```

## Build & Run
```shell
# skaffold dev --trigger notify
```