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

- To get the inital password use below command

```
kubectl  -n argocd get secret/argocd-initial-admin-secret -ojsonpath={'.data.password'} | base64 -d; echo
```

- In the argocd UI add the github repo `https://github.com/thirumurthis/argocd-app.git`


- Create the application in the argocd

```
cd argo_sync_wave

kubectl -n argocd app_1.yaml
```

-- The default documentation based update for the configmap

https://argo-cd.readthedocs.io/en/latest/operator-manual/health/

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  resource.customizations.health.argoproj.io_Application: |
    hs = {}
    hs.status = "Progressing"
    hs.message = ""
    if obj.status ~= nil then
      if obj.status.health ~= nil then
        hs.status = obj.status.health.status
        if obj.status.health.message ~= nil then
          hs.message = obj.status.health.message
        end
      end
    end
    return hs
```

- below code could be used for health check based on Claude 

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  # Application health check
  resource.customizations.health.argoproj.io_Application: |
    hs = {}
    hs.status = "Progressing"
    hs.message = ""
    
    if obj.status ~= nil then
      -- Check health status
      if obj.status.health ~= nil then
        hs.status = obj.status.health.status
        if obj.status.health.message ~= nil then
          hs.message = obj.status.health.message
        end
      end
      
      -- Must be synced
      if obj.status.sync ~= nil and obj.status.sync.status ~= nil then
        if obj.status.sync.status ~= "Synced" then
          hs.status = "Progressing"
          hs.message = "Sync status: " .. obj.status.sync.status
          return hs
        end
      else
        hs.status = "Progressing"
        hs.message = "Waiting for sync status"
        return hs
      end
      
      -- Check for active operations
      if obj.status.operationState ~= nil and obj.status.operationState.phase ~= nil then
        local phase = obj.status.operationState.phase
        if phase == "Running" then
          hs.status = "Progressing"
          hs.message = "Sync operation running"
          return hs
        elseif phase == "Failed" then
          hs.status = "Degraded"
          hs.message = "Sync operation failed"
          return hs
        elseif phase == "Terminating" then
          hs.status = "Progressing"
          hs.message = "Terminating"
          return hs
        end
      end
      
      -- Only healthy if synced AND healthy AND no active operations
      if hs.status == "Healthy" and 
         obj.status.sync.status == "Synced" and
         (obj.status.operationState == nil or 
          obj.status.operationState.phase == "Succeeded") then
        return hs
      end
      
      -- Check if there are resources out of sync
      if obj.status.summary ~= nil then
        local summary = obj.status.summary
        if summary.images ~= nil or summary.externalURLs ~= nil then
          -- Additional checks can be added here
        end
      end
    end
    
    hs.status = "Progressing"
    hs.message = "Application not ready"
    return hs
  
  # Timeout settings
  timeout.reconciliation: 300s
```