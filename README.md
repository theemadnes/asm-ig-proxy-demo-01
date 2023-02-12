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

### set up DNS
> my DNS environment isn't working so Miguel has helpfully created a DNS record and cert in his own environment to get this working; for now, ignore this section
create public zone
```
gcloud dns --project=mc-e2m-01 managed-zones create alexmattson-demo --description="" --dns-name="alexmattson.demo.altostrat.com." --visibility="public" --dnssec-state="off"
```

registrar setup
```
ns-cloud-e1.googledomains.com.
ns-cloud-e2.googledomains.com.
ns-cloud-e3.googledomains.com.
ns-cloud-e4.googledomains.com.
```

### add TLS cert to `asm-ingress` namespace
> of course, you need to have created a TLS cert and private key; replace with your own values
```
kubectl --context=${cluster_name} create -n asm-ingress secret tls whereami-credential \
  --key=tls-cert/private-key.pem \
  --cert=tls-cert/certificate.pem
```

### create gateway resource
```
kubectl --context=${cluster_name} -n asm-ingress apply -f gateway/
```

### create virtualservice
```
kubectl --context=${cluster_name} -n whereami apply -f virtualservice/
```

### call the service via NLB + proxies
```
curl https://whereami.miguelmendoza.demo.altostrat.com
```

### notes / junk

> ignore this if you're not me

get control plane status for current kubecontext
```
kubectl -n istio-system get controlplanerevision
```

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

get certbot installed
```
brew install certbot
```

sample create cluster
```
gcloud beta container --project "${project_id}" clusters create "${cluster_name}" --region "${region}" --no-enable-basic-auth --cluster-version "1.25.5-gke.2000" --release-channel "regular" --machine-type "e2-standard-4" --image-type "COS_CONTAINERD" --disk-type "pd-balanced" --disk-size "100" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --max-pods-per-node "32" --num-nodes "1" --logging=SYSTEM,WORKLOAD --monitoring=SYSTEM --enable-ip-alias --network "projects/${project_id}/global/networks/default" --subnetwork "projects/${project_id}/regions/${region}/subnetworks/default" --no-enable-intra-node-visibility --default-max-pods-per-node "32" --enable-autoscaling --min-nodes "0" --max-nodes "3" --location-policy "BALANCED" --enable-dataplane-v2 --no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --labels mesh_id=proj-${project_number} --enable-managed-prometheus --workload-pool "${project_id}.svc.id.goog" --enable-shielded-nodes #--node-locations "${region}-a","${region}-b","${region}-f"
```