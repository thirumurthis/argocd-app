```sh
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.1/standard-install.yaml
```

```sh
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.19.2/cert-manager.yaml
```

```sh
kubectl create ns argocd
```

```sh
kubectl apply -n argocd  -k argocd_install/kustomize/
```

```sh
helm repo add apisix https://charts.apiseven.com
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

```sh
kubectl create ns apisix
```

```sh
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

```sh
kubectl -n argocd apply -f cert_issuer.yaml
kubectl -n argocd apply -f cert_certificate.yaml
kubectl -n argocd apply -f ingress_argocd.yaml
```

- To get the inital password use below command

```sh
kubectl  -n argocd get secret/argocd-initial-admin-secret -ojsonpath={'.data.password'} | base64 -d; echo
```

- In the argocd UI add the github repo `https://github.com/thirumurthis/argocd-app.git`


- Create the application in the argocd

```sh
cd argo_sync_wave

kubectl -n argocd app_1.yaml
```

-- The default documentation based update for the configmap

https://argo-cd.readthedocs.io/en/latest/operator-manual/health/

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

---
- The copilot suggested child app of apps

```yaml
resource.customizations.health.argoproj.io_Application: |
    hs = {}
    hs.status = "Progressing"
    hs.message = "Waiting for child application to be ready"

    if obj.status == nil then
      return hs
    end

    -- If sync is still running
    if obj.status.operationState ~= nil then
      if obj.status.operationState.phase == "Running" then
        hs.status = "Progressing"
        hs.message = "Child application sync in progress"
        return hs
      end

      if obj.status.operationState.phase == "Failed" then
        hs.status = "Degraded"
        hs.message = "Child application sync failed"
        return hs
      end
    end

    -- Check sync status
    if obj.status.sync ~= nil then
      if obj.status.sync.status ~= "Synced" then
        hs.status = "Progressing"
        hs.message = "Child application not synced"
        return hs
      end
    end

    -- Check health status
    if obj.status.health ~= nil then
      if obj.status.health.status == "Healthy" then
        hs.status = "Healthy"
        hs.message = "Child application synced and healthy"
        return hs
      end

      if obj.status.health.status == "Degraded" then
        hs.status = "Degraded"
        hs.message = obj.status.health.message or "Child application degraded"
        return hs
      end
    end
    return hs
```