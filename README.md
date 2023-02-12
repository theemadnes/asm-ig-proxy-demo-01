# asm-ig-proxy-demo-01
Example of using a subset of Anthos Service Mesh / Istio capabilities by doing TLS term at IG layer and proxying requests to backend service.

### do some exports

replace with your values
```
export region=us-central1
export project_id=mc-e2m-01
export cluster_name=proxy-and-app-02
export project_number=447024719410
```

### setup first GKE cluster and enable ASM MCP

> cluster creation done in console, but make sure to add context for that cluster to your kubeconfig
```
gcloud container clusters get-credentials ${cluster_name} --region ${region} --project ${project_id}
```

rename context for simplicity
```
kubectx ${cluster_name}=gke_${project_id}_${region}_${cluster_name}
```

### create backend app

```
# we're using service mesh, but not doing sidecar injection on app, as we don't need all those mesh features for our app
kubectl --context=${cluster_name} create namespace whereami
kubectl --context=${cluster_name} -n whereami apply -f app/
```

### create ingress gateway deployment

```
kubectl --context=${cluster_name} create namespace asm-ingress
kubectl --context=${cluster_name} label namespace asm-ingress istio-injection- istio.io/rev=asm-managed --overwrite
kubectl --context=${cluster_name} -n asm-ingress apply -f ingress-gateway/
```



### notes / junk

> ignore this if you're not me

get fleet membership(s)
```
gcloud --project=${project_id} container fleet memberships list 
```

clean up old fleet membership for `proxy-and-app-01`
```
gcloud --project=${project_id} container fleet memberships delete proxy-and-app-01-1 
```

also clean up other clusters
```
gcloud --project=${project_id} container fleet memberships delete gke-us-central1 
gcloud --project=${project_id} container fleet memberships delete gke-us-west2
```