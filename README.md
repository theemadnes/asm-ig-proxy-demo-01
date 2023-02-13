# asm-ig-proxy-demo-01
Example of using a subset of Anthos Service Mesh / Istio capabilities by doing TLS term at IG layer and proxying requests to backend service.

### do some exports

replace with your values
```
export region=us-central1
export project_id=mc-e2m-01
export cluster_name=proxy-and-app-01
export cluster_name_2=proxy-and-app-02 # just for testing 
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

follow [these instructions](https://cloud.google.com/service-mesh/docs/managed/provision-managed-anthos-service-mesh#enable_the_fleet_feature) to get ASM w/ Managed Control Plane enabled

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
gcloud dns --project=${project_id} managed-zones create alexmattson-demo --description="" --dns-name="alexmattson.demo.altostrat.com." --visibility="public" --dnssec-state="off"
```

(internal) verify DNS delegation
```
dig alexmattson.demo.altostrat.com NS +short
```

get IP from ingress gateway LB service
```
# note that you'll need to wait a few minutes after applying the ingress gateway service for the LB IP to be available
INGRESS_GATEWAY_SVC_IP=$(kubectl --context=${cluster_name} get svc --namespace asm-ingress istio-ingressgateway --output jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

create A record for `whereami` (using super low TTL for now)
```
# create one for the main name and another for the individual cluster
gcloud dns --project=${project_id} record-sets create whereami.alexmattson.demo.altostrat.com. --zone="alexmattson-demo" --type="A" --ttl="1" --rrdatas=${INGRESS_GATEWAY_SVC_IP}
gcloud dns --project=${project_id} record-sets create whereami-1.alexmattson.demo.altostrat.com. --zone="alexmattson-demo" --type="A" --ttl="1" --rrdatas=${INGRESS_GATEWAY_SVC_IP}
```

### use cert-manager to create TLS cert to be used by ingress gateway
install cert-manager using https://cert-manager.io/docs/installation/kubectl/
```
kubectl --context=${cluster_name} apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.yaml
```

create issuer
```
kubectl --context=${cluster_name} -n asm-ingress apply -f cert-manager/issuer-prod.yaml
```

create certificate
```
kubectl --context=${cluster_name} -n asm-ingress apply -f cert-manager/certificate-prod.yaml
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
curl https://whereami.miguelmendoza.demo.altostrat.com # for miguel's demo environment
curl https://whereami.alexmattson.demo.altostrat.com # for mine
```

### notes / junk

> ignore this if you're not me; they're just debugging notes that you don't need

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
brew install certbot # seems broken on Linux due to augeas issues
```

sample create cluster
```
gcloud beta container --project "${project_id}" clusters create "${cluster_name}" --region "${region}" --no-enable-basic-auth --cluster-version "1.25.5-gke.2000" --release-channel "regular" --machine-type "e2-standard-4" --image-type "COS_CONTAINERD" --disk-type "pd-balanced" --disk-size "100" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --max-pods-per-node "32" --num-nodes "1" --logging=SYSTEM,WORKLOAD --monitoring=SYSTEM --enable-ip-alias --network "projects/${project_id}/global/networks/default" --subnetwork "projects/${project_id}/regions/${region}/subnetworks/default" --no-enable-intra-node-visibility --default-max-pods-per-node "32" --enable-autoscaling --min-nodes "0" --max-nodes "3" --location-policy "BALANCED" --enable-dataplane-v2 --no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --labels mesh_id=proj-${project_number} --enable-managed-prometheus --workload-pool "${project_id}.svc.id.goog" --enable-shielded-nodes #--node-locations "${region}-a","${region}-b","${region}-f"
```

sample create cluster 2
```
gcloud beta container --project "${project_id}" clusters create "${cluster_name_2}" --region "${region}" --no-enable-basic-auth --cluster-version "1.25.5-gke.2000" --release-channel "regular" --machine-type "e2-standard-4" --image-type "COS_CONTAINERD" --disk-type "pd-balanced" --disk-size "100" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --max-pods-per-node "32" --num-nodes "1" --logging=SYSTEM,WORKLOAD --monitoring=SYSTEM --enable-ip-alias --network "projects/${project_id}/global/networks/default" --subnetwork "projects/${project_id}/regions/${region}/subnetworks/default" --no-enable-intra-node-visibility --default-max-pods-per-node "32" --enable-autoscaling --min-nodes "0" --max-nodes "3" --location-policy "BALANCED" --enable-dataplane-v2 --no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --labels mesh_id=proj-${project_number} --enable-managed-prometheus --workload-pool "${project_id}.svc.id.goog" --enable-shielded-nodes #--node-locations "${region}-a","${region}-b","${region}-f"
```

> after creating cluster 2, make sure to enable ASM via Fleet API

verify DNS delegation
```
dig alexmattson.demo.altostrat.com NS +short
```

### cluster 2 setup

```
kubectx ${cluster_name_2}=gke_${project_id}_${region}_${cluster_name_2}
```

```
# we're using service mesh, but not doing sidecar injection on app, as we don't need all those mesh features for our app
kubectl --context=${cluster_name_2} create namespace whereami
kubectl --context=${cluster_name_2} -n whereami apply -f app/
```

```
kubectl --context=${cluster_name_2} create namespace asm-ingress
kubectl --context=${cluster_name_2} label namespace asm-ingress istio-injection- istio.io/rev=asm-managed --overwrite
kubectl --context=${cluster_name_2} -n asm-ingress apply -f ingress-gateway/
```

```
INGRESS_GATEWAY_SVC_IP_2=$(kubectl --context=${cluster_name_2} get svc --namespace asm-ingress istio-ingressgateway --output jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

```
kubectl --context=${cluster_name_2} apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.yaml
```

create issuer
```
kubectl --context=${cluster_name_2} -n asm-ingress apply -f cert-manager/issuer-prod.yaml
```

create certificate
```
kubectl --context=${cluster_name_2} -n asm-ingress apply -f cert-manager/certificate-prod-2.yaml
```

create sample TLS certs from miguel's environment # nevermind, not needed for cluster 2
```
#kubectl --context=${cluster_name_2} create -n asm-ingress secret tls whereami-credential   --key=tls-cert/private-key.pem   --cert=tls-cert/certificate.pem
```

```
kubectl --context=${cluster_name_2} -n asm-ingress apply -f gateway/
```

```
kubectl --context=${cluster_name_2} -n whereami apply -f virtualservice/
```

update DNS A record to include cluster2 svc ip
```
gcloud dns --project=mc-e2m-01 record-sets update whereami.alexmattson.demo.altostrat.com. --type="A" --zone="alexmattson-demo" --rrdatas="$INGRESS_GATEWAY_SVC_IP,$INGRESS_GATEWAY_SVC_IP_2" --ttl="1"
gcloud dns --project=${project_id} record-sets create whereami-2.alexmattson.demo.altostrat.com. --zone="alexmattson-demo" --type="A" --ttl="1" --rrdatas=${INGRESS_GATEWAY_SVC_IP_2}
```