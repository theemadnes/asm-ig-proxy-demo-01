# asm-ig-proxy-demo-01
Example of using a subset of Anthos Service Mesh / Istio capabilities by doing TLS term at IG layer and proxying requests to backend service.

### do some exports

replace with your values
```
export region=us-central1
export project_id=mc-e2m-01
export cluster_name=proxy-and-app-01
export project_number=447024719410
```

### setup first GKE cluster w/ ASM MCP enabled and two node pools: default-pool and proxy-pool

```
gcloud beta container --project ${project_id} clusters create ${cluster_name} --region ${region} --no-enable-basic-auth --cluster-version "1.24.8-gke.2000" --release-channel "regular" --machine-type "e2-standard-4" --image-type "COS_CONTAINERD" --disk-type "pd-balanced" --disk-size "100" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --max-pods-per-node "110" --num-nodes "1" --logging=SYSTEM,WORKLOAD --monitoring=SYSTEM --enable-ip-alias --network "projects/${project_id}/global/networks/default" --subnetwork "projects/${project_id}/regions/${region}/subnetworks/default" --no-enable-intra-node-visibility --default-max-pods-per-node "110" --enable-autoscaling --min-nodes "0" --max-nodes "3" --location-policy "BALANCED" --enable-dataplane-v2 --no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --labels mesh_id=proj-${project_number} --workload-pool "${project_id}.svc.id.goog" && gcloud beta container --project ${project_id} node-pools create "proxy-pool" --cluster ${cluster_name} --region ${region} --machine-type "e2-standard-4" --image-type "COS_CONTAINERD" --disk-type "pd-balanced" --disk-size "100" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --num-nodes "1" --enable-autoscaling --min-nodes "0" --max-nodes "3" --location-policy "BALANCED" --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --max-pods-per-node "110"
```

rename context
```
kubectx ${cluster_name}=gke_${project_id}_${region}_${cluster_name}
```

### create backend app

```
kubectl --context=${cluster_name} create namespace whereami
kubectl --context=${cluster_name} -n whereami apply -f app/
```

### create ingress gateway deployment

```
kubectl --context=${cluster_name} create namespace asm-ingress
kubectl --context=${cluster_name} label namespace asm-ingress istio-injection- istio.io/rev=asm-managed --overwrite
kubectl --context=${cluster_name} -n asm-ingress apply -f ingress-gateway/
```