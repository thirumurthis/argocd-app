```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.1/standard-install.yaml
```

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.19.2/cert-manager.yaml
```

```
kubectl create ns argocd
```

```
kubectl apply -n argocd  -k argocd_install/kustomize/
```

```
helm repo add apisix https://charts.apiseven.com
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

```
kubectl create ns apisix
```

```
helm upgrade -i apisix apisix/apisix --namespace apisix \
--set service.type=NodePort \
--set service.http.enabled=true \
--set service.http.servicePort=80 \
--set service.http.containerPort=9080 \
--set service.http.nodePort=30080 \
--set service.tls.servicePort=443 \
--set service.tls.nodePort=30443 \
--set dashboard.enabled=true \
--set ingress-controller.enabled=true \
--set ingress-controller.config.apisix.serviceNamespace=apisix \
--set ingress-controller.config.kubernetes.enableGatewayAPI=true \
--set ingress-controller.gatewayProxy.createDefault=true
```

cd argocd_install
```
kubectl -n argocd apply -f cert_issuer.yaml
kubectl -n argocd apply -f cert_certificate.yaml
kubectl -n argocd apply -f ingress_argocd.yaml
```